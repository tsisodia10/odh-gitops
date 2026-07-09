# RHCL & MaaS on xKS — Issue Report

**Date:** June 8, 2026  
**Cluster:** AKS `swkale-llmd-rg-1` (East US), Tesla T4 GPU (Standard_NC4as_T4_v3)  
**Chart:** rhai-on-xks-chart v3.5.0-ea.1 (rebased to latest upstream/main on June 8)  
**Branch:** `feat/rhcl-maas-integration`

---

## Images Used

| Component | Image | Source |
|---|---|---|
| MaaS API | `registry.redhat.io/rhoai/odh-maas-api-rhel9:v3.4.0` | Productized |
| MaaS Controller | `registry.redhat.io/rhoai/odh-maas-controller-rhel9:v3.4.0` | Productized |
| Authorino Operator | `registry.redhat.io/rhcl-1/authorino-rhel9-operator@sha256:0b60bb...` | Productized |
| Kuadrant Operator | `registry.redhat.io/rhcl-1/rhcl-rhel9-operator@sha256:1afb2f...` | Productized |
| Limitador Operator | `registry.redhat.io/rhcl-1/limitador-rhel9-operator@sha256:6092c5...` | Productized |
| Authorino Runtime | `registry.redhat.io/rhcl-1/authorino-rhel9@sha256:be7790...` | Productized |
| Limitador Runtime | `registry.redhat.io/rhcl-1/limitador-rhel9@sha256:aff28d...` | Productized |
| vLLM | `registry.redhat.io/rhaiis/vllm-cuda-rhel9@sha256:fc68d6...` | Productized |
| PostgreSQL | `registry.redhat.io/rhel9/postgresql-15:latest` | Productized |
| rhai-operator | `quay.io/opendatahub/opendatahub-operator:latest` | Upstream |
| llmisvc-controller | `quay.io/opendatahub/odh-kserve-llmisvc-controller:odh-stable` | Upstream |
| kserve-controller | `ghcr.io/opendatahub-io/rhaii-on-xks/kserve-controller:e6b5db0` | Upstream |

---

## Issue Summary

| # | Issue | Status | Severity |
|---|---|---|---|
| 1 | uidModelcar not set in inferenceservice-config | Open (acknowledged by team) | Medium (only affects OCI modelcar models) |
| 2 | CRD validation rejects Go templates | **Resolved** | N/A |
| 3 | Missing rhai-ca secret | **Resolved** | N/A |
| 4 | Template bash merge broken | **Resolved** | N/A |
| 5 | DestinationRule CA cert not mounted in gateway | Open | High (blocks all gateway traffic to models) |
| 6 | rhai-operator crash + webhook blocks model creation | Open | High (blocks model deployment) |

---

## Resolved Issues

### Issue #2: CRD validation rejects Go templates in LLMInferenceServiceConfig

**Original error (June 2):**
```
LLMInferenceServiceConfig.serving.kserve.io "v3-5-0-ea-1-kserve-config-llm-router-route" not found
```
The `router-route` and `scheduler` configs were missing from `redhat-ods-applications`.

**Current status (June 8, after rebase):**
All 14 LLMInferenceServiceConfigs exist. Kserve CR shows `phase: Ready`. There is a brief transient error during reconciliation that self-resolves within ~2 minutes. No manual intervention needed.

**Conclusion:** Resolved in current builds.

---

### Issue #3: Missing rhai-ca secret in cert-manager namespace

**Original error (June 2):**
```
ReconcileCertsError: failed to get CA secret cert-manager/rhai-ca: Secret "rhai-ca" not found
```

**Current status (June 8):**
The `rhai-ca` secret is correctly created by the operator. The fix was added in PR #3419 (`RHAI_CA_SECRET_NAME=rhai-ca` env var in operator deployment). Confirmed working on current build.

**Conclusion:** Resolved. Fix is in the chart at `templates/manager/deployment-rhods-operator.yaml` lines 101-102.

---

### Issue #4: LLMInferenceServiceConfig template generates broken bash when merged

**Original error (June 2):**
```
eval: line 133: syntax error near unexpected token 'then'
vllm serve /mnt/models ... if [ "$KSERVE_INFER_ROCE" = "true" ]; then
```
The RoCE inference script was concatenated with the vLLM serve command.

**How we reproduced (June 2):** Deployed an LLMInferenceService without explicit `command`/`args` override. The template merged incorrectly.

**Current status (June 8):**
Deployed a model without any command/args override. The template merged correctly — command is `["/bin/bash", "-c"]` with the full RoCE script and `vllm serve` properly structured. vLLM starts and serves health checks.

**Conclusion:** Resolved in current `opendatahub-operator:latest` / `llmisvc-controller:odh-stable` builds.

---

## Open Issues

### Issue #1: uidModelcar not set in inferenceservice-config

**Severity:** Medium — only affects OCI modelcar models (`oci://` URI), not HuggingFace models (`hf://` URI)

**Description:**
The rhai-operator reconciles the `inferenceservice-config` ConfigMap in `redhat-ods-applications` but does not include `uidModelcar`. The modelcar sidecar runs as root (uid 0) while the main vLLM container runs as uid 1001. The modelcar creates a symlink via `/proc/<pid>/root/models` which requires same-UID access.

**How to reproduce:**
```bash
# Check the ConfigMap
kubectl get configmap inferenceservice-config -n redhat-ods-applications \
  -o jsonpath='{.data.storageInitializer}' | python3 -m json.tool | grep uid
# Output: (no uidModelcar field)

# Deploy a model with oci:// URI
kubectl apply -f - <<EOF
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: test-modelcar
  namespace: model-serving
spec:
  model:
    name: test
    uri: "oci://registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5"
  replicas: 1
EOF

# Check pod logs — main container will show:
# ls: cannot access '/mnt/models/': Permission denied
```

**Impact:** Cannot use OCI modelcar-based models (the standard recommended approach) without manually patching the ConfigMap. HuggingFace models (`hf://` URI) use storage-initializer and are not affected.

**Workaround:**
```bash
kubectl scale deploy rhai-operator -n redhat-ods-operator --replicas=0
kubectl patch configmap inferenceservice-config -n redhat-ods-applications --type=merge \
  -p '{"data":{"storageInitializer":"{\"cpuLimit\":\"1\",\"cpuModelcar\":\"10m\",\"cpuRequest\":\"100m\",\"enableModelcar\":true,\"image\":\"quay.io/opendatahub/kserve-storage-initializer:odh-stable\",\"memoryLimit\":\"24Gi\",\"memoryModelcar\":\"15Mi\",\"memoryRequest\":\"100Mi\",\"uidModelcar\":1001}"}}'
kubectl scale deploy rhai-operator -n redhat-ods-operator --replicas=1
```

**Team acknowledgement:** Team confirmed they are aware of this issue.

---

### Issue #5: DestinationRule CA cert not mounted in gateway pod

**Severity:** High — blocks all authenticated requests through the gateway to models

**Description:**
When the KServe controller creates a DestinationRule for a model's workload service, it references a CA certificate at `/var/run/secrets/rhai/ca.crt`. This path does not exist in the Istio gateway pod (`maas-default-gateway-istio`), causing TLS failures when the gateway tries to connect to the model backend.

**How to reproduce:**
```bash
# 1. Deploy a model (any model that reaches Ready state)

# 2. Check the DestinationRule
kubectl get destinationrule -n model-serving -o json | jq '.items[0].spec.trafficPolicy.tls'
# Output:
# {
#   "caCertificates": "/var/run/secrets/rhai/ca.crt",
#   "insecureSkipVerify": false,
#   "mode": "SIMPLE",
#   "sni": "...-kserve-workload-svc.model-serving.svc.cluster.local"
# }

# 3. Check the gateway pod — the CA cert path doesn't exist
kubectl exec -n istio-system -l app=maas-default-gateway-istio -- \
  ls /var/run/secrets/rhai/ca.crt 2>&1
# Output: No such file or directory

# 4. Send authenticated request through gateway
kubectl run test --rm -i --restart=Never --image=curlimages/curl -- \
  curl -sv --max-time 10 \
  "http://maas-default-gateway-istio.istio-system.svc/model-serving/<model>/v1/chat/completions" \
  -H "Host: inference.maas.local" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"model":"<model>","messages":[{"role":"user","content":"Hi"}],"max_tokens":5}'
# Output:
# HTTP/1.1 503 Service Unavailable
# upstream connect error: TLS error: Secret is not supplied by SDS
```

**Gateway proxy logs:**
```
error  cache  resource:file-root:/var/run/secrets/rhai/ca.crt failed to generate secret for proxy from file:
  open /var/run/secrets/rhai/ca.crt: no such file or directory
```

**Impact:** No authenticated request can reach a model through the MaaS gateway. Direct requests to the model service (bypassing gateway) work fine.

**Team response:** The team mentioned MaaS Gateways should follow the same setup as inference gateways. Reference: https://github.com/opendatahub-io/odh-gitops/tree/main/charts/rhai-on-xks-chart#inference-gateway

**Workaround:**
```bash
# Scale down controllers to prevent reconciliation
kubectl scale deployment kserve-controller-manager -n opendatahub --replicas=0
kubectl scale deployment llmisvc-controller-manager -n redhat-ods-applications --replicas=0

# Patch DestinationRule
kubectl patch destinationrule <model>-kserve-workload-svc -n model-serving --type=merge \
  -p '{"spec":{"trafficPolicy":{"tls":{"insecureSkipVerify":true}}}}'
kubectl patch destinationrule <model>-kserve-workload-svc -n model-serving --type=json \
  -p '[{"op":"remove","path":"/spec/trafficPolicy/tls/caCertificates"},{"op":"remove","path":"/spec/trafficPolicy/tls/sni"}]'

# Restart gateway
kubectl rollout restart deployment maas-default-gateway-istio -n istio-system

# Scale controllers back
kubectl scale deployment kserve-controller-manager -n opendatahub --replicas=1
kubectl scale deployment llmisvc-controller-manager -n redhat-ods-applications --replicas=1
```

---

### Issue #6: rhai-operator CrashLoopBackOff + webhook blocks model creation

**Severity:** High — blocks all LLMInferenceService creation

**Description:**
The rhai-operator (`quay.io/opendatahub/opendatahub-operator:latest`) crashes on startup with:
```
no matches for kind "Platform" in version "config.opendatahub.io/v1alpha1"
```
The operator expects a `Platform` CRD that doesn't exist on the xKS cluster. Because the operator's MutatingWebhookConfiguration uses `failurePolicy: Fail`, all LLMInferenceService create/update operations fail with:
```
failed to call webhook: no endpoints available for service "rhai-operator-webhook-service"
```

**How to reproduce:**
```bash
# Check operator status
kubectl get pods -n redhat-ods-operator
# Output: rhai-operator-xxx  0/1  CrashLoopBackOff

# Check operator logs
kubectl logs -n redhat-ods-operator deploy/rhai-operator --tail=3
# Output:
# "problem running manager"
# error: "no matches for kind \"Platform\" in version \"config.opendatahub.io/v1alpha1\""

# Try creating an LLMInferenceService
kubectl apply -f model.yaml
# Output:
# Error: failed calling webhook "connection-llmisvc.opendatahub.io":
# no endpoints available for service "rhai-operator-webhook-service"
```

**Impact:** Cannot create or modify any LLMInferenceService until the webhook is patched.

**Workaround:**
```bash
kubectl get mutatingwebhookconfiguration rhai-operator-mutating-webhook-configuration -o json | \
  jq '.webhooks[].failurePolicy = "Ignore"' | kubectl replace -f -
```

**Note:** This might be a version mismatch issue — the `opendatahub-operator:latest` image may have been updated to require the `Platform` CRD which isn't part of the xKS chart.

---

## E2E Test Results (with workarounds for #5 and #6 applied)

| Test | Method | Result |
|---|---|---|
| Unauthenticated request blocked | `curl` to gateway without auth header | HTTP 401, `www-authenticate: Bearer realm="azure-ad-jwt"`, `x-ext-auth-reason: credential not found` |
| Authenticated request reaches model | `curl` to gateway with Azure AD JWT | HTTP 200, chat completion response: "Hello! How can I" |
| MaaS API /v1/models | `curl` to gateway with Azure AD JWT | HTTP 200, `{"data":[],"object":"list"}` |
| MaaS subscription flow | Create MaaSModelRef + MaaSSubscription, POST /v1/api-keys | HTTP 201, API key issued: `sk-oai-...` |
| Model call with API key | `curl` to gateway with JWT + x-maas-api-key | HTTP 200, chat completion: "The answer to 2+2 is 4" |
| Rate limiting | Patch limit to 2/min, send 5 requests | Requests 1-2: 200, Requests 3-5: 429 |
| Model without command override | Deploy LLMInferenceService with no explicit command/args | Template merged correctly, vLLM started, pod Running |
| Direct model inference | `curl` to model service directly (bypass gateway) | HTTP 200, chat completion |

---

## How to Reproduce on a Clean Cluster

1. Rebase branch `feat/rhcl-maas-integration` to latest `upstream/main`
2. Deploy chart:
   ```bash
   helm template kserve-rhaii-xks ./charts/rhai-on-xks-chart \
     --set enabled=true --set azure.enabled=true \
     --set rhcl.enabled=true --set maas.enabled=true \
     --set maas.azureAD.enabled=true \
     --set maas.azureAD.tenantId=<TENANT_ID> \
     --set maas.azureAD.clientId=<CLIENT_ID> \
     --set-json 'imagePullSecret={"dockerConfigJson":"{\"auths\":{}}"}' \
     -n opendatahub | kubectl apply --server-side --force-conflicts -f -
   ```
3. Wait 2-3 minutes for all pods to start
4. Patch webhook: `kubectl get mutatingwebhookconfiguration rhai-operator-mutating-webhook-configuration -o json | jq '.webhooks[].failurePolicy = "Ignore"' | kubectl replace -f -`
5. Deploy model (see E2E Deployment Guide)
6. Patch DestinationRule (see Issue #5 workaround)
7. Run E2E tests (see E2E Deployment Guide)
