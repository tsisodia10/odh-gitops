# RHCL & MaaS — Execution Plan for Productized Image Validation

**Date:** June 4, 2026  
**Owner:** Twinkll Sisodia  
**Estimated Duration:** 8-10 working days

---

## Current Status

13 of 15 images are already productized. Only 2 MaaS images need swapping:

| Component | Current Image | Target |
|---|---|---|
| MaaS API | `quay.io/opendatahub/maas-api:latest` | `registry.redhat.io/rhoai/odh-maas-api-rhel9:<3.4-tag>` |
| MaaS Controller | `quay.io/opendatahub/maas-controller:latest` | `registry.redhat.io/rhoai/odh-maas-controller-rhel9:<3.4-tag>` |

All other images (RHCL operators, Authorino, Limitador, vLLM, PostgreSQL) are already running from `registry.redhat.io`.

---

## Tasks

### 1. Unblock Registry Access (~1-2 days)

| Task |
|---|
| Request registry.redhat.io access for `rhoai/odh-maas-*` images from release/entitlements team |
| Wait for access to be granted (external dependency) |
| Verify pull access works by pulling the image manifest |

### 2. Identify Correct Image Tags + Update Chart (~1 day)

| Task |
|---|
| Find correct 3.4 GA tags for `odh-maas-api-rhel9` and `odh-maas-controller-rhel9` — may need coordination with release engineering |
| Update `values.yaml` with productized MaaS image references |
| Update pull secret on cluster to include rhoai entitlement |

### 3. Deploy and Validate Components (~2 days)

| Task |
|---|
| Helm upgrade with productized MaaS images |
| Verify MaaS API and MaaS Controller pods start with productized images |
| Debug any startup failures (different env vars, security contexts, entrypoints) |
| Verify all RHCL operators still healthy (Authorino, Limitador, Kuadrant) |
| Verify gateway programmed and policies enforced |
| Deploy test model (Qwen2.5-0.5B on T4 GPU) |
| Apply cluster workarounds if needed (uidModelcar, rhai-ca secret, DestinationRule TLS) |

### 4. E2E Auth + MaaS Subscription + Rate Limiting (~2-3 days)

| Task |
|---|
| Get Azure AD token, test unauthenticated request blocked (401), authenticated request passes (200) |
| Test MaaS API `/v1/models` with auth token |
| Create MaaSModelRef + MaaSSubscription, generate API key, call model with API key through gateway |
| Test rate limiting (lower limit, send burst, verify 429) |
| Debug any differences between productized and upstream behavior |

### 5. Documentation and PR (~1 day)

| Task |
|---|
| Regenerate snapshot tests with productized image refs |
| Run `make chart-test` and `make helm-docs` |
| Update PR with productized image changes |
| Update KT doc with productized image test results and evidence |

---

## Dependencies

| Dependency | Status | Impact |
|---|---|---|
| Registry access to `rhoai/odh-maas-*` images | **BLOCKED** | Cannot proceed without this |
| Correct image tags identified | Unknown | Need release engineering confirmation |
| Current upstream PR merged or ready for update | Under review | Productized changes build on this PR |
| AKS cluster + GPU node available | Running | Needed for model deployment |

---

## Definition of Done

- [ ] MaaS API running with productized image from registry.redhat.io
- [ ] MaaS Controller running with productized image from registry.redhat.io
- [ ] Unauthenticated request blocked (401)
- [ ] Authenticated request with Azure AD JWT reaches model (200)
- [ ] MaaS subscription flow works (API key issued, model called, 200)
- [ ] Rate limiting enforced (429)
- [ ] Snapshot tests passing
- [ ] PR updated
- [ ] KT doc updated with evidence
