# MaaS xKS Support — Remaining Effort Estimate

## 1. Cloud Controller Manager Integration for RHCL

**Owner:** dbianchi's team (AICP)  
**Effort from us:** None — dbianchi offered to handle this  
**What's involved:** Add RHCL to KubernetesEngine CRD, package RHCL chart in cloud manager image, add RBAC

## 2. MaaS xKS Support in opendatahub-operator

| Phase | Estimate | What's involved |
|---|---|---|
| Development | 2-4 weeks | New codebase — learn operator build/test pipeline, add xKS kustomize overlay (`maas/overlays/xks/`), resolve PodMonitor CRD dependency, build custom operator image |
| Cross-team review | 1-2 weeks | dbianchi's team reviews and iterates on the PR in opendatahub-operator repo |
| Image build + release pipeline | 1 week | New operator image needs to go through Konflux/Brew build pipeline to produce a usable image |
| E2E testing on xKS | 1 week | Deploy new image on AKS, validate MaaS deploys correctly, re-run full E2E (auth, subscription, rate limiting) |
| **Total** | **5-8 weeks** | |

**Note:** Could be shorter (3-4 weeks) with pairing support from dbianchi's team since they know the operator codebase.

## Alternative: Helm-based MaaS (per RHAISTRAT-1377)

If the decision is to go Helm-based (as the Jira originally scoped):

| Phase | Estimate | What's involved |
|---|---|---|
| Restore MaaS templates | 0 days | Already built — restore 27 files from git history |
| E2E testing | 0 days | Already tested and documented |
| **Total** | **Ready to merge immediately** | |

## Context

- RHAISTRAT-1377 scoped MaaS xKS as "Helm-based installation, operator integration excluded from Dev Preview"
- MaaS Helm charts were built, E2E tested (auth 401/200, subscription flow 200, rate limiting 429), and then removed per PR #109 reviewer feedback directing operator-based approach
- The operator's MaaS module activates on xKS but doesn't deploy resources — manifests only have OpenShift overlays, no xKS rendering
- RHCL dependency chart is done and tested (PR #109)

## What's Already Delivered

- RHCL dependency chart (`charts/dependencies/rhcl-operator/`) — tested on AKS, 4/4 operators running, AuthPolicy affecting routes
- MaaS Helm-based deployment — E2E tested on AKS with Azure AD JWT auth, productized images
- MaaS operator investigation — identified missing xKS overlay, tested CRD/reconciler path
- Full KT documentation with test evidence
- RHOAIENG-67048 filed for Platform CRD issue
