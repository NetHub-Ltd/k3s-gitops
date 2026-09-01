# k3s-gitops

**Single source of truth** for the NetHub-Ltd k3s cluster, managed by [Flux](https://fluxcd.io).

This repository controls everything that runs on the cluster.  
You never run `kubectl apply` by hand for applications — you commit to this repo and Flux does the rest.

---

## Table of Contents

1. [Repository Structure](#1-repository-structure)
2. [Prerequisites](#2-prerequisites)
3. [How to Add a New Service](#3-how-to-add-a-new-service)
4. [Secrets with SOPS + age](#4-secrets-with-sops--age)
5. [Image Automation (auto-deploy new images)](#5-image-automation-auto-deploy-new-images)
6. [Service Repository CI (GitHub Actions)](#6-service-repository-ci-github-actions)
7. [Full End-to-End Flow](#7-full-end-to-end-flow)
8. [Day-2 Operations](#8-day-2-operations)
9. [Troubleshooting](#9-troubleshooting)
10. [Bootstrap Reference (already done)](#10-bootstrap-reference-already-done)

---

## 1. Repository Structure

```text
k3s-gitops/
├── .sops.yaml                     # Encryption rules (age public key)
├── .gitignore
├── README.md                      # This guide
├── clusters/
│   └── k3s/                       # Entry point for this cluster
│       ├── flux-system/           # Created by flux bootstrap (do not edit by hand)
│       ├── infrastructure.yaml    # Flux Kustomization → infrastructure/
│       ├── apps.yaml              # Flux Kustomization → apps/
│       └── image-automation.yaml  # Image update automation
├── infrastructure/
│   ├── sources/                   # HelmRepositories, OCIRepositories, etc.
│   ├── controllers/               # cert-manager, ingress, monitoring, …
│   └── kustomization.yaml
└── apps/
    ├── example-app/               # Working example (podinfo)
    │   ├── namespace.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── secret.enc.yaml        # SOPS-encrypted secret
    │   └── kustomization.yaml
    └── kustomization.yaml         # Lists every app folder
```

**Key idea**

- `clusters/k3s/` = what Flux watches
- `apps/` = your applications (one folder per service)
- `infrastructure/` = cluster-wide components

---

## 2. Prerequisites

On your laptop / jump host you need:

| Tool | Purpose | Install |
|------|---------|---------|
| `kubectl` | Talk to the cluster | already installed |
| `flux` | Flux CLI | `curl -s https://fluxcd.io/install.sh \| sudo bash` |
| `sops` | Encrypt/decrypt secrets | [GitHub releases](https://github.com/getsops/sops/releases) or `brew install sops` |
| `age` | Encryption backend for SOPS | [GitHub releases](https://github.com/FiloSottile/age/releases) or `brew install age` |
| GitHub PAT | Used by Flux & CI | fine-grained, `contents: write` on this repo |

You must also have the **age private key** (`age.agekey`) stored safely (password manager).  
The corresponding public key is already configured in `.sops.yaml`.

---

## 3. How to Add a New Service

### 3.1 Create the folder

```bash
# From the root of this repo
mkdir -p apps/my-service
```

### 3.2 Minimal files

**`apps/my-service/namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-service
  labels:
    app.kubernetes.io/name: my-service
```

**`apps/my-service/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  namespace: my-service
  labels:
    app.kubernetes.io/name: my-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: my-service
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-service
    spec:
      containers:
        - name: app
          # Mark the image for Flux Image Automation (see section 5)
          image: ghcr.io/nethub-ltd/my-service:0.1.0  # {"$imagepolicy": "flux-system:my-service"}
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - secretRef:
                name: my-service-secrets
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

**`apps/my-service/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: my-service
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: my-service
  ports:
    - name: http
      port: 80
      targetPort: http
```

**`apps/my-service/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: my-service
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - secret.enc.yaml          # created in the next section
```

### 3.3 Register the service

Edit `apps/kustomization.yaml` and add your folder:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - example-app
  - my-service          # ← add this line
```

### 3.4 Commit & push

```bash
git add apps/my-service apps/kustomization.yaml
git commit -m "feat(apps): add my-service"
git push
```

Flux will detect the change within ~1 minute and deploy the service.

---

## 4. Secrets with SOPS + age

**Never commit plain-text secrets.**

### 4.1 Create a secret file

```yaml
# apps/my-service/secret.yaml  (plain text – temporary)
apiVersion: v1
kind: Secret
metadata:
  name: my-service-secrets
  namespace: my-service
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:password@db:5432/mydb"
  API_KEY: "super-secret-value"
  ANOTHER_SECRET: "..."
```

### 4.2 Encrypt it

```bash
# From the root of the repo (so .sops.yaml is picked up)
sops --encrypt --in-place apps/my-service/secret.yaml

# Rename for clarity (optional but recommended)
mv apps/my-service/secret.yaml apps/my-service/secret.enc.yaml
```

Only the values under `data` / `stringData` are encrypted.  
Metadata stays readable → clean Git diffs.

### 4.3 Decrypt locally (when you need to edit)

```bash
sops apps/my-service/secret.enc.yaml          # opens in $EDITOR
# or
sops --decrypt apps/my-service/secret.enc.yaml
```

### 4.4 Rotate a secret

1. Decrypt → edit → re-encrypt
2. Commit & push
3. Flux applies the new secret and restarts pods that reference it (if you use `envFrom` or volume mounts that trigger restart)

---

## 5. Image Automation (auto-deploy new images)

Goal: when a new image is pushed to GHCR, Flux automatically updates the tag in this repo and deploys it.

### 5.1 Mark the image in the Deployment

In `deployment.yaml`:

```yaml
image: ghcr.io/nethub-ltd/my-service:0.1.0  # {"$imagepolicy": "flux-system:my-service"}
```

The comment is a **marker**. Flux Image Automation looks for it.

### 5.2 Create ImageRepository + ImagePolicy

Add these files (or put them under `clusters/k3s/` / a dedicated folder):

**`clusters/k3s/image-my-service.yaml`** (example)

```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: my-service
  namespace: flux-system
spec:
  image: ghcr.io/nethub-ltd/my-service
  interval: 5m
  # If the package is private, add:
  # secretRef:
  #   name: ghcr-credentials

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-service
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: my-service
  policy:
    semver:
      range: ">=0.1.0"
  # Or for "always latest":
  # policy:
  #   alphabetical:
  #     order: asc
```

### 5.3 Enable the update automation

Edit `clusters/k3s/image-automation.yaml` (or create it) so it looks like:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: |
        chore(images): update {{range .Updated.Images}}{{.}} {{end}}
    push:
      branch: main
  update:
    path: ./apps
    strategy: Setters
```

Commit & push. From now on Flux will:

1. Scan GHCR every 5 minutes
2. Pick the newest tag that matches the policy
3. Commit the new tag into this GitOps repo
4. Deploy the new version

---

## 6. Service Repository CI (GitHub Actions)

Put this workflow in **every service repository** under `.github/workflows/build-and-push.yml`:

```yaml
name: Build and Push to GHCR

on:
  push:
    branches: [main, master]
    tags: ["v*"]
  pull_request:
    branches: [main, master]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}   # e.g. nethub-ltd/my-service

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest,enable={{is_default_branch}}
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Important**

- The workflow uses the built-in `GITHUB_TOKEN` → no extra secrets needed for public/private packages in the same org.
- For private packages, Flux needs a `docker-registry` secret named e.g. `ghcr-credentials` (see section 5).

---

## 7. Full End-to-End Flow

1. Developer pushes code to the **service repository**.
2. GitHub Actions builds the image and pushes it to `ghcr.io/nethub-ltd/<service>:<tag>`.
3. Flux Image Reflector sees the new tag.
4. Image Policy selects the newest matching tag.
5. Image Automation commits the new tag into **this GitOps repo**.
6. Flux Kustomization sees the change → reconciles → new pods roll out.
7. Secrets are decrypted on-the-fly by the kustomize-controller using the `sops-age` secret.

You only ever touch Git. The cluster stays in sync automatically.

---

## 8. Day-2 Operations

### Force a reconcile

```bash
flux reconcile source git flux-system
flux reconcile kustomization apps
flux reconcile kustomization infrastructure
```

### Suspend / resume an app

```bash
flux suspend kustomization apps
flux resume kustomization apps
```

### See what Flux is doing

```bash
flux get all -A
flux logs --level=error
flux events --for Kustomization/apps
```

### Update Flux itself

Flux self-manages. When a new version is released, update the manifests in `clusters/k3s/flux-system/` (or re-run bootstrap with a newer CLI).

---

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `secrets "sops-age" not found` | Age private key not loaded | `kubectl create secret generic sops-age …` (see bootstrap section) |
| Kustomization stuck on `DependencyNotReady` | Parent Kustomization failing | Check the parent with `flux get kustomizations` and `flux logs` |
| Image never updates | Marker missing or wrong policy | Verify the `# {"$imagepolicy": "..."}` comment and the ImagePolicy range |
| `ImageRepository` / `ImagePolicy` NotFound | `clusters/k3s/*.yaml` image manifests not applied | Ensure `clusters/k3s/kustomization.yaml` lists `image-tawala-api.yaml` and `image-automation.yaml`; reconcile `flux-system` |
| Secret values are still encrypted in the cluster | Decryption not enabled on the Kustomization | Ensure `spec.decryption.provider: sops` is present |
| Pods CrashLoopBackOff after secret change | Application does not reload env | Restart the Deployment: `kubectl rollout restart deployment/…` |

Useful commands:

```bash
# Decrypt a secret locally to verify content
sops -d apps/my-service/secret.enc.yaml

# Check what Flux sees
kubectl get imagerepository,imagepolicy -n flux-system
kubectl describe kustomization apps -n flux-system
```

---

## 10. Bootstrap Reference (already done)

These steps were completed when the cluster was first set up. Kept here for disaster recovery / new clusters.

```bash
# 1. Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# 2. Bootstrap
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

# 3. Load age private key
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

---

## Quick Start Checklist for a New Service

- [ ] Create `apps/<service>/` with namespace, deployment, service, kustomization
- [ ] Create & encrypt secret → `secret.enc.yaml`
- [ ] Add folder to `apps/kustomization.yaml`
- [ ] (Optional) Add ImageRepository + ImagePolicy + image marker
- [ ] Commit & push
- [ ] Verify: `flux get kustomizations` and `kubectl get pods -n <service>`

---

**Questions / improvements?**  
Open an issue or PR in this repository.
