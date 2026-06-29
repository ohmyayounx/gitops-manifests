# M3S on GKE - Step-by-Step Setup Guide

## Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                        GKE CI/CD ARCHITECTURE                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Developer Push → GitHub (app repo, branch: master)                    │
│       │                                                                │
│       ▼                                                                │
│  Jenkins (CI) - di K3s server existing                                 │
│    1. Build Docker image                                               │
│    2. Push ke DockerHub                                                │
│    3. Update image tag di gitops-manifests                             │
│       │                                                                │
│       ▼                                                                │
│  GitHub (gitops-manifests, branch: main)                               │
│       │                                                                │
│       ▼                                                                │
│  ArgoCD (bisa di K3s atau GKE)                                         │
│    - Watch gitops repo                                                 │
│    - Sync ke GKE cluster                                               │
│       │                                                                │
│       ▼                                                                │
│  ┌─────────────────────────────────────────────────┐                   │
│  │  GKE Cluster (asia-southeast2)                  │                   │
│  │                                                  │                   │
│  │  ┌─────────────────┐  ┌─────────────────┐      │                   │
│  │  │  m3s-ms-api     │  │  m3s-mm-api     │      │                   │
│  │  │  - Deployment   │  │  - Deployment   │      │                   │
│  │  │  - Service      │  │  - Service      │      │                   │
│  │  │  - Ingress(LB)  │  │  - Ingress(LB)  │      │                   │
│  │  │  - HPA          │  │  - HPA          │      │                   │
│  │  └─────────────────┘  └─────────────────┘      │                   │
│  │                                                  │                   │
│  │  External Secrets ← GCP Secret Manager           │                   │
│  │  ManagedCertificate (auto SSL)                   │                   │
│  │  Cloud Load Balancer (auto provisioned)          │                   │
│  └─────────────────────────────────────────────────┘                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

## Perbedaan K3s vs GKE

| Aspect | K3s (Development) | GKE (Production) |
|--------|-------------------|------------------|
| Ingress | Traefik | GCP Cloud Load Balancer |
| SSL/TLS | Manual cert | ManagedCertificate (auto) |
| Secrets | Vault (internal) | GCP Secret Manager |
| Scaling | HPA (limited nodes) | HPA + Cluster Autoscaler |
| Network | Flat network | VPC-native + NetworkPolicy |
| Auth | kubeconfig file | Workload Identity |
| Static IP | N/A | Global static IP |

---

## STEP 1: Prerequisites

```bash
# Install gcloud CLI (jika belum)
# https://cloud.google.com/sdk/docs/install

# Login
gcloud auth login
gcloud config set project it-re-492803

# Enable required APIs
gcloud services enable \
  container.googleapis.com \
  secretmanager.googleapis.com \
  compute.googleapis.com \
  artifactregistry.googleapis.com
```

## STEP 2: Buat GKE Cluster

```bash
# Autopilot cluster (Google manage nodes)
gcloud container clusters create-auto m3s-production \
  --region asia-southeast2 \
  --project it-re-492803

# Get credentials (merge ke local kubeconfig)
gcloud container clusters get-credentials m3s-production \
  --region asia-southeast2 \
  --project it-re-492803

# Verify
kubectl get nodes
```

## STEP 3: Reserve Static IP untuk Ingress

```bash
# Buat global static IP
gcloud compute addresses create m3s-ms-api-ip --global
gcloud compute addresses create m3s-mm-api-ip --global

# Lihat IP yang didapat
gcloud compute addresses describe m3s-ms-api-ip --global --format="get(address)"
gcloud compute addresses describe m3s-mm-api-ip --global --format="get(address)"

# → Update DNS record di Cloudflare/DNS provider:
#   ms-m3sbri-api.pcsindonesia.com  → <IP ms-api>
#   mm-m3sbri-api.pcsindonesia.com  → <IP mm-api>
```

## STEP 4: Setup GCP Secret Manager

```bash
# Buat secrets di GCP Secret Manager
# Format: JSON object dengan semua env vars

# ms-api environment variables
cat <<'EOF' > /tmp/ms-api-env.json
{
  "GIN_MODE": "release",
  "APP_NAME": "MS-API",
  "APP_DEBUG": "false",
  "APP_URL": "ms-m3sbri-api.pcsindonesia.com",
  "APP_PORT": "8002",
  "APP_TIMEZONE": "Asia/Jakarta",
  "DATABASE_HOST": "10.20.0.5",
  "DATABASE_PORT": "3306",
  "DATABASE_USERNAME": "production-apps",
  "DATABASE_PASSWORD": "YOUR_DB_PASSWORD",
  "DATABASE_MM": "production_bri_pcs_master_mm",
  "DATABASE_MS": "production_bri_pcs_master_ms",
  "JWT_SECRET": "YOUR_JWT_SECRET",
  "JWT_SECRET_GENERAL": "YOUR_JWT_GENERAL",
  "AES_KEY": "YOUR_AES_KEY",
  "AES_SALT": "YOUR_AES_SALT",
  "PUBSUB_PROJECT_ID": "it-re-492803"
}
EOF

gcloud secrets create m3s-ms-api-env \
  --data-file=/tmp/ms-api-env.json

# ms-api service account
gcloud secrets create m3s-ms-api-service-account \
  --data-file=path/to/service_account_credential.json

# mm-api (sama formatnya)
gcloud secrets create m3s-mm-api-env \
  --data-file=/tmp/mm-api-env.json

gcloud secrets create m3s-mm-api-service-account \
  --data-file=path/to/mm_service_account.json

# Cleanup local files
rm -f /tmp/ms-api-env.json /tmp/mm-api-env.json
```

## STEP 5: Setup Workload Identity

```bash
# Buat GCP Service Account untuk External Secrets
gcloud iam service-accounts create external-secrets-sa \
  --display-name="External Secrets Operator"

# Grant access ke Secret Manager
gcloud projects add-iam-policy-binding it-re-492803 \
  --member="serviceAccount:external-secrets-sa@it-re-492803.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Link ke K8s ServiceAccount (Workload Identity binding)
gcloud iam service-accounts add-iam-policy-binding \
  external-secrets-sa@it-re-492803.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:it-re-492803.svc.id.goog[external-secrets/external-secrets]"

# Buat SA untuk apps (optional, jika app butuh akses GCP langsung)
gcloud iam service-accounts create m3s-ms-api \
  --display-name="MS API Service"

gcloud iam service-accounts create m3s-mm-api \
  --display-name="MM API Service"

# Binding Workload Identity untuk app SAs
gcloud iam service-accounts add-iam-policy-binding \
  m3s-ms-api@it-re-492803.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:it-re-492803.svc.id.goog[m3s-ms-api/ms-api-sa]"

gcloud iam service-accounts add-iam-policy-binding \
  m3s-mm-api@it-re-492803.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:it-re-492803.svc.id.goog[m3s-mm-api/mm-api-sa]"
```

## STEP 6: Install External Secrets Operator

```bash
# Add Helm repo
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Install dengan Workload Identity annotation
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set serviceAccount.annotations."iam\.gke\.io/gcp-service-account"=external-secrets-sa@it-re-492803.iam.gserviceaccount.com

# Verify
kubectl get pods -n external-secrets
```

## STEP 7: Install ArgoCD di GKE (atau connect dari existing)

### Opsi A: ArgoCD baru di GKE
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Expose ArgoCD (untuk akses UI)
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### Opsi B: ArgoCD di K3s manage GKE cluster (multi-cluster)
```bash
# Di server K3s yang ada ArgoCD:
argocd cluster add gke_it-re-492803_asia-southeast2_m3s-production

# Ini akan register GKE sebagai target cluster di ArgoCD
# Lalu dapatkan server URL:
argocd cluster list
```

Setelah dapat endpoint, update file `argocd-apps/m3s-ms-api-gke.yaml`:
```yaml
destination:
  server: https://<GKE_ENDPOINT>  # Ganti!
```

## STEP 8: Buat DockerHub Pull Secret

```bash
# Buat secret untuk pull private images dari DockerHub
kubectl create secret docker-registry dockerhub-pull \
  --namespace=m3s-ms-api \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=devopspcs \
  --docker-password=<PASSWORD>

kubectl create secret docker-registry dockerhub-pull \
  --namespace=m3s-mm-api \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=devopspcs \
  --docker-password=<PASSWORD>
```

## STEP 9: Apply ArgoCD Applications

```bash
kubectl apply -f argocd-apps/m3s-ms-api-gke.yaml
kubectl apply -f argocd-apps/m3s-mm-api-gke.yaml

# Monitor sync status
kubectl get applications -n argocd
```

## STEP 10: Setup Jenkins Jobs

Buat 2 Jenkins Pipeline jobs baru:
- **m3s-ms-api-gke-deploy** → Pipeline from SCM → `gitops-manifests/m3s-ms-api-gke/Jenkinsfile`
- **m3s-mm-api-gke-deploy** → Pipeline from SCM → `gitops-manifests/m3s-mm-api-gke/Jenkinsfile`

## STEP 11: Verify

```bash
# Check deployments
kubectl get all -n m3s-ms-api
kubectl get all -n m3s-mm-api

# Check ExternalSecrets synced
kubectl get externalsecrets -A

# Check ingress & SSL cert
kubectl get ingress -A
kubectl get managedcertificates -A

# Test endpoint
curl -v https://ms-m3sbri-api.pcsindonesia.com/health
curl -v https://mm-m3sbri-api.pcsindonesia.com/health
```

---

## Networking: Database Access

Database kalian di `10.20.0.5:3306` (internal network). Dari GKE, ada beberapa opsi:

### Opsi 1: Cloud VPN (recommended)
```bash
# Buat VPN tunnel antara GKE VPC dan network on-prem
gcloud compute vpn-tunnels create m3s-vpn \
  --peer-address=<ON_PREM_PUBLIC_IP> \
  --shared-secret=<SECRET> \
  --region=asia-southeast2 \
  --target-vpn-gateway=<GATEWAY>
```

### Opsi 2: Cloud SQL Proxy (jika migrasi ke Cloud SQL)
Migrasi database ke Cloud SQL dan pakai Cloud SQL Auth Proxy sidecar.

### Opsi 3: Expose DB via public IP + firewall rules
Paling simple tapi kurang secure. Whitelist GKE NAT IP saja.

---

## Cost Estimation (Monthly)

| Resource | Estimated Cost |
|----------|---------------|
| GKE Standard (2x e2-medium) | ~$50 |
| Cloud Load Balancer (2) | ~$36 |
| Static IP (2) | ~$14 |
| Secret Manager | ~$0.06 |
| Egress (estimasi 10GB) | ~$1 |
| **Total** | **~$100/month** |

GKE Autopilot bisa lebih murah jika traffic rendah (pay per pod resource).

---

## Troubleshooting

### ManagedCertificate stuck "Provisioning"
- DNS harus sudah point ke static IP
- Tunggu 15-60 menit untuk provisioning
- `kubectl describe managedcertificate ms-api-cert -n m3s-ms-api`

### Pod CrashLoopBackOff
- `kubectl logs -f deployment/ms-api -n m3s-ms-api`
- Check ExternalSecret sudah sync: `kubectl get externalsecret -n m3s-ms-api`

### Ingress no address
- Tunggu 3-5 menit setelah create
- Check events: `kubectl describe ingress ms-api -n m3s-ms-api`
