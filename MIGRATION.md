# Service migration: nethub-cluster → k3s-gitops

Track each service as it moves. **Do not apply migrated services from nethub-cluster.**

| Service | Source (nethub-cluster) | Target (k3s-gitops) | Status | Notes |
|---------|-------------------------|---------------------|--------|-------|
| nethub-api | `values/fastapi.yaml` | `apps/nethub-api/` | done | GHCR + image automation |
| keycloak | `nethub_stack/services/keycloak.yaml` | `apps/keycloak/` | done | |
| **redis-shared** | `nethub_stack/services/redis.yaml` | `apps/redis-shared/` | **done** | **Adopt in place** — same STS/Service/PVC names. SOPS `redis-creds`. No extra Namespace. |
| mazeltov | `k3s/mazeltov/` | TBD | pending | Separate namespace |
| cloudflared / middlewares | `nethub_stack/services/*` | TBD | pending | |
| databases (CNPG) | `nethub_stack/database/*` | TBD | pending | |

## Redis adopt notes

- DNS unchanged: `redis-shared.nethub.svc.cluster.local:6379`
- PVC claim template name must stay `redis-data` (volume `redis-data-redis-shared-0`)
- First apply should match live spec to avoid unnecessary pod restart
- Password is SOPS-encrypted; rotate later with a coordinated client update

## Kustomize note (shared namespace)

Only **one** `Namespace/nethub` under `apps/` (`apps/tawala-api/namespace.yaml`).

## Keycloak image

- Tracked in `apps/keycloak/deployment.yaml`
- Bumped to `quay.io/keycloak/keycloak:26.7.3` (from 26.0)
- Realm/clients/themes portability: prefer exported realm JSON + optional dedicated config repo (see PR notes)
## Keycloak custom image

- Source repo: `NetHub-Ltd/keycloak-config`
- Image: `ghcr.io/nethub-ltd/keycloak` (themes + realm templates, base 26.7.3)
- ImageRepository/Policy: `clusters/k3s/image-keycloak.yaml`
- Deployment marker: `# {"$imagepolicy": "flux-system:keycloak"}`
