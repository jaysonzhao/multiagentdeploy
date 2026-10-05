# Multi-agent OpenShift deployment kit

Reusable deployment guidance and reference artifacts for running the
[Multi-Agent Loan Origination](https://github.com/rh-ai-quickstart/multi-agent-loan-origination)
demo on OpenShift with an existing OpenShift AI MaaS model and MLflow tracking.
The upstream project's Apache 2.0 license is included in [LICENSE](LICENSE).

## Contents

- [OpenShift MaaS deployment skill](.agents/skills/openshift-maas-demo-deployment/SKILL.md)
  (also mirrored for Claude Code under `.claude/skills/`).
- [`deploy/helm/mortgage-ai/`](deploy/helm/mortgage-ai/): application chart,
  including a non-secret `values.cluster-v4vsc.yaml` example for gpt-oss-20b
  MaaS and OpenShift AI MLflow.
- [`deploy/openshift/architecture-guide/`](deploy/openshift/architecture-guide/):
  responsive architecture and persona prompt page, with a Kustomize deployment
  for a separate Route in the application project.

The cluster overlay is an example for one environment; inspect the current
MaaS endpoint, model identifier, MLflow service and TLS setup before reusing it.
No API key or local database, object-store, or Keycloak password belongs in
this repository. Keep private values in an ignored `values.local.yaml` and put
the MaaS key in a Secret scoped to the application project.

## Deploy

Follow the [chart instructions](deploy/helm/mortgage-ai/README.md) to create the
project-scoped MaaS Secret and deploy the app. To publish the guide into that
same project:

```bash
oc apply -k deploy/openshift/architecture-guide \
  -n multi-agent-loan-origination
oc get route loan-architecture-guide -n multi-agent-loan-origination
```

The guide has its own Deployment, Service, ConfigMap and HTTPS Route; it does
not modify the Helm release or OpenShift AI configuration.
