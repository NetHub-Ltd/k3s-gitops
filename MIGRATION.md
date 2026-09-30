# Service migration: nethub-cluster → k3s-gitops

Track each service as it moves. **Do not apply migrated services from nethub-cluster.**

| Service | Source (nethub-cluster) | Target (k3s-gitops) | Status | Notes |
|---------|-------------------------|---------------------|--------|-------|
| nethub-api | `values/fastapi.yaml` | `apps/nethub-api/` | done | GHCR + image automation |
| **keycloak** | `nethub_stack/services/keycloak.yaml` | removed | **removed** | Hard-cut replaced by **Zitadel** on `auth.nethub.co.ke` |
| **zitadel** | — | `apps/zitadel/` | **active** | ExternalDomain `auth.nethub.co.ke`; DB `zitadel` on CNPG |
| **redis-shared** | `nethub_stack/services/redis.yaml` | `apps/redis-shared/` | **done** | Adopt in place |
| **cnpg / nethub-db-cluster** | R2 live | `apps/cnpg-nethub-db/` | **done** | Adopt in place |
| mazeltov | `k3s/mazeltov/` | TBD | pending | |
| cloudflared / middlewares | `nethub_stack/services/*` | TBD | pending | |

## Shared credentials (DB + Redis apps)

- **Secret** `nethub-db-app-creds` (ns `nethub`): shared `nethub_admin` for nethub-api, tawala (and was keycloak).
- **Secret** `nethub-redis-app-creds` (ns `nethub`): app Redis URL.
- **Secret** `zitadel-secrets`: Zitadel masterkey + first admin + Postgres env for Zitadel.

## Zitadel hard-cut (dev)

1. Keycloak Deployment/Ingress/Service removed from Flux.
2. CNPG `Database` `zitadel` owned by `nethub_admin`.
3. Job `drop-keycloak-db` drops Postgres database `keycloak` (FORCE).
4. Ingress `auth.nethub.co.ke` + `asfalis.nethub.co.ke` → Zitadel.
5. **NetHubKe / Tawala OIDC still pointed at Keycloak-shaped config** until app repos are updated — expect auth breakage until then.

### Retrieve first admin password

```bash
sops -d apps/zitadel/secret.enc.yaml | grep FIRSTINSTANCE
```

Masterkey is immutable after first successful init — do not change `masterkey` in secret after Zitadel has written data.

## CNPG adopt notes

- Namespace `postgres`, cluster `nethub-db-cluster`, PVC 15Gi local-path.
- DNS: `nethub-db-cluster-rw` / `nethub-db-pooler`.

## Zitadel ops notes

- Redis cache uses **DB index 10** only (`zitadel-redis-cache` secret). Do not point at DB 0 (Tawala/apps).
- Traefik must use **h2c** to the Zitadel service (`serversscheme: h2c`) or console shows "Unknown Content-type received" on gRPC-Web calls.
- CPU request: 300m.

## IdP-agnostic NetHubKe (app work, not this repo)

Prefer env names `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_JWKS_URL` (or discovery from issuer) instead of `KEYCLOAK_*`. Auth.js: generic OIDC provider, not `providers/keycloak`. Claim mapping adapter for roles — do not hardcode `realm_access` only.
