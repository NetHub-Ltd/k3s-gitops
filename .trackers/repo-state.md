# Repository State

**Repo:** https://github.com/NetHub-Ltd/k3s-gitops.git  
**Default branch:** `main`  
**Topic branch:** `fix/flux-apply-image-automation`  
**Preferred deploy:** Flux → k3s

## Current focus
Ensure ImageRepository / ImagePolicy / ImageUpdateAutomation are applied so tawala-api can auto-advance image tags.

## Notes
- apps/tawala-api pins `ghcr.io/nethub-ltd/tawala-api:0.0.39` with imagepolicy marker
- GHCR has `0.0.40`; cluster lacked ImageRepository/ImagePolicy (NotFound)
