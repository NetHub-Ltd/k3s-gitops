# Rollback (NetPay Flux)
1. Revert PR on dev/main — removes netpay from apps kustomization; Flux prunes Deployment/Service/Ingress.
2. ImageRepository/Policy removed with cluster kustomization revert.
3. CNPG Database `netpay` remains unless explicitly deleted (data-preserving default).
4. To drop DB only after approval: delete Database CR `netpay` in namespace postgres.
