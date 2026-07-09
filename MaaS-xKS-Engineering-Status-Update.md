# MaaS on xKS: Engineering Status Update

**RHAISTRAT-1377 | RHAIRFE-1236 | Target: RHOAI 3.5 (August 2026)**

**Author:** Twinkll Sisodia (Ecosystem Engineering) | **Date:** June 23, 2026

**Purpose:** Engineering update on implementation progress, gap closure, and remaining items. Complements Naina Singh's readiness assessment (June 18, 2026).

---

## Gap Closure Summary

Naina's readiness assessment identified 6 TP gaps. Here is the current engineering status:

| # | TP Gap (from assessment) | Status | Evidence |
|---|--------------------------|--------|----------|
| 1 | MaaS xKS overlay in operator | **CLOSED** | PR [#1015](https://github.com/opendatahub-io/models-as-a-service/pull/1015) + PR [#3670](https://github.com/opendatahub-io/opendatahub-operator/pull/3670) |
| 2 | RHCL dependency chart merged | **CLOSED** (awaiting review) | PR [#109](https://github.com/opendatahub-io/odh-gitops/pull/109), rebased, snapshot tests passing |
| 3 | KServe bugs resolved | **2 of 3 CLOSED** | #5 workaround automated, #6 resolved. #1 (uidModelcar) open but HF models unaffected |
| 4 | Zero-workaround E2E | **MOSTLY CLOSED** | Full user journey validated on AKS with API key auth and inference |
| 5 | RHCL support agreement | **OPEN** (PM-level) | Not engineering scope. Template exists from llm-d. |
| 6 | Documentation on product site | **OPEN** (doc team) | Deployment guide and test reports available as gists |

---

## PRs Raised

### PR #1015 — models-as-a-service ([link](https://github.com/opendatahub-io/models-as-a-service/pull/1015))

**xKS kustomize overlay for MaaS controller**

- Deploys MaaS controller on vanilla Kubernetes (AKS, EKS, GKE)
- Patches out OpenShift-specific annotations (`service.beta.openshift.io/*`)
- Adds deployment volume mount for webhook TLS cert
- Skips OpenShift-only resources: networking (uses Istio), monitoring (PodMonitor), observability

| What | Detail |
|------|--------|
| Files | `deployment/overlays/xks/kustomization.yaml` (1 file) |
| Pattern followed | Same as KServe's `overlays/odh-xks` (Luca Burgazzoli) |
| Reviewers | SB159, ryancham715, ishitasequeira, jland-redhat |
| Base branch | `main` (corrected from `stable` per ishitasequeira's feedback) |

### PR #3670 — opendatahub-operator ([link](https://github.com/opendatahub-io/opendatahub-operator/pull/3670))

**Go code to select xKS overlay**

3-line change in `modelsasservice_support.go`:
```go
if cluster.GetClusterInfo().Type == cluster.ClusterTypeKubernetes {
    kPath = filepath.Join(mi.Path, mi.ContextDir, "overlays", "xks")
}
```

| What | Detail |
|------|--------|
| Pattern followed | Same as KServe's overlay selection in `kserve_support.go` |
| Reviewers | dbianchi, platform team |
| Open ask from reviewer | dbianchi requested E2E test enablement for xKS (see Remaining Items) |

### PR #109 — odh-gitops ([link](https://github.com/opendatahub-io/odh-gitops/pull/109))

**RHCL dependency chart + MaaS chart integration**

| Change | File(s) | Pattern followed |
|--------|---------|-----------------|
| RHCL (Kuadrant) as dependency chart | `charts/dependencies/rhcl-operator/` | Same as cert-manager-operator, sail-operator |
| PostgreSQL as dependency chart | `charts/dependencies/postgresql/` | Same pattern |
| `components.maas.enabled` values flag | `values.yaml` | Same as `components.kserve.enabled` |
| Values-driven operator env var | `deployment-rhods-operator.yaml` | Same as other `RHAI_DISABLE_*` vars |
| ModelsAsService CRD | `templates/crds/` | Same as Kserve CRD |
| ModelsAsService CR in post-install hook | `post-install-crs-job.yaml` | Same as Kserve CR block |
| MaaS gateway post-install hook | `create-maas-gateway.sh` + job template | Same as `create-gateway.sh` |
| Kuadrant RBAC fix | `clusterrole-kuadrant-operator.yaml` | Missing rules added |
| Hook RBAC for modelsasservices | `post-install-crs-rbac.yaml` | Extended existing ClusterRole |
| Pull secret propagation | `values.yaml` | Added kuadrant namespaces to existing list |
| Snapshot tests | `snapshot-config.yaml` | `azure-with-maas`, `azure-with-maas-and-pull-secret` |

---

## E2E Test Results (AKS, June 18 2026)

Tested on AKS with Standard_NC4as_T4_v3 GPU nodes. Custom operator image built integrating all 3 PRs.

| Test | Result |
|------|--------|
| Operator deploys MaaS controller via xKS overlay | **Pass** — `ModelsAsService` CR Ready: True |
| RHCL operators running | **Pass** — All 4 operators Running, 0 restarts (after RBAC fix) |
| MaaS gateway created with Istio | **Pass** — Programmed with external IP |
| MaaS API deployed by controller | **Pass** — Running 1/1 |
| Tenant reconciled | **Pass** — Ready: Reconciled |
| MaaSModelRef | **Pass** — Ready, HTTPRoute on MaaS gateway |
| MaaSAuthPolicy | **Pass** — Active, auto-generated Kuadrant AuthPolicy |
| MaaSSubscription | **Pass** — Active, TokenRateLimitPolicy ready |
| Create API key via MaaS API | **Pass** — `sk-oai-...` returned |
| Unauthenticated request | **Pass** — 401 Unauthorized (RHCL enforced) |
| Inference with API key | **Pass** — Model response: "Kubernetes is an open-source platform..." |

**Request flow validated:**
```
User (API key) → MaaS Gateway (Istio) → RHCL/Authorino (validates API key, checks subscription)
  → RHCL/Limitador (enforces rate limits) → Model (Qwen2.5-0.5B via vLLM) → Response
```

---

## Issues Found and Resolved

### Resolved by our PRs

| # | Issue | Root cause | Fix |
|---|-------|-----------|-----|
| 1 | MaaS controller CrashLoopBackOff | Webhook TLS cert missing (OCP service-serving CA doesn't exist on xKS) | Chart post-install hook creates cert-manager Certificate; overlay adds volume mount |
| 2 | Kuadrant operator crash-loop (436 restarts) | Missing RBAC for `monitoring.coreos.com` (PodMonitor/ServiceMonitor) | Added rules to ClusterRole in RHCL dependency chart |
| 3 | Authorino can't call MaaS API over HTTPS | Authorino doesn't trust `rhai-ca-issuer` certs for HTTPS callbacks | Combined CA bundle (system + rhai CA) mounted into Authorino via gateway hook |
| 4 | Model TLS verification fails on MaaS gateway | Two different CAs: `rhai-ca` (chart) vs `opendatahub-ca` (KServe) | Both CAs included in CA bundle by gateway hook |
| 5 | Webhook cert `$(CERTIFICATE_NAMESPACE)` not resolved | Kustomize variable not substituted by operator | Moved cert creation to chart hook where namespace is known |
| 6 | MaaS subscription stuck in Failed | No `maas-default-gateway` Gateway on xKS | Chart post-install hook creates gateway with Istio GatewayClass |
| 7 | PostgreSQL not provisioned | No auto-provisioning on xKS | Added PostgreSQL as dependency chart |

### Reported to other teams (open)

| # | Issue | Owner | Workaround |
|---|-------|-------|-----------|
| 1 | uidModelcar not set (OCI modelcar only) | KServe (INFERENG-7332) | Use HuggingFace models instead of OCI modelcar |
| 2 | KServe DestinationRule uses `opendatahub` CA path | KServe/Platform | Gateway hook mounts CA at both paths |
| 3 | MaaS controller hardcodes `opendatahub` namespace for DB | MaaS team (RHOAIENG-69604) | Works on ODH; skipped on RHOAI E2E |

---

## Updated Architecture Diagram

```
Customer runs: helm install --set components.maas.enabled=true

  Helm chart (odh-gitops)
    ├── Deploys RHAI operator (with MaaS enabled)
    ├── Post-install hook: Creates Kserve CR + ModelsAsService CR
    ├── Post-install hook: Creates inference gateway (CA bundle, TLS, Istio)
    └── Post-install hook: Creates MaaS gateway (CA bundle, TLS, Istio, Authorino CA trust)

  RHAI Operator (opendatahub-operator)
    ├── Sees ModelsAsService CR → selects overlays/xks (our Go change)
    ├── Deploys MaaS controller + webhook (from models-as-a-service overlay)
    └── MaaS controller auto-creates: Tenant, MaaS API, AuthPolicies, HTTPRoutes

  Dependencies (separate helm installs)
    ├── cert-manager-operator — TLS certificates
    ├── sail-operator / Istio — Gateway API, service mesh
    ├── rhcl-operator — Kuadrant, Authorino (auth), Limitador (rate limiting)
    └── postgresql — Database for MaaS API

  User actions (manual, by design)
    ├── Deploy LLMInferenceService in 'llm' namespace
    ├── Create MaaSModelRef, MaaSAuthPolicy, MaaSSubscription
    └── Create API keys via MaaS API → Use for authenticated inference
```

---

## Merge Order

PRs must merge in sequence due to the operator image build pipeline:

1. **PR #1015** (models-as-a-service) — xKS overlay gets baked into operator image
2. **PR #3670** (opendatahub-operator) — Go code selects the overlay; new operator image built
3. **PR #109** (odh-gitops) — Chart references the new operator image

---

## Remaining Items

| Item | Owner | Effort | Blocks |
|------|-------|--------|--------|
| PR reviews and merges | Reviewers (dbianchi, MaaS team) | — | Everything |
| Image version bump (RHCL 1.3 → 1.4, MaaS to 3.5 tags) | Ecosystem Engineering | Trivial | Need confirmed tags |
| E2E tests in CI (`test-kind-odh-e2e.yaml`) | Ecosystem Engineering + dbianchi | 1-2 days | dbianchi asked on PR #3670 |
| RHCL support agreement | PM (Naina/Matthew/Jonathan) | — | TP release |
| Official product documentation | Doc team (Shamela) | — | TP release |
| Production operator image via Konflux | Automatic | — | After PRs merge |
| uidModelcar fix (INFERENG-7332) | KServe team | — | OCI modelcar models only |

---

## Deployment Instructions

```bash
# Dependencies
helm install cert-manager-operator charts/dependencies/cert-manager-operator
helm install sail-operator charts/dependencies/sail-operator
helm install rhcl-operator charts/dependencies/rhcl-operator
helm install postgresql charts/dependencies/postgresql

# Main chart with MaaS enabled
helm template kserve-rhaii-xks charts/rhai-on-xks-chart \
  --set azure.enabled=true \
  --set components.maas.enabled=true \
  | kubectl apply -f -

# Deploy a model
kubectl create namespace llm
kubectl apply -f llminferenceservice.yaml -n llm

# Create MaaS resources
kubectl apply -f maasmodelref.yaml -n llm
kubectl apply -f maasauthpolicy.yaml -n models-as-a-service
kubectl apply -f maassubscription.yaml -n models-as-a-service

# Create API key and test
curl -sk https://maas-api.<apps-ns>.svc:8443/v1/api-keys \
  -H "x-maas-username: my-user" \
  -H 'x-maas-group: ["system:authenticated"]' \
  -H "x-maas-tenant: models-as-a-service" \
  -X POST -H "Content-Type: application/json" \
  -d '{"name":"my-key","subscription":"my-sub","expiresIn":"24h"}'

# Inference with API key
curl http://<MAAS_GATEWAY_IP>/llm/<model>/v1/chat/completions \
  -H "Authorization: Bearer sk-oai-..." \
  -H "Content-Type: application/json" \
  -d '{"model":"<model>","messages":[{"role":"user","content":"What is Kubernetes?"}]}'
```

---

## Key References

**PRs:**
- [models-as-a-service #1015](https://github.com/opendatahub-io/models-as-a-service/pull/1015) — xKS overlay
- [opendatahub-operator #3670](https://github.com/opendatahub-io/opendatahub-operator/pull/3670) — Operator Go code
- [odh-gitops #109](https://github.com/opendatahub-io/odh-gitops/pull/109) — RHCL chart + MaaS enablement

**Jira:**
- RHAISTRAT-1377: RHAII MaaS on xKS
- RHAIRFE-1236: RHAII MaaS on xKS (RFE)
- INFERENG-7332: uidModelcar issue
- RHOAIENG-67048: Platform CRD missing
- RHOAIENG-69604: MaaS controller hardcodes namespace

**Slack:** #forum-ai-engineering-rhaii

**Contacts:**
- Twinkll Sisodia — Implementation (Ecosystem Engineering)
- dbianchi — Operator/platform review
- Chaitanya Kulkarni, Ishita Sequeira — MaaS team guidance
- Naina Singh, Matthew Chiccino — PM/scoping
- Anish Asthana — llm-d xKS reference
