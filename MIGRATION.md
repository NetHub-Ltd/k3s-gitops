# Service migration: nethub-cluster → k3s-gitops

Track each service as it moves. **Do not apply migrated services from nethub-cluster.**

| Service | Source (nethub-cluster) | Target (k3s-gitops) | Status | Notes |
|---------|-------------------------|---------------------|--------|-------|
| **nethub-api** | `k3s/nethub/webapp-helm-chart-v2/values/fastapi.yaml` + Helm chart | `apps/nethub-api/` | **done** | Ingress: `api.nethub.co.ke` (Traefik only, no cert-manager). DB host from values: `nethub-db-pooler.postgres.svc.cluster.local`. Image: `daviekaranja254/nethubke:v0.0.3` (placeholder secrets in SOPS — rotate before prod). |
| keycloak | `nethub_stack/services/keycloak.yaml` | `apps/keycloak/` | pending | Next wave |
| redis | `nethub_stack/services/redis.yaml` | TBD | pending | |
| cloudflared / middlewares | `nethub_stack/services/*` | TBD | pending | |
| databases (CNPG refs) | `nethub_stack/database/*` | TBD | pending | Shared dependency — do not recreate lightly |

## Marker convention

- This table is updated in the **same PR** that adds an app under `apps/`.
- Workloads get label `nethub.co.ke/migrated-from: nethub-cluster` where applicable.
- In **nethub-cluster**, add a short `MIGRATED.md` note or header comment pointing at k3s-gitops path.

## Cutover note

After Flux reconciles a service healthy, scale down or stop applying the old nethub-cluster definition for that service to avoid double Ingress / double Deployment.
