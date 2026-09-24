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
      owner: my-org
      cloneAddr: https://github.com/my-org/my-repo.git
```

On `helm install`/`helm upgrade`, a post-upgrade Job calls the Gitea API to
create each repo as a pull mirror if it does not already exist; the call is
idempotent and safe to rerun. Gitea's built-in `cron.update_mirrors` task
(enabled by default) then pulls each mirror on its own schedule according to
`mirror.defaultInterval` (`24h` here), so once a mirror is created no
additional script is needed for the daily sync.

## Deploy

Install the chart and its PostgreSQL dependency:

```bash
oc project c53ff1-dev
helm dependency update .
helm upgrade --install gitea . \
	--namespace c53ff1-dev \
	--set route.host=<gitea-route-host>
```

The PostgreSQL operator creates the `gitea` database and user. The generated
`pguser` Secret is consumed by the Gitea deployment; no separate PostgreSQL
installation is required.

Check the deployment:

```bash
oc get pods,pvc,svc,route
oc rollout status deployment/gitea-cas-gitea
```

The chart currently exposes HTTP through an OpenShift Route and exposes SSH
on Service port `2222`. SSH ingress requires a cluster-specific external
TCP mapping; HTTPS clone and push should be used until that mapping exists.
