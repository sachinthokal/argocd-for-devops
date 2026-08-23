# Project Task Summary: 3-Tier Enterprise Telecom App (STelCoM) Cloud-Native Deployment

**Role:** DevOps & Cloud-Native Architect

**Scope:** Provision and configure an end-to-end 3-Tier containerized Telecom Application on Kubernetes. Manage secrets securely using HashiCorp Vault via CSI Driver, configure Kubernetes Gateway API routing via Kong API Gateway, and enforce enterprise-grade security policies (Key-Auth, Rate Limiting, Header Transformers, Observability Metrics, Correlation ID, and IP Allowlisting).

---

## 1. Stack Handover Matrix

| Tier | Component | Docker Image / Tech | Ports | Health Probe Strategy |
| --- | --- | --- | --- | --- |
| **Frontend** | Web UI Simulator | `sachinthokal/stelcom-frontend:v8` | `80`, `443` | HTTP Get `/` on port 80 |
| **Backend** | Telecom Core Engine | `sachinthokal/stelcom-backend:v7` | `8080` | Spring Boot Actuator (`/api/actuator/health/*`) |
| **Database** | Primary Datastore | `postgres:15-alpine` | `5432` | CLI `pg_isready` (User: `stelcom_admin`, DB: `stelcom_db`) |
| **API Gateway** | Kong Gateway | `kong:3.9.3` | `8083` (Proxy), `8100` (Status) | Native Liveness/Readiness probes |

---

## 2. Infrastructure Setup & Add-ons Installation

```bash
# 1. Add Helm Repositories
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo add kong https://charts.konghq.com
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

# 4. Kong API Gateway (Gateway API & Status Listeners Enabled)
helm install kong kong/kong \
  --namespace kong \
  --create-namespace \
  --set ingressController.gatewayDiscovery.enabled=true \
  --set proxy.type=NodePort \
  --set proxy.http.nodePort=30083 \
  --set env.status_listen="0.0.0.0:8100"

```

---

## 3. HashiCorp Vault RBAC & Secret Ingestion

Configured Vault secrets engine and mapped Kubernetes RBAC service accounts inside `vault-0`:

```bash
kubectl exec -i vault-0 -n vault -- /bin/sh << 'EOF'

# 1. Enable K8s Auth Engine
vault auth enable kubernetes

# 2. Map Kubernetes API host
vault write auth/kubernetes/config kubernetes_host="https://kubernetes.default.svc:443"

# 3. Write App & DB Secrets (including Gateway Consumer Key)
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

## 5. Deployment Manifest Execution Sequence

Apply manifests in sequential order:

```bash
# Core Cluster Setup & Secret Sync
kubectl apply -f 00-namespace.yaml                 # Dedicated namespace 'stel-ns'
kubectl apply -f 01-stel-config.yaml               # DB schema initialization ConfigMap
kubectl apply -f 02-stel-hashicorp-vault.yaml      # SecretProviderClass & myapp-sa
kubectl apply -f 03-stel-db-statefulsets.yaml      # PostgreSQL StatefulSet & Headless Service
kubectl apply -f 04-stel-backend-deployment.yaml   # Spring Boot core with Vault CSI mount
kubectl apply -f 05-stel-frontend-deployment.yaml  # NGINX Frontend Simulator UI

# Gateway API & Kong Security Policy Suite
kubectl apply -f 07-stel-gateway-setup.yaml        # GatewayClass & Gateway Definition
kubectl apply -f 08-stel-httproute.yaml            # HTTPRoute routing rules & filters
kubectl apply -f 09-stel-rate-limit.yaml           # Traffic control (5 req/min limit)
kubectl apply -f 10-stel-key-auth.yaml             # Kong Consumer & Vault API Key binding
kubectl apply -f 11-stel-transformers.yaml         # Header mutation & backend masking
kubectl apply -f 12-stel-prometheus.yaml           # Observability metrics scrape plugin
kubectl apply -f 13-stel-correlation-id.yaml       # Distributed request tracing UUID
kubectl apply -f 14-stel-ip-restriction.yaml       # Admin CIDR/IP allowlisting (403 Forbidden)

```

---

## 6. End-to-End Validation & Chaos Test Scenarios

**1. Public GET & Correlation ID Check (200 OK):**

```bash
curl -i http://localhost:8083/api/admin/ping

```

* **Result:** Returns `200 OK` with `X-Correlation-ID: <uuid>` injected by Kong.

**2. Unauthorized POST Security Lock (401 Unauthorized):**

```bash
curl -i -X POST http://localhost:8083/api/telecom/recharge \
  -H "Content-Type: application/xml" -d '<req>test</req>'

```

* **Result:** Returns `401 Unauthorized` (`No API key found in request`).

**3. Authorized POST Transaction with Transformers (200 OK):**

```bash
curl -i -X POST http://localhost:8083/api/telecom/recharge \
  -H "X-Api-Key: stelcom-secret-key-9988" \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"><soapenv:Body><tel:RechargeRequest xmlns:tel="http://stelcom.com/recharge"><tel:SubscriberMsisdn>9876543210</tel:SubscriberMsisdn><tel:PlanId>UNLTD_28D_1.5GB</tel:PlanId><tel:Amount currency="INR">299.00</tel:Amount></tel:RechargeRequest></soapenv:Body></soapenv:Envelope>'

```

* **Result:** Returns `200 OK`, `X-Powered-By: STelCoM-Engine-V6`, and decrements `RateLimit-Remaining`.

**4. Rate Limiting Denial-of-Service Defense (429 Too Many Requests):**
Execute 6 rapid requests. The 6th request triggers:

```text
HTTP/1.1 429 Too Many Requests
{"message":"API Rate limit exceeded! Please slow down"}

```

**5. IP Allowlist Security Barrier (403 Forbidden):**
Requests from non-whitelisted IPs or subnets:

```text
HTTP/1.1 403 Forbidden
{"message":"IP address not allowed"}

```

**6. Prometheus Observability Scraping:**

```bash
curl -s http://localhost:8100/metrics | grep "kong_http_requests_total"

```

* **Result:** Outputs per-consumer and per-status-code counters for Prometheus and Grafana dashboards.