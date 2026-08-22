# Project Task Summary: 3-Tier Enterprise App (StelcoM) K8s Deployment

**Role:** DevOps Engineer

**Scope:** Set up cluster infra, configure HashiCorp Vault Secret Sync via CSI Driver, configure ingress routing, and deploy the containerized 3-Tier Telecom application with production-grade resource controls and health probes.

---

## 1. Stack Handover Matrix

| Tier | Component | Docker Image / Tech | Ports | Health Probe Strategy |
| --- | --- | --- | --- | --- |
| **Frontend** | Web UI (SPA) | `sachinthokal/stelcom-frontend:v8` | `80`, `443` | HTTP Get `/` on port 80 |
| **Backend** | Telecom Core Engine | `sachinthokal/stelcom-backend:v7` | `8080` | Spring Boot Actuator (`/api/actuator/health/*`) |
| **Database** | Primary Datastore | `postgres:15-alpine` | `5432` | CLI `pg_isready` (User: `stelcom_admin`, DB: `stelcom_db`) |

---

## 2. Infrastructure Setup & Add-ons Installation

Installed core cluster add-ons using Helm:

```bash
# 1. Add Helm Repositories
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# 2. Secrets Store CSI Driver (Enabled Secret Auto-Sync and Rotation)
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true \
  --set enableSecretRotation=true

# 3. HashiCorp Vault (Dev mode with CSI support enabled)
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.dev.enabled=true" \
  --set "server.service.type=NodePort" \
  --set "server.service.nodePort=30084" \
  --set "csi.enabled=true"

# 4. Ingress-NGINX Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

```

---

## 3. HashiCorp Vault RBAC & Secret Ingestion

Configured Vault secrets engine and Kubernetes RBAC auth inside the `vault-0` container:

```bash
kubectl exec -i vault-0 -n vault -- /bin/sh << 'EOF'

# 1. Enable K8s Auth Engine
vault auth enable kubernetes

# 2. Map Kubernetes API host
vault write auth/kubernetes/config kubernetes_host="https://kubernetes.default.svc:443"

# 3. Write App & DB Secrets
vault kv put secret/myapp/config \
  DB_DRIVER="org.postgresql.Driver" \
  POSTGRES_DB="stelcom_db" \
  DB_PASSWORD="StelcomSecurePass2026" \
  DB_URL="jdbc:postgresql://postgres-service:5432/stelcom_db" \
  DB_USER="stelcom_admin" \
  GATEWAY_API_KEY="stelcom-secret-key-9988"

# 4. Define Access Policy
vault policy write myapp-policy - <<EOP
path "secret/data/myapp/config" {
  capabilities = ["read"]
}
EOP

# 5. Bind Policy to K8s Service Account (myapp-sa in stel-ns)
vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=stel-ns \
  policies=myapp-policy \
  ttl=1h

EOF

```

---

## 4. Production Resource Allocation & Probes Architecture

### Resource Requests & Limits Sizing

| Container | CPU Request | CPU Limit | Memory Request | Memory Limit |
| --- | --- | --- | --- | --- |
| **stelcom-frontend** | `50m` | `200m` | `32Mi` | `64Mi` |
| **stelcom-backend** | `250m` | `1000m` | `256Mi` | `512Mi` |
| **postgres-container** | `100m` | `500m` | `256Mi` | `512Mi` |

---

### Health Checks & Failure Mitigation

* **Database (`postgres-statefulset`):**
* **Startup Probe:** `exec: pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"` (delay: 5s, period: 5s, failureThreshold: 20)
* **Liveness Probe:** `exec: pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"` (delay: 15s, period: 10s, failureThreshold: 3)
* **Readiness Probe:** `exec: pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"` (delay: 5s, period: 5s, failureThreshold: 3)

* **Backend (`stelcom-backend-deployment`):**
* **Startup Probe:** `httpGet: /api/actuator/health` (port: 8080, delay: 15s, period: 10s, failureThreshold: 30)
* **Liveness Probe:** `httpGet: /api/actuator/health/liveness` (port: 8080, delay: 30s, period: 15s, failureThreshold: 3)
* **Readiness Probe:** `httpGet: /api/actuator/health/readiness` (port: 8080, delay: 20s, period: 10s, failureThreshold: 3)

* **Frontend (`stelcom-frontend-deployment`):**
* **Startup Probe:** `httpGet: /` (port: 80, delay: 3s, period: 5s, failureThreshold: 6)
* **Liveness Probe:** `httpGet: /` (port: 80, delay: 5s, period: 10s, failureThreshold: 3)
* **Readiness Probe:** `httpGet: /` (port: 80, delay: 3s, period: 5s, failureThreshold: 3)

---

## 5. Deployment Manifest Execution Order

Executed and verified manifests in the designated deployment order:

```bash
kubectl apply -f 00-namespace.yaml                 # Created dedicated namespace 'stel-ns'
kubectl apply -f 01-stel-config.yaml               # Mounted DB schema initialization script
kubectl apply -f 02-stel-hashicorp-vault.yaml      # Bound SecretProviderClass & ServiceAccount (myapp-sa)
kubectl apply -f 03-stel-db-statefulsets.yaml      # Provisioned PostgreSQL with headless DNS service
kubectl apply -f 04-stel-backend-deployment.yaml   # Deployed Spring Boot app with Vault volume mount
kubectl apply -f 05-stel-frontend-deployment.yaml  # Deployed NGINX static UI container
kubectl apply -f 06-stel-ingress.yaml              # Exposed routes: UI (/) & API (/api) via Ingress

```

---

## 6. Verification & Validation Commands

* **Pods & Services Status:**

```bash
kubectl get pods,svc,ingress -n stel-ns -o wide

```

* **Verify Vault Injected Secret Sync:**

```bash
kubectl get secret app-env-secrets -n stel-ns

```

* **Direct Backend Health Check:**

```bash
curl -i http://localhost/api/actuator/health
curl -i http://localhost/api/admin/ping

```
