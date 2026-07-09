# MaaS on xKS — Implementation Summary

## Status

E2E validated on AKS. Full user journey tested: API key creation, authenticated inference through MaaS gateway, RHCL auth enforcement (401 on unauthenticated requests).

## PRs

### 1. models-as-a-service [#1015](https://github.com/opendatahub-io/models-as-a-service/pull/1015)

**xKS kustomize overlay for MaaS controller deployment**

- Deploys MaaS controller on vanilla Kubernetes (AKS, EKS, GKE)
- Patches out OpenShift-specific annotations (`service.beta.openshift.io/serving-cert-secret-name`, `inject-cabundle`)
- Adds deployment volume mount for webhook TLS cert (cert created by the chart, not the overlay)
- Skips OpenShift-specific resources: `networking/maas` (gateway), `observability` (PodMonitor/Telemetry), `monitoring`

Files:
```
deployment/overlays/xks/kustomization.yaml   (1 file, patches only)
```

### 2. opendatahub-operator [#3670](https://github.com/opendatahub-io/opendatahub-operator/pull/3670)

**Go code to select xKS overlay when `ODH_PLATFORM_TYPE=XKS`**

3-line change in `modelsasservice_support.go`:
```go
if cluster.GetClusterInfo().Type == cluster.ClusterTypeKubernetes {
    kPath = filepath.Join(mi.Path, mi.ContextDir, "overlays", "xks")
}
```

### 3. odh-gitops [#109](https://github.com/opendatahub-io/odh-gitops/pull/109)

**RHCL dependency chart + MaaS chart integration**

| Change | File(s) |
|--------|---------|
| RHCL (Kuadrant) as dependency chart | `charts/dependencies/rhcl-operator/` (CRDs, operators, RBAC, ServiceMonitors) |
| Kuadrant RBAC fix for `monitoring.coreos.com` | `clusterrole-kuadrant-operator.yaml` |
| PostgreSQL as dependency chart | `charts/dependencies/postgresql/` |
| `components.maas.enabled` values flag | `values.yaml` |
| Values-driven `RHAI_DISABLE_MODELSASSERVICE_COMPONENT` | `deployment-rhods-operator.yaml` |
| ModelsAsService CRD | `templates/crds/customresourcedefinition-modelsasservices...yaml` |
| ModelsAsService CR in post-install hook | `post-install-crs-job.yaml` |
| MaaS gateway post-install hook | `files/create-maas-gateway.sh` + `post-install-maas-gateway-job.yaml` |
| Hook RBAC for modelsasservices + authorino | `post-install-crs-rbac.yaml` |
| Pull secret propagation to kuadrant namespaces | `values.yaml` (`dependencyNamespaces`) |
| Snapshot tests | `azure-with-maas`, `azure-with-maas-and-pull-secret` |

## Deployment

```bash
# Step 1: Dependencies (one-time)
helm install cert-manager-operator charts/dependencies/cert-manager-operator
helm install sail-operator charts/dependencies/sail-operator
helm install rhcl-operator charts/dependencies/rhcl-operator
helm install postgresql charts/dependencies/postgresql

# Step 2: Main chart with MaaS enabled
helm template kserve-rhaii-xks charts/rhai-on-xks-chart \
  --set azure.enabled=true \
  --set components.maas.enabled=true \
  --set rhaiOperator.image=<operator-image-with-xks-overlay> \
  | kubectl apply -f -

# Step 3: Deploy a model (user step)
kubectl create namespace llm
kubectl apply -f my-llminferenceservice.yaml -n llm

# Step 4: Create MaaS resources (user step)
kubectl apply -f my-maasmodelref.yaml -n llm
kubectl apply -f my-maasauthpolicy.yaml -n models-as-a-service
kubectl apply -f my-maassubscription.yaml -n models-as-a-service
```

## What the chart automates (with `components.maas.enabled=true`)

| Step | Automated by |
|------|-------------|
| Enable MaaS in operator | `RHAI_DISABLE_MODELSASSERVICE_COMPONENT=false` via values |
| Create ModelsAsService CR | Post-install hook (weight 1) |
| Create `openshift-ingress` namespace | MaaS gateway hook (weight 3) |
| CA bundle ConfigMaps (rhai-ca + opendatahub-ca) | MaaS gateway hook |
| MaaS gateway config (Istio proxy CA mounts) | MaaS gateway hook |
| MaaS gateway TLS Certificate | MaaS gateway hook |
| MaaS controller webhook Certificate | MaaS gateway hook |
| MaaS API serving Certificate | MaaS gateway hook |
| MaaS Gateway (Istio, HTTP + HTTPS) | MaaS gateway hook |
| Pull secret to gateway namespace | MaaS gateway hook |
| Authorino CA trust (combined CA bundle) | MaaS gateway hook |
| PostgreSQL database + connection secret | Dependency chart |

## What remains manual (by design)

| Step | Reason |
|------|--------|
| Deploy LLMInferenceService in `llm` namespace | Model-specific (user chooses model, GPU, etc.) |
| Create MaaSModelRef | User defines which models to expose via MaaS |
| Create MaaSAuthPolicy | User defines who has access |
| Create MaaSSubscription | User defines rate limits and ownership |
| Create API keys | User action via MaaS API |

## E2E Test Results (AKS)

| Step | Result |
|------|--------|
| Operator deploys MaaS controller via xKS overlay | Pass — `ModelsAsService` CR Ready: True |
| RHCL operators running (Kuadrant, Authorino, Limitador) | Pass — 0 restarts after RBAC fix |
| MaaS gateway created with Istio | Pass — Programmed with external IP |
| MaaS API deployed by controller | Pass — Running 1/1 |
| Tenant reconciled | Pass — Ready: Reconciled |
| MaaSModelRef Ready | Pass — HTTPRoute on MaaS gateway |
| MaaSAuthPolicy Active | Pass — Auto-generated Kuadrant AuthPolicy |
| MaaSSubscription Active | Pass — TokenRateLimitPolicy ready |
| Create API key via MaaS API | Pass — `sk-oai-...` returned |
| Unauthenticated request | Pass — 401 Unauthorized (RHCL enforced) |
| Inference with API key | Pass — "Kubernetes is an open-source platform..." |

## Architecture

```
User (API key)
    │
    ▼
MaaS Gateway (Istio, openshift-ingress)
    │
    ├── RHCL/Authorino: validates API key, checks subscription
    │
    ├── RHCL/Limitador: enforces TokenRateLimitPolicy
    │
    ▼
Model (Qwen2.5-0.5B via vLLM, llm namespace)
    │
    ▼
Response
```

## Merge Order

PRs must merge in sequence:

1. **models-as-a-service #1015** — xKS overlay gets baked into operator image at build time
2. **opendatahub-operator #3670** — Go code selects the overlay; new operator image is built
3. **odh-gitops #109** — Chart references the new operator image and deploys everything

## Issues Found and Fixed

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| MaaS controller CrashLoopBackOff | Missing webhook TLS cert (OCP service-serving CA doesn't exist on xKS) | Chart post-install hook creates cert-manager Certificate |
| Kuadrant operator crash-loops (436 restarts) | Missing RBAC for `monitoring.coreos.com` | Added podmonitors/servicemonitors to ClusterRole |
| Authorino can't call MaaS API over HTTPS | Authorino doesn't trust rhai-ca-issuer certs | Combined CA bundle mounted + SSL_CERT_FILE env var |
| Model TLS verification fails on MaaS gateway | rhai-ca vs opendatahub-ca mismatch | Both CAs included in CA bundle |
| Webhook cert `$(CERTIFICATE_NAMESPACE)` not resolved | Kustomize variable not substituted by operator | Moved cert creation to chart hook (correct namespace known) |
