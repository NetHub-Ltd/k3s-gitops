# Task Tracker

**Branch:** `fix/flux-image-api-v1`  
**Base:** `main`

## Goal
Fix flux-system reconcile: ImagePolicy/ImageRepository/ImageUpdateAutomation must use API version served by the cluster CRDs (`v1`, not `v1beta2`).

## Approved scope
Continuation of image-automation apply work — unblock ReconciliationFailed.

## Done
- [x] clusters/k3s/kustomization.yaml (PR #9)
- [x] Bump image toolkit manifests to `image.toolkit.fluxcd.io/v1`

## Remaining
- [ ] PR merge + cluster reconcile
- [ ] Verify ImagePolicy selects 0.0.40 via automation (no manual pin)

## Out of scope
- Manual deployment image tag change
