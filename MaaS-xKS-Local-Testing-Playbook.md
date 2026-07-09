# MaaS xKS — Local Testing Playbook

How we tested the full operator-managed MaaS pipeline on AKS before merging any PRs.

## The Problem

The implementation spans 3 repos with a build dependency chain. We can't test by merging PRs one at a time — the operator image needs all changes baked in. So we built a custom operator image locally.

## PRs Under Test

| Repo | PR | Branch | What |
|------|----|--------|------|
| `models-as-a-service` | [#1015](https://github.com/opendatahub-io/models-as-a-service/pull/1015) | `feat/xks-overlay` | xKS kustomize overlay + cert-manager webhook TLS |
| `opendatahub-operator` | [#3670](https://github.com/opendatahub-io/opendatahub-operator/pull/3670) | `feat/maas-xks-overlay` | Go code to select `overlays/xks` when `ODH_PLATFORM_TYPE=XKS` |
| `odh-gitops` | [#109](https://github.com/opendatahub-io/odh-gitops/pull/109) | `feat/rhcl-maas-integration` | RHCL dependency chart |

## Step 1: Point Operator Build to Fork

The operator's `get_all_manifests.sh` downloads MaaS manifests from upstream during image build. We temporarily pointed it to our fork's branch so the xKS overlay gets baked in.

```bash
cd opendatahub-operator

# Change line 32 in get_all_manifests.sh:
# FROM:
#   ["maas"]="opendatahub-io:models-as-a-service:stable@b409c456...:deployment"
# TO:
#   ["maas"]="tsisodia10:models-as-a-service:feat/xks-overlay@<commit-sha>:deployment"
```

The Go code change (PR #3670) was already on the `feat/maas-xks-overlay` branch.

## Step 2: Build the Operator Image

```bash
cd opendatahub-operator

# Build with CGO_ENABLED=0 (required for cross-compilation from ARM Mac to linux/amd64)
make image-build \
  IMG=quay.io/tsisodia10/opendatahub-operator:maas-xks-test \
  IMAGE_BUILDER=podman \
  CGO_ENABLED=0
```

This builds a 3-stage container:
1. **manifests stage** — runs `get_all_manifests.sh`, downloads MaaS manifests from our fork (includes `overlays/xks/`)
2. **builder stage** — compiles Go binary with xKS overlay selection logic
3. **final stage** — combines binary + manifests into the operator image

Build took ~2 minutes.

## Step 3: Push to Registry

```bash
podman push quay.io/tsisodia10/opendatahub-operator:maas-xks-test
```

Make the quay.io repo **public** so AKS can pull it (Settings > Repository Visibility > Public).

## Step 4: Deploy on AKS

The cluster already had the Helm chart deployed with:
- `ODH_PLATFORM_TYPE=XKS`
- `RHAI_DISABLE_MODELSASSERVICE_COMPONENT=false`
- `ModelsAsService` CRD installed
- `ModelsAsService` CR created by post-install hook

Update **both** the init container and main container to use the custom image:

```bash
kubectl patch deployment rhai-operator -n redhat-ods-operator --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/initContainers/0/image",
   "value": "quay.io/tsisodia10/opendatahub-operator:maas-xks-test"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/image",
   "value": "quay.io/tsisodia10/opendatahub-operator:maas-xks-test"}
]'
```

> **Important:** You must update the init container (`copy-manifests`) too — it copies manifests from `/opt/manifests-template/` to `/opt/manifests/` at startup. If only the main container is updated, the init container copies old manifests without the xKS overlay.

## Step 5: Verify Operator Reconciliation

```bash
# Check operator logs for MaaS reconciliation
kubectl logs -n redhat-ods-operator -l name=rhai-operator --tail=30 | grep modelsas
```

Expected output — no errors, all actions execute:
```
Executing action: renderMaasOperatorInstall
Executing action: deploy.(*Action).run-fm
Executing action: ensureMaasClusterConfigControllerRef
Executing action: status/deployments.(*Action).run-fm
Executing action: gc.(*Action).run-fm
```

```bash
# Check ModelsAsService CR status
kubectl get modelsasservices -A
```

Expected:
```
NAME                      READY   REASON
default-modelsasservice   True
```

## Step 6: Verify MaaS Controller

```bash
kubectl get pods -n redhat-ods-applications | grep maas
```

Expected:
```
maas-controller-99c98fc8-rdd64   1/1   Running   0   39s
```

```bash
# Check controller logs
kubectl logs -n redhat-ods-applications -l control-plane=maas-controller --tail=30
```

Expected — all reconcilers start:
```
Starting Controller: external-model-reconciler — Starting workers
Starting Controller: maasmodelref — Starting workers
Starting Controller: maassubscription — Starting workers
Starting Controller: maasauthpolicy — Starting workers
Starting Controller: deployment — Starting workers
Starting Controller: tenant — Starting workers
```

## Step 7: Verify MaaS Resources

```bash
# Check model refs
kubectl get maasmodelrefs -A
```
```
NAMESPACE             NAME         PHASE     AGE
models-as-a-service   qwen2-0-5b   Pending   13d
```

```bash
# Check subscriptions
kubectl get maassubscriptions -A
```
```
NAMESPACE             NAME                    PHASE    AGE
models-as-a-service   e2e-test-subscription   Failed   13d
```

The subscription shows `Failed` because the MaaS gateway doesn't exist yet (the controller logs `HTTPRoute not found for model, skipping TokenRateLimitPolicy creation`). This is expected — the gateway is a separate follow-up item.

## Step 8: Verify Cert-Manager Webhook TLS

```bash
kubectl get certificate -n redhat-ods-applications
```
```
NAME                           READY   SECRET                         AGE
maas-controller-webhook-cert   True    maas-controller-webhook-cert   30m
```

This confirms the xKS overlay's cert-manager Certificate is working — it generates the TLS secret that gets volume-mounted into the controller at `/tmp/k8s-webhook-server/serving-certs`.

## Step 9: Verify Manifest Structure Inside Pod

```bash
kubectl exec -n redhat-ods-operator deployment/rhai-operator -- \
  ls /opt/manifests/maas/overlays/
```
```
odh
openshift
xks
```

Confirms the xKS overlay was successfully baked into the operator image and copied to runtime path.

## Issues Encountered

| Issue | Cause | Fix |
|-------|-------|-----|
| `gcc: error: unrecognized command-line option '-m64'` | Cross-compiling from ARM Mac with CGO_ENABLED=1 | Build with `CGO_ENABLED=0` |
| `401 UNAUTHORIZED` on image push | Quay.io repo created as private by default | Made repo public in quay.io settings |
| `ErrImagePull` on AKS | Same — AKS couldn't pull private image | Same fix |
| `bundle not found at overlays/xks` | Only main container was updated, init container still used old image | Patched both init container and main container |
| MaaS subscription `Failed` | MaaS gateway not deployed (follow-up item) | Expected — not in scope for controller validation |

## Cleanup

After testing, revert the `get_all_manifests.sh` change (it was only for local testing):

```bash
cd opendatahub-operator
git checkout -- get_all_manifests.sh
```

To restore the operator to the upstream image:

```bash
kubectl patch deployment rhai-operator -n redhat-ods-operator --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/initContainers/0/image",
   "value": "quay.io/opendatahub/opendatahub-operator:latest"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/image",
   "value": "quay.io/opendatahub/opendatahub-operator:latest"}
]'
```

## Result

All 3 PRs integrate correctly. The operator-managed MaaS controller deploys and runs on AKS with the xKS overlay providing cert-manager webhook TLS. `ModelsAsService` CR reaches `Ready: True`.
