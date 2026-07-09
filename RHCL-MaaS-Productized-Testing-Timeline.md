# RHCL & MaaS — Productized Image Testing Timeline

**Context:** Upstream migration and E2E testing is complete. Naina has directed that the next phase should validate on productized images (3.4 GA builds, then 3.5 TP when available).  
**Date:** June 4, 2026

---

## Phase 1: Productized Image Migration (Week 1)

| Task | Estimate | Risk |
|---|---|---|
| Get access to 3.4 GA productized images from `registry.redhat.io` (vLLM, KServe controller, llmisvc-controller, storage-initializer, kserve-agent, kserve-router, Authorino, Limitador, Kuadrant operator, inference-scheduler, routing-sidecar) | 1 day | Pull secret issues, image not published yet, need to contact release engineering for image references |
| Update Helm chart values.yaml and templates to reference all productized image tags instead of upstream (`quay.io/opendatahub` -> `registry.redhat.io/rhaiis` or `registry.redhat.io/rhoai`) | 2 days | Some images may have different entrypoints, env vars, or directory structures than upstream. Image tag naming convention may differ. |
| Deploy on a clean AKS cluster with all productized images | 1 day | Productized images may have different security contexts, different default configs, or missing environment variables that upstream had |
| Debug any deployment failures from productized images | 2 days | Unknown — could be zero issues or could be image pull errors, CRD version mismatches, operator behavior differences |

**Phase 1 total: ~6 days**

---

## Phase 2: E2E Validation on Productized Images (Week 2)

| Task | Estimate | Risk |
|---|---|---|
| Verify all RHCL operators start with productized images (Authorino, Limitador, Kuadrant) | 1 day | Productized operator images may expect different CRD versions or have different RBAC requirements |
| Verify MaaS stack starts with productized images (API, controller, PostgreSQL) | 1 day | MaaS API productized image may have different routes, auth logic, or database schema |
| Deploy model and verify inference works | 1 day | Same modelcar UID / CRD validation / CA secret issues we hit with upstream, but may manifest differently with productized controller |
| Test Azure AD JWT auth flow (unauthenticated blocked, authenticated passes) | 0.5 day | Should work if RHCL operators are same version, but productized Authorino may handle JWT differently |
| Test MaaS subscription flow (create subscription, issue API key, call model with API key) | 1 day | MaaS API header format fix (CEL expressions) may need adjustment if productized MaaS API version differs |
| Test rate limiting (lower limit, send burst, verify 429) | 0.5 day | Should work if Limitador is same version |
| Document results, update PR | 0.5 day | |

**Phase 2 total: ~5.5 days**

---

## Phase 3: KServe Operator Fixes (Separate track — Product/Operator team)

These are issues discovered during upstream testing that were worked around on the cluster. They need to be fixed in the product, not in the Helm chart. Each should be tracked as a separate Jira ticket.

| Issue | Owner | Estimate | Detail |
|---|---|---|---|
| `uidModelcar` not set in `inferenceservice-config` | KServe / rhai-operator team | 2 days | Operator's reconciliation loop needs to include `uidModelcar: 1001` in the storageInitializer config. Without this, modelcar-based OCI models can't deploy. Currently requires scaling down operator to patch manually. |
| CRD validation rejects Go templates in `LLMInferenceServiceConfig` | KServe CRD team | 4 days | The v1alpha2 CRD's regex validation on HTTPRoute path/namespace fields rejects `{{ .ObjectMeta.Namespace }}` template syntax. `router-route` and `scheduler` configs can't be created. Needs CRD schema change or a different templating approach. |
| Missing `rhai-ca` secret | rhai-operator team | 1 day | Operator creates `rhai-ca-issuer` ClusterIssuer referencing `opendatahub-ca` secret, but the controller looks for `rhai-ca`. Either create the secret in the chart or align the `CA_SECRET_NAME` parameter. |
| Broken bash template merge in `kserve-config-llm-template` | KServe llmisvc-controller team | 3 days | RoCE inference script gets concatenated with vLLM serve command producing invalid bash. Template merge logic needs fixing. |
| DestinationRule CA cert not mounted in gateway pod | Istio / chart team | 1.5 days | DestinationRules reference `/var/run/secrets/opendatahub/ca.crt` but the Istio gateway pod doesn't have it mounted. Currently worked around with `insecureSkipVerify: true`. |
| Webhook blocks resource creation when operator is down | rhai-operator team | 1 day | MutatingWebhookConfiguration uses `failurePolicy: Fail`. Should consider `Ignore` or document the maintenance procedure. |

**Phase 3 total: ~12.5 days**

---

## Phase 4: 3.5 TP Build Validation (When builds are available)

| Task | Estimate | Risk |
|---|---|---|
| Swap 3.4 GA images to 3.5 TP images | 1 day | New image tags, possibly new CRDs or API changes |
| Re-run full E2E test suite | 2 days | 3.5 may introduce breaking changes, new features, or different defaults |
| Fix any 3.5-specific issues | 2 days | Unknown scope |

**Phase 4 total: ~5 days**

---

## Summary

| Phase | Estimate | Owner |
|---|---|---|
| Phase 1: Productized image migration | ~6 days | Ecosystem engineering / next person |
| Phase 2: E2E validation on productized images | ~5.5 days | Ecosystem engineering / next person |
| Phase 3: KServe operator fixes | ~12.5 days | KServe / rhai-operator product team |
| Phase 4: 3.5 TP validation | ~5 days | Ecosystem engineering / next person |
| **Total (Phases 1+2, testing scope)** | **~2.5 weeks** | |
| **Total (all phases including KServe fixes)** | **~5.5 weeks** | |

---

## What's Already Done (Upstream)

- All 58 RHCL + MaaS template files migrated from rhaii-on-xks to odh-gitops
- Azure AD JWT auth on gateway — unauthenticated blocked (401), authenticated passes (200)
- MaaS subscription flow — MaaSModelRef + MaaSSubscription created, API key issued, model called with API key (200)
- Rate limiting — Limitador returns 429 after exceeding threshold
- Snapshot tests added and passing
- Helm docs regenerated
- PR submitted: https://github.com/tsisodia10/odh-gitops/pull/1
- KT document with full test evidence available

**Key takeaway:** Phases 1+2 are the productized image testing that Naina is asking about. Phase 3 is KServe product work that blocks clean deployments and should be tracked as separate Jira tickets by the operator team. Phase 4 depends on when 3.5 builds are available.
