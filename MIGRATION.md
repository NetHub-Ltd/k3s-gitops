# Service migration: nethub-cluster → k3s-gitops

Track each service as it moves. **Do not apply migrated services from nethub-cluster.**

| Service | Source (nethub-cluster) | Target (k3s-gitops) | Status | Notes |
|---------|-------------------------|---------------------|--------|-------|
| nethub-api | `k3s/nethub/webapp-helm-chart-v2/values/fastapi.yaml` | `apps/nethub-api/` | done / in PR | See PR #15 if not yet on main. Ingress `api.nethub.co.ke`. |
| **keycloak** | `nethub_stack/services/keycloak.yaml` | `apps/keycloak/` | **done** | Ingress `auth.nethub.co.ke` + `asfalis.nethub.co.ke`. DB URL: `jdbc:postgresql://nethub-db-cluster-rw.postgres.svc.cluster.local:5432/keycloak`. Image `quay.io/keycloak/keycloak:26.0`. Traefik only (no cert-manager). Traefik Middleware CRD not migrated (follow-up). Realm import not wired (existing DB data). |
| redis | `nethub_stack/services/redis.yaml` | TBD | pending | |
| cloudflared / middlewares | `nethub_stack/services/*` | TBD | pending | |
| databases (CNPG refs) | `nethub_stack/database/*` | TBD | pending | Shared dependency |

## Marker convention

- Update this table in the same PR that adds `apps/<service>/`.
- Label workloads with `nethub.co.ke/migrated-from: nethub-cluster`.
- In nethub-cluster, add `MIGRATED-*.md` pointing at k3s-gitops path.

## Cutover

After Flux shows the app healthy, stop applying the old nethub-cluster manifests for that service to avoid duplicate Deployment/Ingress.
