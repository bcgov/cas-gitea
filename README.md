# cas-gitea

Helm chart for running Gitea on OpenShift with the `cas-postgres-cluster`
dependency in the `c53ff1-dev` namespace.

### Admin user

The chart reads admin credentials from the Openshift Secret named by
`admin.secretName`; it does not create that Secret. Create it in the release
namespace before deploying using:

```bash
oc create secret generic gitea-admin \
	--namespace <namespace> \
	--from-literal=username=<admin-username> \
	--from-literal=password='<admin-password>' \
	--from-literal=email=<admin-email>
```

The init container creates the admin user on first deployment. 

### GitHub mirror backup

The chart can register public GitHub repositories as Gitea pull mirrors, so
Gitea stays as an incrementally-updated backup image source. Enable it and
list the repositories to mirror:

```yaml
mirror:
  enabled: true
  defaultInterval: 24h
  repos:
    - name: my-repo
      cloneAddr: https://github.com/my-org/my-repo.git
```

On `helm install`/`helm upgrade`, a post-upgrade Job calls the Gitea API to
create the `Values.organization` organization if it is missing, then creates
each repo as a pull mirror owned by that organization if it does not already
exist; both calls are idempotent and safe to rerun. Gitea's built-in
`cron.update_mirrors` task (enabled by default) then pulls each mirror on its
own schedule according to `mirror.defaultInterval` (`24h` here), so once a
mirror is created no additional script is needed for the daily sync.

## Deploy

Install the chart and its PostgreSQL dependency:

```bash
oc project c53ff1-dev
helm dependency update .
helm upgrade --install gitea . --namespace c53ff1-dev
```

The PostgreSQL operator creates the `gitea` database and user. The generated
`pguser` Secret is consumed by the Gitea deployment; no separate PostgreSQL
installation is required.

Check the deployment:

```bash
oc get pods,pvc,svc,route
oc rollout status deployment/gitea-cas-gitea
```

The chart currently exposes HTTP through an OpenShift Route.


## TODO

### Document how to utilize Repo
In the event that Github is unavailable and there is a need to utilize the repo 
stored on Gitea to accept code pushes, the steps to get there should be documented.

Currently, the `cas-registration` repo created here is a `mirror`. Gitea has built-
in features to automatically and incrementally keep a mirror repo up-to-date. But
a mirror repo is read-only. So in the event of a need to be able to push code to
the Gitea-hosted repo, we would fork a regular repo off the mirror repo.

We should still define a strategy for how we would then bring the
Github repo back up-to-date after the outage, but there shouldn't
be any risk of data-loss from scheduled tasks, so there is low risk

### User Management
Each team member needs to register as a Gitea user before they can be added to the
the preconfigured `developers` team which grants push code to a Gitea repo. Gitea 
supports OAuth integrations, so we could use Keycloak as a sign-in option. While 
Github sign-in is also an option, it seems unideal as the whole point of this exercise 
is to NOT be reliant on Github.

We could also add configurations that create user accounts on deploy.

### CI/CD
Gitea supports many Github actions. But an easy way to run them in Openshift is
only included with the enterprise edition of Gitea (ARC - actions runner controller).
Without that, Openshift's strict Security Context Constraints block the docker-in-docker
approach that the community edition relies on.

We could configure the runners to run on the Gitea pod, but that could cause resource
issues.

There are other 3rd party packages that could also work (dind, kaniko), but more research is needed.
