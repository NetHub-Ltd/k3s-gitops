# Task: Migrate keycloak into k3s-gitops

- **Status:** Implementation on feat/migrate-keycloak
- **Done:** apps/keycloak/*, SOPS secret, apps/kustomization.yaml, MIGRATION.md
- **Remaining:** PR merge, rotate secrets, Flux verify, mark nethub-cluster source
# Task: Migrate nethub-api into k3s-gitops

- **Goal:** First service transfer (nethub-api) per approved proposal
- **Status:** Implementation complete on branch feat/migrate-nethub-api
- **Done:**
  - apps/nethub-api/* (namespace, deployment, service, ingress, secret.enc.yaml, kustomization)
  - Ingress mirrors tawala (Traefik only, no cert-manager)
  - DB/env from nethub-cluster values/fastapi.yaml
  - SOPS-encrypted secrets
  - apps/kustomization.yaml lists nethub-api
  - MIGRATION.md marks nethub-api done; keycloak pending
- **Remaining:** PR to main; user rotates SOPS secrets to real values; verify Flux; then Keycloak PR
- **Out of scope this PR:** keycloak, redis, cert-manager annotations, image automation (Docker Hub image)
