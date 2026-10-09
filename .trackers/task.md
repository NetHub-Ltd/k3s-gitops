# Task: NetPay Flux deploy
- [x] apps/netpay manifests (deployment, service, ingress, secret.enc.yaml)
- [x] CNPG Database CR netpay
- [x] ImageRepository + ImagePolicy (1.1.x)
- [x] Wire into apps/ and clusters/k3s/ kustomizations
- [ ] Operator: sops-edit DATABASE_URL with real DB password
- [ ] Operator: set OIDC_CLIENT_ID after Zitadel SPA client created
- [ ] Operator: confirm ADMIN_EMAIL
- [ ] Post-merge: DNS pay.nethub.co.ke → cluster; verify /health and /config.json
