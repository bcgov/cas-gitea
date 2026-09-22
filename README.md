# cas-gitea

Helm chart for running Gitea on OpenShift with the `cas-postgres-cluster`
dependency in the `c53ff1-dev` namespace.

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
