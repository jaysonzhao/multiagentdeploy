---
name: openshift-maas-demo-deployment
description: Deploy and document the multi-agent loan origination demo on OpenShift with an existing OpenShift AI MaaS model and MLflow tracing. Use when adapting its Helm chart, creating a project-scoped MaaS Secret, deploying the architecture and persona guide, or validating the live demo.
---

# OpenShift MaaS Demo Deployment

Use this workflow for the `rh-ai-quickstart/multi-agent-loan-origination`
application and similar project-scoped demo deployments. The reference chart
and guide are in this repository under `deploy/`.

## Guardrails

- Keep all application resources in the explicitly requested application
  project. Do not modify resources in other user projects.
- Treat OpenShift AI as a managed dependency. Discover the already-deployed
  MaaS endpoint, model ID, MLflow service and CA read-only. Do not patch a
  `DataScienceCluster`, gateway, model server, operator, or shared OpenShift AI
  setting to make the app work.
- Keep the MaaS API key and database, MinIO, SQLAdmin and Keycloak passwords
  outside Git, Helm values, ConfigMaps, terminal output and screenshots. Use
  project-scoped Secrets and reference them by name and key. Never paste a key
  into `--set`, a manifest, a skill, or a page.
- Do not uninstall or replace an existing Helm release to add the guide. The
  guide has its own Deployment, Service, ConfigMap and Route.
- The included cluster overlay is environment-specific. Re-check every
  endpoint, model identifier, namespace, CA and route before reusing it.

## Workflow

### 1. Inspect the app and target

Read repository instructions and inspect the chart and current worktree first.
Keep unrelated local edits intact. Confirm the selected cluster and identity
with `oc whoami` and `oc whoami --show-server`, then inspect the requested
project, existing Helm release, routes and workload state. Do not switch to a
different cluster based on an old kubeconfig context.

For an existing OpenShift AI installation, verify the MaaS model and MLflow
service from live state or the user's supplied endpoint details. Use the
OpenAI-compatible inference URL and exact model ID exposed by MaaS; this demo's
example is `publishers/llm/models/gpt-oss-20b`. The MLflow URI varies by cluster;
the example overlay uses the internal service with TLS and the service CA.

### 2. Create app-scoped credentials

Create or reuse the application project only when the user requests it. Ask the
user to create a MaaS key if none is available; never mint or retrieve a key by
changing OpenShift AI settings. Store the supplied key directly in the app
project without putting it in Helm values or Git:

```bash
read -r -s MAAS_API_KEY
printf '%s' "$MAAS_API_KEY" | oc create secret generic maas-credentials \
  -n "$PROJECT" --from-file=api-key=/dev/stdin
unset MAAS_API_KEY
```

Use the chart's `llm.existingSecret` reference. Supply app database, MinIO,
SQLAdmin and Keycloak passwords through a mode-`0600` ignored
`values.local.yaml` or an approved secret manager. Do not stage that file.

### 3. Deploy the application

Review Helm values and rendered resources before changing an existing release.
For a new release, use the cluster overlay and private local values file:

```bash
helm lint deploy/helm/mortgage-ai \
  -f deploy/helm/mortgage-ai/values.cluster-v4vsc.yaml
helm template mortgage-ai deploy/helm/mortgage-ai \
  -n "$PROJECT" \
  -f deploy/helm/mortgage-ai/values.cluster-v4vsc.yaml \
  -f deploy/helm/mortgage-ai/values.local.yaml >/dev/null
helm upgrade --install mortgage-ai deploy/helm/mortgage-ai \
  -n "$PROJECT" \
  -f deploy/helm/mortgage-ai/values.cluster-v4vsc.yaml \
  -f deploy/helm/mortgage-ai/values.local.yaml \
  --wait --timeout 15m
```

The chart references the MaaS key from an existing Secret. Its MLflow option
mounts the OpenShift service CA and uses a RoleBinding in the app namespace to
the platform-provided MLflow integration ClusterRole. This grants access to
the app service account without changing OpenShift AI configuration. Keep TLS
verification enabled when the CA is available.

The seed Job runs after install only. Do not restore a post-upgrade seed hook or
force reseeding; upgrades must not reset demo records.

If the application is already deployed and the request is only to add the
guide, skip Helm and use the separate Kustomize deployment below.

### 4. Deploy the architecture and prompt guide

The page summarizes the browser/UI/API flow, LangGraph's five role-scoped
agents, Keycloak, MaaS, MLflow, MCP risk tools, PostgreSQL/pgvector, MinIO, and
the RBAC, HMDA, PII and audit boundaries. It includes every example question
from `docs/personas.md`, with one-question and per-persona copy controls.

Keep the questions grounded in the source docs. When updating them, recount all
five personas and verify each source prompt remains copyable. Update the demo
link and source links when the target routes or upstream docs change.

Deploy only to the app project:

```bash
oc apply -k deploy/openshift/architecture-guide -n "$PROJECT"
oc rollout status deployment/loan-architecture-guide -n "$PROJECT" \
  --timeout=180s
oc get route loan-architecture-guide -n "$PROJECT"
```

The Kustomize generator versions the ConfigMap by content, which rolls the
static server when the page changes. Remove an obsolete generated ConfigMap
only after confirming the Deployment references the new one.

### 5. Verify live behavior

- Confirm the app and guide Deployments are ready and their Routes are admitted.
- Load the app Route and exercise a representative public question against the
  configured MaaS model. Do not expose API keys in command arguments or logs.
- Confirm a corresponding trace appears in the configured MLflow experiment.
- Load the guide Route; check the architecture labels, five persona sections,
  every source question, individual copy controls and five copy-all controls.
- Inspect pod logs with care and redact credentials, tokens, borrower PII and
  document contents before sharing evidence.
- Report which checks were live, which were static, and any remaining
  environment-specific limitation.

## Store changes in Git

Before staging, inspect `git status` and the complete diff. Exclude local values,
tokens, Secrets, generated credentials and unrelated work. Run `git diff
--check`, `helm lint` for chart changes, and `oc kustomize` for guide changes.
Push only to the repository and branch the user selected; never force-push.
