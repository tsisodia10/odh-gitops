# MaaS on xKS — End-to-End Deployment Flow

## Starting point: an empty AKS cluster

You have a vanilla AKS cluster with nothing on it. No operators, no CRDs, no AI stuff.

## Step 1: User runs `helm install` for dependency charts

The user (or automation) installs the dependency charts one by one:

```bash
helm install cert-manager charts/dependencies/cert-manager-operator/
helm install gateway-api charts/dependencies/gateway-api/
helm install sail-operator charts/dependencies/sail-operator/
helm install lws-operator charts/dependencies/lws-operator/
helm install rhcl-operator charts/dependencies/rhcl-operator/     # PR #109
helm install postgresql charts/dependencies/postgresql/            # PR #120
```

**What this creates on the cluster:**
- cert-manager operator + its CRDs (can issue TLS certificates)
- Gateway API CRDs (so Istio can create Gateways)
- Sail/Istio operator + Istio CR (service mesh for traffic routing)
- LWS operator (for distributed training workloads)
- RHCL/Kuadrant operators (Authorino + Limitador + DNS + Kuadrant) + Kuadrant CR
- PostgreSQL database (for MaaS API to store subscriptions, API keys)

At this point: operators are running, but there's no RHOAI yet.

## Step 2: User runs `helm install` for the main xKS chart

```bash
helm install rhai charts/rhai-on-xks-chart/ \
  --set azure.enabled=true \
  --set components.maas.enabled=true
```

**What Helm does immediately (template rendering):**
- Creates the RHAI operator Deployment (the brain)
- Creates the cloud manager operator Deployment (manages cloud-specific infra)
- Creates CRDs: KServe, ModelsAsService, Platform, KubernetesEngine
- Creates RBAC (ClusterRoles, ServiceAccounts, etc.)
- Creates pull secrets in all dependency namespaces

**What Helm hooks do after install (in order):**

| Weight | Hook Job | What it does |
|--------|----------|-------------|
| 0 | `post-install-crs-rbac` | Creates ServiceAccount + ClusterRole for the hook jobs |
| 1 | `post-install-crs-job` | Creates 3 CRs via `kubectl apply`: KServe CR, KubernetesEngine CR, **ModelsAsService CR** |
| 2 | `post-install-gateway-job` | Creates the inference Gateway (for regular model serving) |
| 3 | `post-install-maas-gateway-job` | Creates the MaaS Gateway (for authenticated model serving) — PR #120 |

## Step 3: Operators reconcile (automatic, no user action)

Now the CRs exist on the cluster. Operators wake up and start reconciling:

### Cloud Manager sees KubernetesEngine CR

```
KubernetesEngine CR says:
  "certManager: Managed, sailOperator: Managed, lws: Managed"

Cloud Manager checks: are these operators running? Yes.
Cloud Manager: "OK, dependencies satisfied."
```

### RHAI Operator sees KServe CR

```
RHAI Operator reads KServe CR
  → Looks at xKS kustomize overlay
  → Deploys KServe controller, KServe webhook
  → KServe is now ready to serve models
```

### RHAI Operator sees ModelsAsService CR

```
RHAI Operator reads ModelsAsService CR
  → Checks: ODH_PLATFORM_TYPE=XKS                    ← Operator PR #3670
  → Selects deployment/overlays/xks/ overlay          ← MaaS repo PR #1015
  → Overlay includes: MaaS controller + webhook + CRDs + cert-manager certs
  → Operator deploys all of it
  → MaaS controller pod starts, finds its TLS cert (cert-manager created it), goes Running
```

### MaaS Gateway hook runs

```
create-maas-gateway.sh (PR #120):
  1. Creates gateway namespace
  2. Copies CA bundles
  3. Creates TLS certificates via cert-manager
  4. Creates Istio Gateway (HTTP + HTTPS)
  5. Configures Authorino to trust the CA (so it can call MaaS API over HTTPS)
```

## Step 4: User deploys a model (manual)

Now the platform is ready. The user creates their model:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: llama-3-8b
spec:
  modelId: meta-llama/Llama-3-8B-Instruct
  # ... GPU resources, etc.
```

Then enables MaaS access for that model:

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata:
  name: llama-3-8b-ref
spec:
  modelRef: llama-3-8b
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata:
  name: llama-3-8b-auth
spec:
  modelRef: llama-3-8b-ref
  # rate limits, etc.
```

## Step 5: User gets an API key and calls the model

```bash
# Get an API key from MaaS API
curl -X POST https://maas-api.example.com/v1/api-keys \
  -d '{"model": "llama-3-8b", "subscription": "my-sub"}'
# Returns: {"api_key": "sk-abc123..."}

# Call the model through MaaS Gateway
curl https://maas-gateway.example.com/v1/chat/completions \
  -H "Authorization: Bearer sk-abc123..." \
  -d '{"model": "llama-3-8b", "messages": [...]}'
```

The request flows through:

```
User → MaaS Gateway (Istio) → Authorino (validates API key) → Limitador (checks rate limit) → vLLM pod → Response
```

## Summary: who deploys what

| Deployed by | What |
|------------|------|
| **User** (helm install dependency charts) | cert-manager, Istio, Gateway API, LWS, RHCL, PostgreSQL |
| **User** (helm install main chart) | RHAI operator, cloud manager, CRDs, pull secrets |
| **Helm hooks** (automatic) | KServe CR, ModelsAsService CR, KubernetesEngine CR, Gateways, certs |
| **RHAI operator** (automatic) | KServe controller, MaaS controller, MaaS API |
| **Cloud manager** (automatic) | Validates dependencies are running |
| **User** (manual) | Models (LLMInferenceService), MaaS access (MaaSModelRef, MaaSAuthPolicy) |

## PRs in this flow

| Step | PR |
|------|---------|
| Step 1 (RHCL install) | odh-gitops PR #109 — the RHCL dependency chart |
| Step 1 (PostgreSQL install) | odh-gitops PR #120 — the PostgreSQL dependency chart |
| Step 2 (hooks create ModelsAsService CR + MaaS Gateway) | odh-gitops PR #120 — hook jobs and gateway script |
| Step 3 (operator selects xKS overlay) | opendatahub-operator PR #3670 — 3-line Go change |
| Step 3 (overlay has cert-manager certs) | models-as-a-service PR #1015 — xKS kustomize overlay |
