# NetHubKe application DB reset (Architecture A)

## Scope
- **In scope:** empty the NetHubKe **app** database named in `nethub-db-app-creds.nethub_db_name`.
- **Out of scope:** CNPG cluster, PVC, `zitadel`, `netpay`, Redis.

## Prerequisites
- Confirm `nethub_db_name` is **not** `postgres`, `zitadel`, or `netpay`.
- Confirm `database-nethub.yaml` `spec.name` matches that secret value (default assumed `nethub`).
- Job refuses unsafe names.

## Procedure
1. Merge this PR; Flux applies `reset-nethub-app-db-v1`.
2. Watch: `kubectl -n nethub logs job/reset-nethub-app-db-v1`
3. Roll API: `kubectl -n nethub rollout restart deployment/nethub-api`
4. Confirm migrations + `GET /ready` on `api.nethub.co.ke`.
5. Remove `job-reset-nethub-app-db-v1.yaml` from `kustomization.yaml` and delete the Job.

## App env (already wired)
| Source | Keys |
|--------|------|
| `nethub-api-secrets` (SOPS) | `OIDC_ISSUER`, `OIDC_JWKS_URL`, `AUDIENCE`, optional duplicates of DB |
| `nethub-db-app-creds` | `host`, `port`, `username`, `password` → `FASTAPI_DB_*`; `nethub_db_name` → `FASTAPI_DB_NAME` |
| `nethub-api-config` (ConfigMap) | `ENVIRONMENT`, `ALLOWED_ORIGINS`, `JWT_LEEWAY_SECONDS`, `DIAGNOSTICS_ENABLED`, `RATE_LIMIT_*` |

App code already uses `FASTAPI_DB_*` / `OIDC_*` / `AUDIENCE` — no rename required.
