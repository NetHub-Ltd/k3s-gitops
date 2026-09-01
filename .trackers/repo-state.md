# Repository State

**Repo:** https://github.com/NetHub-Ltd/k3s-gitops.git  
**Default branch:** `main`  
**Topic branch:** `fix/image-policy-bounded-semver`

## Notes
- ImageRepository scan works (52 tags); unbounded policy risked electing 0.1.x over 0.0.40
- Pattern: per-app ImageRepository+ImagePolicy, shared automation + ghcr-credentials
