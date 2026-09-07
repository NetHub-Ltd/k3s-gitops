# Rollback
Remove `- keycloak` from apps/kustomization.yaml; Flux prune. Or revert PR.

- Remove `- nethub-api` from apps/kustomization.yaml and let Flux prune (prune: true on apps Kustomization).
- Or delete apps/nethub-api and revert MIGRATION.md row.
- Previous known-good: main tip before this branch.
