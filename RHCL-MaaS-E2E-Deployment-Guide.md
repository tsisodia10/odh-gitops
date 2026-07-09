# RHCL & MaaS — E2E Deployment & Testing Guide

**Chart:** rhai-on-xks-chart v3.5.0-ea.1  
**Tested on:** AKS cluster (Standard_NC4as_T4_v3, Tesla T4 GPU)  
**MaaS Images:** registry.redhat.io/rhoai/odh-maas-api-rhel9:v3.4.0, registry.redhat.io/rhoai/odh-maas-controller-rhel9:v3.4.0

---

## 1. Prerequisites

- AKS cluster with GPU node pool (or any Kubernetes cluster with GPU)
- `kubectl` access to the cluster
- Helm v3.17+
- Pull secret for `registry.redhat.io` (needs access to `rhoai/`, `rhcl-1/`, `rhaiis/` namespaces)
- Azure AD app registration (for JWT auth testing) with client ID, client secret, and tenant ID

---

## 2. Deploy the Chart

```bash
helm install kserve-rhaii-xks ./charts/rhai-on-xks-chart -n opendatahub \
  --set azure.enabled=true \
  --set rhcl.enabled=true \
  --set maas.enabled=true \
  --set maas.azureAD.enabled=true \
  --set maas.azureAD.tenantId=<YOUR_TENANT_ID> \
  --set maas.azureAD.clientId=<YOUR_CLIENT_ID> \
  --set imagePullSecret.dockerConfigJson='<YOUR_PULL_SECRET>'
```

### Verify deployment

```bash
# RHCL operators
kubectl get pods -n kuadrant-operators
# Expected: authorino-operator, kuadrant-operator, limitador-operator — all Running

# RHCL runtime
kubectl get pods -n kuadrant-system
# Expected: authorino, limitador — all Running

# MaaS
kubectl get pods -n models-as-a-service
# Expected: maas-api, maas-controller, postgres — all Running

# Gateway
kubectl get gateway -n istio-system
# Expected: maas-default-gateway with external IP, PROGRAMMED=True

# Policies
kubectl get authpolicy,ratelimitpolicy -A
# Expected: all Accepted + Enforced
```

---

## 3. Apply KServe Workarounds

These are required due to KServe operator issues (not RHCL/MaaS chart issues). Without these, model deployment will fail.

### 3a. Create rhai-ca secret

```bash
kubectl get secret opendatahub-ca -n cert-manager -o json | \
  jq 'del(.metadata.resourceVersion,.metadata.uid,.metadata.creationTimestamp,.metadata.managedFields,.metadata.ownerReferences) | .metadata.name="rhai-ca"' | \
  kubectl apply -f -
```

### 3b. Patch uidModelcar (only needed for OCI modelcar models)

```bash
# Scale down operator to prevent it from reverting the change
kubectl scale deploy rhai-operator -n redhat-ods-operator --replicas=0

# Patch the webhook so LLMInferenceService can be created while operator is down
kubectl get mutatingwebhookconfiguration rhai-operator-mutating-webhook-configuration -o json | \
  jq '.webhooks[].failurePolicy = "Ignore"' | kubectl replace -f -

# Patch the ConfigMap
kubectl patch configmap inferenceservice-config -n redhat-ods-applications --type=merge \
  -p '{"data":{"storageInitializer":"{\"cpuLimit\":\"1\",\"cpuModelcar\":\"10m\",\"cpuRequest\":\"100m\",\"enableModelcar\":true,\"image\":\"quay.io/opendatahub/kserve-storage-initializer:odh-stable\",\"memoryLimit\":\"24Gi\",\"memoryModelcar\":\"15Mi\",\"memoryRequest\":\"100Mi\",\"uidModelcar\":1001}"}}'
```

### 3c. Create missing LLMInferenceServiceConfigs (if not present)

```bash
# Check if router-route and scheduler configs exist
kubectl get llminferenceserviceconfig -n redhat-ods-applications

# If v3-5-0-ea-1-kserve-config-llm-router-route or scheduler are missing,
# they need to be created from the opendatahub namespace copies.
# This may require temporarily relaxing CRD validation — see KT doc for details.
```

---

## 4. Deploy a Test Model

### Option A: HuggingFace model (no modelcar, works on T4)

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: qwen2-0-5b
  namespace: model-serving
  annotations:
    alpha.maas.opendatahub.io/tiers: '["free","standard","enterprise"]'
    security.opendatahub.io/enable-auth: "true"
  labels:
    opendatahub.io/dashboard: "true"
spec:
  model:
    name: qwen2-0-5b
    uri: "hf://Qwen/Qwen2.5-0.5B-Instruct"
  replicas: 1
  router:
    gateway:
      refs:
        - name: maas-default-gateway
          namespace: istio-system
    route: {}
  template:
    containers:
      - name: main
        command: ["/bin/bash", "-c"]
        args:
          - |
            exec vllm serve /mnt/models \
              --served-model-name qwen2-0-5b \
              --port 8000 \
              --max-model-len 4096 \
              --dtype float16 \
              --disable-uvicorn-access-log \
              --enable-ssl-refresh \
              --ssl-certfile /var/run/kserve/tls/tls.crt \
              --ssl-keyfile /var/run/kserve/tls/tls.key
        resources:
          limits:
            nvidia.com/gpu: "1"
            cpu: "4"
            memory: 16Gi
          requests:
            nvidia.com/gpu: "1"
            cpu: "2"
            memory: 8Gi
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

**Note:** Explicit `command` and `args` are required due to KServe template merge issue. Do not rely on the default template.

### Wait for model to be ready

```bash
# Takes 5-10 minutes (model download + vLLM startup)
kubectl get llminferenceservice -n model-serving -w
# Wait until READY=True
```

### 4a. Fix DestinationRule after model deploys

```bash
# Scale down controllers to prevent reconciliation
kubectl scale deployment kserve-controller-manager -n opendatahub --replicas=0
kubectl scale deployment llmisvc-controller-manager -n redhat-ods-applications --replicas=0

# Patch DestinationRule
kubectl patch destinationrule qwen2-0-5b-kserve-workload-svc -n model-serving --type=merge \
  -p '{"spec":{"trafficPolicy":{"tls":{"insecureSkipVerify":true}}}}'
kubectl patch destinationrule qwen2-0-5b-kserve-workload-svc -n model-serving --type=json \
  -p '[{"op":"remove","path":"/spec/trafficPolicy/tls/caCertificates"},{"op":"remove","path":"/spec/trafficPolicy/tls/sni"}]'

# Restart gateway to pick up changes
kubectl rollout restart deployment maas-default-gateway-istio -n istio-system

# Scale controllers back up
kubectl scale deployment kserve-controller-manager -n opendatahub --replicas=1
kubectl scale deployment llmisvc-controller-manager -n redhat-ods-applications --replicas=1
kubectl scale deploy rhai-operator -n redhat-ods-operator --replicas=1
```

---

## 5. E2E Tests

### 5a. Get Azure AD Token

```bash
TOKEN=$(curl -s -X POST \
  "https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token" \
  -d "client_id=<CLIENT_ID>" \
  -d "client_secret=<CLIENT_SECRET>" \
  -d "scope=<CLIENT_ID>/.default" \
  -d "grant_type=client_credentials" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

### 5b. Test: Unauthenticated request blocked (expect 401)

```bash
kubectl run test-noauth --rm -i --restart=Never --image=curlimages/curl -- \
  curl -sv --max-time 10 \
  "http://maas-default-gateway-istio.istio-system.svc/model-serving/qwen2-0-5b/v1/chat/completions" \
  -H "Host: inference.maas.local" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2-0-5b","messages":[{"role":"user","content":"Hi"}],"max_tokens":5}'

# Expected: HTTP 401
# Expected header: www-authenticate: Bearer realm="azure-ad-jwt"
# Expected header: x-ext-auth-reason: credential not found
```

### 5c. Test: Authenticated request reaches model (expect 200)

```bash
# Save token to configmap for use inside the cluster
kubectl create configmap test-token --from-literal=token="$TOKEN"

kubectl run test-auth --rm -i --restart=Never --image=curlimages/curl \
  --overrides='{"spec":{"containers":[{"name":"test-auth","image":"curlimages/curl","command":["sh","-c","TOKEN=$(cat /t/token); curl -s -w \"\\nHTTP: %{http_code}\" --max-time 20 http://maas-default-gateway-istio.istio-system.svc/model-serving/qwen2-0-5b/v1/chat/completions -H \"Host: inference.maas.local\" -H \"Authorization: Bearer $TOKEN\" -H \"Content-Type: application/json\" -d \"{\\\"model\\\":\\\"qwen2-0-5b\\\",\\\"messages\\\":[{\\\"role\\\":\\\"user\\\",\\\"content\\\":\\\"Hi\\\"}],\\\"max_tokens\\\":5}\""],"volumeMounts":[{"name":"t","mountPath":"/t"}]}],"volumes":[{"name":"t","configMap":{"name":"test-token"}}]}}'

kubectl delete configmap test-token

# Expected: HTTP 200 with chat completion response
```

### 5d. Test: MaaS API /v1/models (expect 200)

```bash
kubectl create configmap test-token --from-literal=token="$TOKEN"

kubectl run test-maas --rm -i --restart=Never --image=curlimages/curl \
  --overrides='{"spec":{"containers":[{"name":"test-maas","image":"curlimages/curl","command":["sh","-c","TOKEN=$(cat /t/token); curl -s -w \"\\nHTTP: %{http_code}\" --max-time 10 http://maas-default-gateway-istio.istio-system.svc/v1/models -H \"Host: inference.maas.local\" -H \"Authorization: Bearer $TOKEN\""],"volumeMounts":[{"name":"t","mountPath":"/t"}]}],"volumes":[{"name":"t","configMap":{"name":"test-token"}}]}}'

kubectl delete configmap test-token

# Expected: HTTP 200 with {"data":[],"object":"list"}
```

### 5e. Test: MaaS Subscription Flow

```bash
# 1. Create MaaSModelRef
cat <<'EOF' | kubectl apply -f -
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata:
  name: qwen2-0-5b
  namespace: models-as-a-service
spec:
  modelRef:
    kind: LLMInferenceService
    name: qwen2-0-5b
EOF

# 2. Create MaaSSubscription
cat <<'EOF' | kubectl apply -f -
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: test-subscription
  namespace: models-as-a-service
spec:
  owner:
    users:
      - "<YOUR_AZURE_AD_CLIENT_ID>"
    groups:
      - name: "system:authenticated"
  modelRefs:
    - name: qwen2-0-5b
      namespace: models-as-a-service
      tokenRateLimits:
        - limit: 10000
          window: "1h"
      billingRate:
        perToken: "0.001"
EOF

# 3. Generate API key (direct to MaaS API)
kubectl run gen-key --rm -i --restart=Never --image=curlimages/curl -- \
  curl -sk --max-time 10 \
  "https://maas-api.models-as-a-service.svc:8443/v1/api-keys" \
  -X POST -H "Content-Type: application/json" \
  -H 'x-maas-username: <YOUR_AZURE_AD_CLIENT_ID>' \
  -H 'x-maas-group: ["system:authenticated"]' \
  -d '{"name":"test-key"}'

# Expected: HTTP 201 with {"key":"sk-oai-...","name":"test-key",...}
# Save the "key" value for the next test

# 4. Call model with Azure AD token + API key
kubectl create configmap test-creds --from-literal=token="$TOKEN" --from-literal=apikey="<API_KEY_FROM_STEP_3>"

kubectl run test-full --rm -i --restart=Never --image=curlimages/curl \
  --overrides='{"spec":{"containers":[{"name":"test-full","image":"curlimages/curl","command":["sh","-c","TOKEN=$(cat /c/token); APIKEY=$(cat /c/apikey); curl -s -w \"\\nHTTP: %{http_code}\" --max-time 20 http://maas-default-gateway-istio.istio-system.svc/model-serving/qwen2-0-5b/v1/chat/completions -H \"Host: inference.maas.local\" -H \"Authorization: Bearer $TOKEN\" -H \"x-maas-api-key: $APIKEY\" -H \"Content-Type: application/json\" -d \"{\\\"model\\\":\\\"qwen2-0-5b\\\",\\\"messages\\\":[{\\\"role\\\":\\\"user\\\",\\\"content\\\":\\\"What is 2+2?\\\"}],\\\"max_tokens\\\":10}\""],"volumeMounts":[{"name":"c","mountPath":"/c"}]}],"volumes":[{"name":"c","configMap":{"name":"test-creds"}}]}}'

kubectl delete configmap test-creds

# Expected: HTTP 200 with chat completion
```

### 5f. Test: Rate Limiting (expect 429)

```bash
# Temporarily lower rate limit to 2/min
kubectl patch ratelimitpolicy gateway-request-rate-limit -n istio-system --type=json \
  -p '[{"op":"replace","path":"/spec/defaults/limits/per-api-key/rates/0/limit","value":2}]'

sleep 10

# Send 5 rapid requests with API key
kubectl create configmap test-creds --from-literal=token="$TOKEN"

kubectl run test-rl --rm -i --restart=Never --image=curlimages/curl \
  --overrides='{"spec":{"containers":[{"name":"test-rl","image":"curlimages/curl","command":["sh","-c","TOKEN=$(cat /c/token); for i in 1 2 3 4 5; do CODE=$(curl -s -o /dev/null -w \"%{http_code}\" --max-time 10 http://maas-default-gateway-istio.istio-system.svc/model-serving/qwen2-0-5b/v1/chat/completions -H \"Host: inference.maas.local\" -H \"Authorization: Bearer $TOKEN\" -H \"x-maas-api-key: <API_KEY>\" -H \"Content-Type: application/json\" -d \"{\\\"model\\\":\\\"qwen2-0-5b\\\",\\\"messages\\\":[{\\\"role\\\":\\\"user\\\",\\\"content\\\":\\\"x\\\"}],\\\"max_tokens\\\":1}\"); echo \"Request $i: HTTP $CODE\"; done"],"volumeMounts":[{"name":"c","mountPath":"/c"}]}],"volumes":[{"name":"c","configMap":{"name":"test-creds"}}]}}'

kubectl delete configmap test-creds

# Expected: First 2 return 200, remaining return 429

# Restore rate limit
kubectl patch ratelimitpolicy gateway-request-rate-limit -n istio-system --type=json \
  -p '[{"op":"replace","path":"/spec/defaults/limits/per-api-key/rates/0/limit","value":1000}]'
```

---

## 6. Expected Test Results

| Test | Expected Result |
|---|---|
| Unauthenticated request | HTTP 401, `www-authenticate: Bearer realm="azure-ad-jwt"` |
| Authenticated request | HTTP 200 with chat completion |
| MaaS API /v1/models | HTTP 200 with `{"data":[],"object":"list"}` |
| MaaS API key generation | HTTP 201 with `sk-oai-...` key |
| Model call with API key | HTTP 200 with chat completion |
| Rate limiting | First N return 200, rest return 429 |

---

## 7. Images Reference

| Component | Image |
|---|---|
| MaaS API | `registry.redhat.io/rhoai/odh-maas-api-rhel9:v3.4.0` |
| MaaS Controller | `registry.redhat.io/rhoai/odh-maas-controller-rhel9:v3.4.0` |
| Authorino Operator | `registry.redhat.io/rhcl-1/authorino-rhel9-operator` |
| Kuadrant Operator | `registry.redhat.io/rhcl-1/rhcl-rhel9-operator` |
| Limitador Operator | `registry.redhat.io/rhcl-1/limitador-rhel9-operator` |
| Authorino Runtime | `registry.redhat.io/rhcl-1/authorino-rhel9` |
| Limitador Runtime | `registry.redhat.io/rhcl-1/limitador-rhel9` |
| vLLM | `registry.redhat.io/rhaiis/vllm-cuda-rhel9` |
| PostgreSQL | `registry.redhat.io/rhel9/postgresql-15` |

All images are productized from `registry.redhat.io`.
