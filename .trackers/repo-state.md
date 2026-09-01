# Repository State

**Repo:** https://github.com/NetHub-Ltd/k3s-gitops.git  
**Default branch:** `main`  
**Topic branch:** `fix/flux-image-api-v1`

## Notes
- flux-system Ready=False: no matches for kind ImagePolicy in version v1beta2
- Cluster gotk-components CRDs serve image.toolkit.fluxcd.io/v1 only
- GHCR has tawala-api:0.0.40; deployment still 0.0.39 until automation runs
