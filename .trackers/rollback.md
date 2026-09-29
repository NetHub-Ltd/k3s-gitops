# Rollback
1. Revert the PR commit on main (or remove `- cnpg-nethub-db` from apps/kustomization.yaml and push).
2. Flux will stop managing Cluster/Pooler desired state; live objects remain (prune must not delete them if they were pre-existing — if prune attempts removal, disable prune or re-apply live YAML immediately).
3. Never delete Cluster CR or PVC nethub-db-cluster-1 to "rollback".
