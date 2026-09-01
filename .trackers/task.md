# Task Tracker

**Branch:** `fix/flux-apply-image-automation`  
**Base:** `main`

## Goal
Apply Flux image automation objects so the cluster can select newer semver tags (e.g. 0.0.40) without manual image pins.

## Approved scope
- Add `clusters/k3s/kustomization.yaml` listing flux-system, infrastructure, apps, image-tawala-api, image-automation
- README troubleshooting row for ImageRepository/ImagePolicy NotFound
- Do **not** manually change deployment image tag to 0.0.40

## Done
- [x] Add clusters/k3s/kustomization.yaml
- [x] README troubleshooting note
- [x] Trackers

## Remaining
- [ ] PR review / merge
- [ ] Cluster reconcile verification (user)

## Out of scope
- Manual image tag bump
- TawalaKE CI changes
- JWT / SECRET_KEY
