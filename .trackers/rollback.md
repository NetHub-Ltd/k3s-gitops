# Rollback

**Branch:** `fix/flux-apply-image-automation`

Revert the PR or remove `clusters/k3s/kustomization.yaml` and restore prior README troubleshooting table.

If ImageRepository/ImagePolicy were created by this change, deleting them (or reverting the commit and reconciling with prune) returns to previous apply behavior.

No application schema or data changes.
