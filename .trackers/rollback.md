# Rollback (limited after DROP DATABASE)
1. Revert this PR on main — restores Keycloak manifests in git.
2. Keycloak **database** is dropped by Job — restore from R2 barman backup if you need Keycloak data back.
3. Zitadel masterkey must be preserved if keeping Zitadel data.
