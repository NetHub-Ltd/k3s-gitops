# Task: NetPay shared-secret wiring + crash fix
- [x] Align deployment with tawala/nethub: envFrom netpay-secrets + REDIS_URL from nethub-redis-app-creds
- [x] startupProbe for Alembic boot
- [ ] Operator: sops-edit DATABASE_URL with real password from nethub-db-app-creds
- [ ] Confirm pod /health after secret fix
