# MaaS on xKS — Knowledge Base

## What is this project?

Red Hat AI Inference (RHAII) is a product that deploys AI/ML model serving on Kubernetes. It currently runs on OpenShift. The goal is to extend it to **vanilla Kubernetes** (AKS, EKS, GKE, CoreWeave) — called "xKS" internally.

The product already works on xKS for **LLM-D** (direct model serving). We are adding **MaaS** (Models as a Service) — the layer that provides API key management, subscriptions, rate limiting, and authentication on top of model serving.

## Key Components

### What the user sees

```
User sends request with API key
  → MaaS Gateway (Istio) — entry point
    → RHCL/Authorino — validates API key, checks subscription access
      → RHCL/Limitador — enforces token rate limits
        → Model (vLLM on GPU) — generates response
          → Response back to user
```

### What runs on the cluster

| Component | What it does | Managed by |
|-----------|-------------|-----------|
| **RHAI Operator** | Reads CRs (Kserve, ModelsAsService) and deploys components | Helm chart |
| **KServe / LLMInferenceService controller** | Manages model serving pods, creates HTTPRoutes | Operator |
| **MaaS Controller** | Manages subscriptions, API keys, auth policies, tenants | Operator (via xKS overlay) |
| **MaaS API** | REST API for creating API keys, listing models | MaaS controller (auto-deployed) |
| **Authorino** | RHCL auth engine — validates JWT tokens, API keys | RHCL dependency chart |
| **Limitador** | RHCL rate limiting engine — enforces token rate limits | RHCL dependency chart |
| **Kuadrant Operator** | Manages Authorino + Limitador lifecycle | RHCL dependency chart |
| **PostgreSQL** | Database for MaaS API (stores subscriptions, keys) | PostgreSQL dependency chart |
| **Inference Gateway** | Direct model access (no auth) | Chart post-install hook |
| **MaaS Gateway** | Managed model access (with auth, rate limiting) | Chart post-install hook |

### MaaS Custom Resources (what users create)

| CR | Namespace | Purpose |
|----|-----------|---------|
| `LLMInferenceService` | `llm` | Deploy a model on GPU |
| `MaaSModelRef` | `llm` (same as model) | Expose model through MaaS gateway |
| `MaaSAuthPolicy` | `models-as-a-service` | Define who can access which models |
| `MaaSSubscription` | `models-as-a-service` | Define rate limits and ownership |

### MaaS Resources auto-created by controller

| Resource | Created by | Purpose |
|----------|-----------|---------|
| `Tenant` | MaaS controller | Bootstraps MaaS platform per namespace |
| `HTTPRoute` | KServe controller | Routes traffic from gateway to model |
| `AuthPolicy` (Kuadrant) | MaaS controller | Enforces auth on the MaaS gateway |
| `TokenRateLimitPolicy` | MaaS controller | Enforces rate limits per subscription |
| MaaS API deployment | MaaS controller | REST API for user-facing operations |

## The 3 Repositories

### 1. models-as-a-service (`opendatahub-io/models-as-a-service`)

**What:** MaaS application code + deployment manifests

**Structure:**
```
maas-controller/     — Go code (subscriptions, API keys, auth policies, tenants)
maas-api/            — REST API server (Go)
deployment/
  base/              — Kustomize base manifests
    maas-controller/
      crd/           — MaaS CRDs (MaaSModelRef, MaaSSubscription, etc.)
      rbac/          — ClusterRoles, ServiceAccounts
      manager/       — Controller Deployment
      webhook/       — ValidatingWebhookConfiguration
      monitoring/    — PodMonitor (OpenShift only)
  overlays/
    openshift/       — OCP overlay (includes networking, monitoring, observability)
    odh/             — ODH overlay
    xks/             — Our xKS overlay (NEW)
```

**Our xKS overlay does 3 things:**
1. Removes OpenShift-specific annotations (`service.beta.openshift.io/*`)
2. Adds volume mount for webhook TLS cert (cert created by chart hook, not overlay)
3. Skips networking, monitoring, observability (handled by chart on xKS)

**Why the overlay is minimal:** On OpenShift, the overlay also creates the MaaS gateway and monitoring resources. On xKS, the chart handles these because they need Istio-specific configuration and cert-manager certs.

### 2. opendatahub-operator (`opendatahub-io/opendatahub-operator`)

**What:** The operator that deploys all RHOAI/RHAII components

**How it builds:**
```
Build time:
  get_all_manifests.sh runs during Docker build
    → Downloads manifests from 15+ component repos (KServe, MaaS, Dashboard, etc.)
    → Copies them to /opt/manifests/<component>/
  Go code compiles
  Everything packaged into operator image

Runtime:
  Operator starts → watches for CRs (Kserve, ModelsAsService)
  When ModelsAsService CR appears:
    → Go code in modelsasservice_support.go decides which kustomize path to use
    → On xKS: /opt/manifests/maas/overlays/xks/
    → On OCP: /opt/manifests/maas/base/maas-controller/default/
    → Runs kustomize build → applies result → MaaS controller deploys
```

**Our change:** 3 lines in `modelsasservice_support.go`:
```go
if cluster.GetClusterInfo().Type == cluster.ClusterTypeKubernetes {
    kPath = filepath.Join(mi.Path, mi.ContextDir, "overlays", "xks")
}
```

**E2E tests already exist:** `tests/e2e/modelsasservice_test.go` — tests Tenant creation, CRD validation, singleton enforcement. Currently OpenShift-only (uses `openshift-default` GatewayClass). dbianchi asked us to enable for xKS.

### 3. odh-gitops (`opendatahub-io/odh-gitops`)

**What:** Helm charts for deploying the entire platform

**Structure:**
```
charts/
  rhai-on-xks-chart/          — Main xKS Helm chart
    values.yaml               — All configuration (components.maas.enabled, etc.)
    templates/
      manager/                — Operator deployment
      hooks/                  — Post-install jobs (CRs, gateway, MaaS gateway)
      crds/                   — CRDs (Kserve, ModelsAsService, Platform)
      rbac/                   — Operator ClusterRole
    files/
      create-gateway.sh       — Inference gateway setup script
      create-maas-gateway.sh  — MaaS gateway setup script (NEW)
  rhai-on-openshift-chart/    — OpenShift chart (OLM-based, different delivery)
  dependencies/
    cert-manager-operator/    — cert-manager for TLS
    sail-operator/            — Istio for service mesh
    gateway-api/              — Gateway API CRDs
    lws-operator/             — LeaderWorkerSet for distributed training
    rhcl-operator/            — RHCL (Kuadrant, Authorino, Limitador) (NEW)
    postgresql/               — PostgreSQL for MaaS API database (NEW)
```

**How deployment works:**
```bash
# Step 1: Dependencies (separate helm installs)
helm install cert-manager-operator charts/dependencies/cert-manager-operator
helm install sail-operator charts/dependencies/sail-operator
helm install rhcl-operator charts/dependencies/rhcl-operator
helm install postgresql charts/dependencies/postgresql

# Step 2: Main chart
helm template kserve-rhaii-xks charts/rhai-on-xks-chart \
  --set azure.enabled=true \
  --set components.maas.enabled=true \
  | kubectl apply -f -

# This triggers post-install hooks:
#   Weight 0: RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding)
#   Weight 1: CRs (Kserve, ModelsAsService, KubernetesEngine)
#   Weight 2: Inference gateway (CA bundle, certs, Gateway)
#   Weight 3: MaaS gateway (CA bundle, certs, Gateway, Authorino CA trust)
```

## What We Implemented

### PR #1015 (models-as-a-service) — xKS overlay
- `deployment/overlays/xks/kustomization.yaml`
- Patches out OCP annotations, adds webhook cert volume mount
- Minimal by design — chart handles the rest

### PR #3670 (opendatahub-operator) — overlay selection
- 3-line Go change in `modelsasservice_support.go`
- Selects `overlays/xks` when `ODH_PLATFORM_TYPE=XKS`

### PR #109 (odh-gitops) — chart integration
- RHCL dependency chart (14 CRDs, 4 operator deployments, RBAC, ServiceMonitors)
- PostgreSQL dependency chart
- `components.maas.enabled` values flag
- Values-driven `RHAI_DISABLE_MODELSASSERVICE_COMPONENT`
- ModelsAsService CRD in `templates/crds/`
- ModelsAsService CR creation in post-install hook
- MaaS gateway post-install hook (`create-maas-gateway.sh`):
  - CA bundles (rhai-ca + opendatahub-ca)
  - Gateway config (Istio proxy CA mounts at both paths)
  - TLS certificates (gateway, webhook, API serving)
  - MaaS Gateway with Istio GatewayClass
  - Pull secret propagation
  - Authorino CA trust (combined CA bundle + SSL_CERT_FILE)
- Kuadrant RBAC fix (monitoring.coreos.com)
- Hook RBAC for modelsasservices + authorino
- Pull secret propagation to kuadrant namespaces
- Snapshot tests (azure-with-maas, azure-with-maas-and-pull-secret)

## Issues We Found and Solved

### 1. MaaS controller CrashLoopBackOff
**Symptom:** Controller pod kept crashing with `open /tmp/k8s-webhook-server/serving-certs/tls.crt: no such file or directory`
**Root cause:** On OpenShift, service-serving CA auto-injects webhook TLS certs. On xKS, nothing does.
**Fix:** Chart post-install hook creates a cert-manager Certificate. xKS overlay adds volume mount to project the secret into the container.

### 2. Kuadrant operator crash-loop (436 restarts)
**Symptom:** `kuadrant-operator-controller-manager` CrashLoopBackOff, `Could not wait for Cache to sync: *v1.PodMonitor`
**Root cause:** Operator's ClusterRole was missing `monitoring.coreos.com` RBAC rules. It tried to watch PodMonitors/ServiceMonitors but couldn't list them.
**Fix:** Added podmonitors/servicemonitors verbs to `clusterrole-kuadrant-operator.yaml`.

### 3. Authorino can't call MaaS API (AUTH_FAILURE)
**Symptom:** API key passes Authorino auth, but MaaS API returns AUTH_FAILURE with empty identity headers.
**Root cause:** Authorino's metadata callbacks to MaaS API fail over HTTPS because Authorino doesn't trust the rhai-ca-issuer (self-signed CA). Headers never get injected.
**Fix:** Combined CA bundle (system CA + rhai CA) mounted into Authorino via `spec.volumes`, SSL_CERT_FILE env var set.

### 4. Model TLS verification fails on MaaS gateway
**Symptom:** Gateway returns `CERTIFICATE_VERIFY_FAILED` when proxying to model.
**Root cause:** Two different CAs on the cluster — `rhai-ca` (chart) and `opendatahub-ca` (KServe). CA bundle only had `rhai-ca`.
**Fix:** `create-maas-gateway.sh` checks for `opendatahub-ca` secret and appends it to the bundle.

### 5. Webhook cert namespace variable not resolved
**Symptom:** Cert DNS names contained literal `$(CERTIFICATE_NAMESPACE)` instead of actual namespace.
**Root cause:** Kustomize variable used in the overlay's Certificate resource wasn't substituted by operator's kustomize build.
**Fix:** Moved cert creation to chart hook where the namespace is known. Overlay only does the volume mount.

### 6. MaaS subscription stuck in Failed
**Symptom:** `HTTPRoute not found for model, skipping TokenRateLimitPolicy creation`
**Root cause:** MaaS controller expects a `maas-default-gateway` Gateway in `openshift-ingress` — didn't exist on xKS.
**Fix:** Chart post-install hook creates the gateway with Istio GatewayClass.

### 7. Model namespace mismatch
**Symptom:** MaaSModelRef stuck in Pending, "Waiting for HTTPRoute to be created"
**Root cause:** Controller code: `llmisvcNS := model.Namespace` — looks for LLMInferenceService in same namespace as ModelRef. Model was in `model-serving`, ModelRef in `models-as-a-service`.
**Fix:** Deploy models in `llm` namespace. This also fixes the AuthPolicy path extraction (expects `/llm/<model>`).

## Issues Reported to Other Teams

| Issue | Team | Status |
|-------|------|--------|
| KServe DestinationRule uses `opendatahub` CA path instead of `rhai` | KServe/Platform | Workaround: mount CA at both paths |
| Operator keeps reverting webhook cert on reconcile | Operator/MaaS | Workaround: chart creates correct cert, operator will use it once PR merges |
| MaaS controller hardcodes `opendatahub` namespace for DB secret | MaaS team (RHOAIENG-69604) | Known issue, E2E tests skip on RHOAI |

## What Still Needs To Be Done

| Item | Who | Effort |
|------|-----|--------|
| Update RHCL images from 1.3 to 1.4 | Us | Trivial (version bump) |
| Update MaaS images to 3.5 tags | Us | Trivial (version bump) |
| E2E tests in CI (`test-kind-odh-e2e.yaml`) | Us + dbianchi | 1-2 days |
| RHCL support agreement | Naina/Matthew/Jonathan | Not our scope |
| Official product documentation | Doc team (Shamela) | Not our scope |
| Merge PRs in order (#1015 → #3670 → #109) | Reviewers | Waiting on reviews |
| Production operator image via Konflux | Automatic after merges | Automatic |

## Glossary

| Term | Meaning |
|------|---------|
| **xKS** | Non-OpenShift Kubernetes (AKS, EKS, GKE, CoreWeave) |
| **MaaS** | Models as a Service — API key management, subscriptions, rate limiting |
| **RHCL** | Red Hat Connectivity Link — Kuadrant, Authorino, Limitador (auth + rate limiting) |
| **RHAII / RyAI** | Red Hat AI Inference — the product |
| **RHOAI** | Red Hat OpenShift AI — the OpenShift-only product |
| **LLM-D** | LLM serving via KServe (direct model access, no MaaS) |
| **OLM** | Operator Lifecycle Manager — OpenShift's way to install operators (not available on xKS) |
| **Kuadrant** | API gateway policy engine (part of RHCL) |
| **Authorino** | Auth engine — validates JWT, API keys, injects identity headers |
| **Limitador** | Rate limiting engine — enforces token/request rate limits |
| **KServe** | Kubernetes-native model serving platform |
| **LLMInferenceService** | KServe CR for deploying an LLM |
| **Gateway API** | Kubernetes standard for ingress (Gateway, HTTPRoute, GatewayClass) |
| **Istio** | Service mesh used on xKS (provides GatewayClass, mTLS, traffic management) |
| **cert-manager** | Kubernetes native certificate management |
| **Konflux** | Red Hat's CI/CD system that builds operator images |
| **TP / DP** | Tech Preview / Dev Preview — release maturity levels |
