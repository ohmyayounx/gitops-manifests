# M3S API - CI/CD Setup Guide

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         CI/CD FLOW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Developer Push → GitHub (app repo)                              │
│       │                                                          │
│       ▼                                                          │
│  Jenkins (CI)                                                    │
│    1. Checkout source code                                       │
│    2. Build Docker image                                         │
│    3. Push ke DockerHub                                          │
│    4. Update image tag di gitops-manifests repo                  │
│       │                                                          │
│       ▼                                                          │
│  GitHub (gitops-manifests repo) ← perubahan image tag            │
│       │                                                          │
│       ▼                                                          │
│  ArgoCD (CD)                                                     │
│    - Detect perubahan di Git                                     │
│    - Auto-sync manifest ke K3s cluster                           │
│       │                                                          │
│       ▼                                                          │
│  K3s Cluster                                                     │
│    - External Secrets Operator sync dari Vault                   │
│    - Pull image dari DockerHub                                   │
│    - Rolling update deployment                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Folder Structure

```
gitops-manifests/
├── argocd-apps/              # ArgoCD Application definitions
│   ├── itsm-production.yaml
│   ├── m3s-ms-api.yaml
│   └── m3s-mm-api.yaml
├── itsm-app/                 # ITSM manifests (sudah jalan)
├── m3s-ms-api/               # MS-API manifests (baru)
│   ├── 00-namespace.yaml
│   ├── 01-external-secret.yaml
│   ├── 02-deployment.yaml
│   ├── 03-service.yaml
│   ├── 04-ingress.yaml
│   ├── 05-hpa.yaml
│   ├── 06-dockerhub-pull-secret.yaml
│   └── Jenkinsfile
├── m3s-mm-api/               # MM-API manifests (baru)
│   ├── 00-namespace.yaml
│   ├── 01-external-secret.yaml
│   ├── 02-deployment.yaml
│   ├── 03-service.yaml
│   ├── 04-ingress.yaml
│   ├── 05-hpa.yaml
│   ├── 06-dockerhub-pull-secret.yaml
│   └── Jenkinsfile
└── M3s-ms-mm/                # (OLD - bisa dihapus setelah migrasi)
```

## Step-by-Step Setup

### Step 1: Install External Secrets Operator di K3s

```bash
# SSH ke K3s server
ssh tioyan@139.180.191.232

# Install via Helm
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set installCRDs=true
```

### Step 2: Buat Vault Credentials di masing-masing namespace

```bash
# Untuk ms-api
kubectl create namespace m3s-ms-api
kubectl create secret generic vault-credentials \
  --from-literal=username=devopspcs \
  --from-literal=password=<VAULT_PASSWORD> \
  -n m3s-ms-api

# Untuk mm-api
kubectl create namespace m3s-mm-api
kubectl create secret generic vault-credentials \
  --from-literal=username=devopspcs \
  --from-literal=password=<VAULT_PASSWORD> \
  -n m3s-mm-api
```

### Step 3: Setup Vault Secrets (pastikan path sesuai)

Pastikan secrets sudah ada di Vault dengan path:

| Vault Path | Isi |
|------------|-----|
| `development/data/m3s/api/ms-api/env` | Semua env vars (GIN_MODE, DATABASE_HOST, dll) |
| `development/data/m3s/api/ms-api/service_account_credential` | `{ "content": "<JSON service account>" }` |
| `development/data/m3s/api/mm-api/env` | Semua env vars untuk mm-api |
| `development/data/m3s/api/mm-api/service_account` | `{ "content": "<JSON service account>" }` |
| `development/data/m3s/docker/registry` | `{ "username": "devopspcs", "password": "<pass>" }` |

Jika path berbeda dengan yang sudah ada, update file `01-external-secret.yaml`.

### Step 4: Register ArgoCD Applications

```bash
# Apply semua ArgoCD apps
kubectl apply -f argocd-apps/m3s-ms-api.yaml
kubectl apply -f argocd-apps/m3s-mm-api.yaml

# Verify
kubectl get applications -n argocd
```

### Step 5: Setup Jenkins Jobs (CI only)

Buat 2 Jenkins Pipeline jobs baru:
1. **m3s-ms-api-ci** → Point ke `gitops-manifests/m3s-ms-api/Jenkinsfile`
2. **m3s-mm-api-ci** → Point ke `gitops-manifests/m3s-mm-api/Jenkinsfile`

Jenkins credentials yang diperlukan:
- `github-token` - GitHub username/password (atau Personal Access Token)
- `dockerhub-auth` - DockerHub username/password

### Step 6: Push ke Git & Verify

```bash
git add m3s-ms-api/ m3s-mm-api/ argocd-apps/
git commit -m "feat: add M3S API GitOps manifests"
git push origin main
```

Verify di ArgoCD UI:
- `m3s-ms-api` → should sync and show Healthy
- `m3s-mm-api` → should sync and show Healthy

## Troubleshooting

### ExternalSecret not syncing
```bash
kubectl get externalsecrets -n m3s-ms-api
kubectl describe externalsecret ms-api-env -n m3s-ms-api
```

### ArgoCD not syncing
```bash
kubectl get applications -n argocd
argocd app get m3s-ms-api
argocd app sync m3s-ms-api
```

### Image pull failed
```bash
kubectl get events -n m3s-ms-api
kubectl describe pod -l app=ms-api -n m3s-ms-api
```

## Differences from Old Setup

| Aspect | Old (M3s-ms-mm/) | New (m3s-ms-api/, m3s-mm-api/) |
|--------|-------------------|-------------------------------|
| Secrets | Jenkins fetch dari Vault & SSH ke server | External Secrets Operator auto-sync |
| Deploy | Jenkins SSH + kubectl | ArgoCD auto-sync dari Git |
| Image | `imagePullPolicy: Never` (local) | Pull dari DockerHub registry |
| Scaling | Manual replicas | HPA autoscaling |
| Health | No probes | Readiness + Liveness probes |
| Jenkins role | CI + CD | CI only (build + push + update Git) |
