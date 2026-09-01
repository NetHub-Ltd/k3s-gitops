# k3s-gitops

Single source of truth for the NetHub-Ltd k3s cluster, managed by **Flux**.

## Structure

```
clusters/k3s/          # Flux entry point for this cluster
infrastructure/        # Cluster-wide components (ingress, cert-manager, etc.)
apps/                  # Application manifests (one folder per service)
```

## Prerequisites

- Flux CLI
- `kubectl` access to the k3s cluster
- `age` + `sops` installed locally
- Fine-grained GitHub PAT with `contents: write` on this repository

## Bootstrap (one-time)

```bash
export GITHUB_TOKEN=<fine-grained-PAT>
export GITHUB_USER=NetHub-Ltd
export GITHUB_REPO=k3s-gitops

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=clusters/k3s \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller \
  --token-auth
```

After bootstrap finishes, create the SOPS decryption secret:

```bash
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

Then enable decryption on the root Kustomizations (already prepared in this repo).

## Adding a new service

1. Create `apps/<service-name>/` with Deployment, Service, and any Secrets.
2. Encrypt secrets with SOPS:

   ```bash
   sops --encrypt --in-place apps/<service-name>/secret.yaml
   ```

3. Add the folder to `apps/kustomization.yaml`.
4. Commit & push → Flux deploys it.

## Image Automation

Flux Image Automation is enabled. Mark image fields like this:

```yaml
image: ghcr.io/nethub-ltd/my-service:1.0.0  # {"$imagepolicy": "flux-system:my-service"}
```

## Secrets

All secrets are encrypted with **SOPS + age**.  
Private key (`age.agekey`) must never be committed. It lives only on the cluster as the `sops-age` secret and in your password manager.
