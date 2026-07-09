# MaaS on xKS — End-to-End Deployment Steps

Step-by-step guide to deploy MaaS with RHCL on vanilla Kubernetes (AKS/EKS/GKE) and complete the full user journey.

## Prerequisites

- Kubernetes cluster with GPU nodes (NVIDIA T4 or higher)
- `kubectl`, `helm` CLI tools
- Red Hat registry pull secret (`auth.json` from `registry.redhat.io`)
- cert-manager and Istio already installed (via dependency charts below)

## Phase 1: Install Dependencies

```bash
cd odh-gitops

# 1. cert-manager
helm install cert-manager-operator charts/dependencies/cert-manager-operator

# 2. Gateway API CRDs + Istio
helm install gateway-api charts/dependencies/gateway-api
helm install sail-operator charts/dependencies/sail-operator

# 3. LWS operator
helm install lws-operator charts/dependencies/lws-operator

# 4. RHCL (Kuadrant, Authorino, Limitador)
helm install rhcl-operator charts/dependencies/rhcl-operator

# 5. PostgreSQL for MaaS API
helm install postgresql charts/dependencies/postgresql
```

Verify all operators are running:
```bash
kubectl get pods -n cert-manager-operator
kubectl get pods -n istio-system
kubectl get pods -n kuadrant-operators    # 4 operators, 0 restarts
kubectl get pods -n kuadrant-system       # authorino + limitador
```

## Phase 2: Deploy RHAI with MaaS Enabled

```bash
helm template kserve-rhaii-xks charts/rhai-on-xks-chart \
  --set azure.enabled=true \
  --set components.maas.enabled=true \
  --set-file imagePullSecret.dockerConfigJson=path/to/auth.json \
  | kubectl apply -f -
```

This automatically:
- Deploys the RHAI operator with MaaS enabled
- Creates Kserve and ModelsAsService CRs (post-install hook)
- Creates inference gateway with TLS (post-install hook)
- Creates MaaS gateway with TLS, CA bundles, and Authorino CA trust (post-install hook)
- Creates webhook and API serving certificates

Verify:
```bash
# Operator
kubectl get pods -n redhat-ods-operator
# Expected: rhai-operator 1/1 Running

# Kserve
kubectl get kserve
# Expected: default-kserve Ready

# MaaS
kubectl get modelsasservices
# Expected: default-modelsasservice Ready

# MaaS controller + API
kubectl get pods -n redhat-ods-applications
# Expected: maas-controller 1/1, maas-api 1/1, llmisvc-controller 1/1

# Tenant
kubectl get tenants -A
# Expected: default-tenant Ready Reconciled

# Gateways
kubectl get gateways -A
# Expected: inference-gateway (Programmed), maas-default-gateway (Programmed)
```

## Phase 3: Deploy a Model

Create a namespace and deploy a model. Use `llm` as the namespace so the MaaS gateway path prefix is `/llm/<model-name>` (required by MaaS AuthPolicy).

```yaml
# llm-model.yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: qwen2-0-5b
  namespace: llm
spec:
  model:
    name: qwen2-0-5b
    uri: hf://Qwen/Qwen2.5-0.5B-Instruct
  replicas: 1
  router:
    gateway:
      refs:
      - name: maas-default-gateway
        namespace: openshift-ingress
    route: {}
  template:
    containers:
    - name: main
      resources:
        limits:
          cpu: "4"
          memory: 16Gi
          nvidia.com/gpu: "1"
        requests:
          cpu: "2"
          memory: 8Gi
          nvidia.com/gpu: "1"
    tolerations:
    - effect: NoSchedule
      key: nvidia.com/gpu
      operator: Exists
```

```bash
kubectl create namespace llm
kubectl apply -f llm-model.yaml
```

Wait for the model to be ready:
```bash
kubectl get llminferenceservices -A
# Expected: qwen2-0-5b Ready True, URL: http://<GATEWAY_IP>/llm/qwen2-0-5b
```

## Phase 4: Configure MaaS Access

### 4a. Create MaaSModelRef

Exposes the model through MaaS. Must be in the same namespace as the LLMInferenceService.

```yaml
# maas-modelref.yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata:
  name: qwen2-0-5b
  namespace: llm
spec:
  modelRef:
    kind: LLMInferenceService
    name: qwen2-0-5b
```

```bash
kubectl apply -f maas-modelref.yaml
kubectl get maasmodelrefs -A
# Expected: qwen2-0-5b Ready, HTTPROUTE: qwen2-0-5b-kserve-route, GATEWAY: maas-default-gateway
```

### 4b. Create MaaSAuthPolicy

Defines who can access the model. Goes in `models-as-a-service` namespace.

```yaml
# maas-authpolicy.yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata:
  name: my-auth-policy
  namespace: models-as-a-service
spec:
  subjects:
    users:
      - my-user
  modelRefs:
    - name: qwen2-0-5b
      namespace: llm
```

```bash
kubectl apply -f maas-authpolicy.yaml
kubectl get maasauthpolicies -A
# Expected: my-auth-policy Active
```

### 4c. Create MaaSSubscription

Defines rate limits and model access for a user. Goes in `models-as-a-service` namespace.

```yaml
# maas-subscription.yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: my-subscription
  namespace: models-as-a-service
spec:
  owner:
    users:
      - my-user
  modelRefs:
    - name: qwen2-0-5b
      namespace: llm
      tokenRateLimits:
        - limit: 10000
          window: "1m"
```

```bash
kubectl apply -f maas-subscription.yaml
kubectl get maassubscriptions -A
# Expected: my-subscription Active
```

## Phase 5: User Journey — API Key + Inference

### 5a. Create an API Key

```bash
MAAS_API=https://maas-api.redhat-ods-applications.svc:8443

curl -sk $MAAS_API/v1/api-keys \
  -H "x-maas-username: my-user" \
  -H 'x-maas-group: ["system:authenticated"]' \
  -H "x-maas-tenant: models-as-a-service" \
  -X POST -H "Content-Type: application/json" \
  -d '{"name":"my-key","subscription":"my-subscription","expiresIn":"24h"}'
```

Response:
```json
{
  "key": "sk-oai-abc123...",
  "name": "my-key",
  "subscription": "my-subscription",
  "expiresAt": "2026-06-20T..."
}
```

Save the `key` value — this is your API key.

### 5b. Test Unauthenticated Request (should fail)

```bash
MAAS_GW=http://<MAAS_GATEWAY_IP>

curl -s $MAAS_GW/llm/qwen2-0-5b/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2-0-5b","messages":[{"role":"user","content":"Hi"}],"max_tokens":5}'
```

Expected: **401 Unauthorized** (RHCL enforces authentication)

### 5c. Inference with API Key

```bash
curl -s $MAAS_GW/llm/qwen2-0-5b/v1/chat/completions \
  -H "Authorization: Bearer sk-oai-abc123..." \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2-0-5b","messages":[{"role":"user","content":"What is Kubernetes?"}],"max_tokens":50}'
```

Expected: Model response with the answer.

The request flow:
```
User (API key) → MaaS Gateway → RHCL/Authorino (validates key, checks subscription)
  → RHCL/Limitador (enforces rate limits) → Model (vLLM) → Response
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| MaaS controller CrashLoopBackOff | Webhook TLS cert missing | Chart hook creates it; check `kubectl get certificate -n redhat-ods-applications` |
| Kuadrant operator crash-loops | Missing RBAC for `monitoring.coreos.com` | RBAC fix included in rhcl-operator chart |
| MaaS API returns AUTH_FAILURE | Authorino can't verify MaaS API TLS cert | Chart hook configures Authorino CA trust; check `kubectl get configmap authorino-ca-bundle -n kuadrant-system` |
| Model inference returns TLS error | CA bundle missing opendatahub-ca | Chart hook includes it; verify with `kubectl get configmap rhai-ca-bundle -n openshift-ingress` |
| MaaSSubscription stuck in Degraded | TokenRateLimitPolicy not accepted | Check Kuadrant operator logs; should be 0 restarts after RBAC fix |
| ModelsAsService CR not Ready | Operator not finding xKS overlay | Ensure operator image has MaaS xKS overlay baked in (PR #1015 must be merged first) |
