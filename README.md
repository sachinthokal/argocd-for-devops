# GitOps Deployment Repository with ArgoCD & Kubernetes

This repository serves as the **Single Source of Truth (GitOps Config Repo)** for managing, configuring, and deploying cluster add-ons, core services, and applications using **ArgoCD**.

---

## 📌 GitOps Workflow

```text
[ Application Repo (CI) ] ──(Build & Push Image)──► [ Container Registry ]
            │                                             │
            │ (Updates Image Tag via Git Commit)          │ (Pulls Image)
            ▼                                             ▼
[ argocd-for-devops (This Repo) ] ◄───[ ArgoCD ] ───► [ Kubernetes Cluster ]

```

1. Applications are built and packaged in their respective source/CI repositories.
2. The CI pipeline pushes the new image to the container registry and updates the deployment manifest in this repository.
3. **ArgoCD** continuously monitors this repository and automatically reconciles changes onto the target Kubernetes cluster.

---

## 📂 Repository Structure & Components

```text
.
├── cluster-addons/            # Cluster-wide foundational add-ons
│   ├── calico/                # CNI network plugin configuration & ArgoCD app
│   └── ingress-nginx/         # Ingress Controller Helm/manifest setup
├── fin-track-app/             # Application with Calico Network Policies (Egress control)
├── joke-app/                  # Microservice deployment manifests
├── k8s-probes-app/            # Application demonstrating Liveness/Readiness probes
├── k8s-vault-sync-app/        # Azure Key Vault Secrets Store CSI integration
├── message-viewer-app/        # App configured with ConfigMaps and Secrets
├── sys-info-app/              # System information service manifests
└── vault-auth-k8s-app/        # HashiCorp Vault K8s authentication & secret/data mounts
    ├── data/
    ├── data-mount/
    └── secrets-mount/

```

### Component Details

* **`cluster-addons/`**: Core infrastructure managed via GitOps (Calico CNI, Ingress NGINX controller).
* **`fin-track-app/`**: Demonstrates network security using egress policies (`egress-allow-selective.yaml`, `egress-deny-all`).
* **`k8s-vault-sync-app` & `vault-auth-k8s-app**`: Production secrets management patterns integrating Azure Key Vault / HashiCorp Vault via CSI and ConfigMap mounts.
* **`k8s-probes-app`**: Standard health check, self-healing, and lifecycle configurations.
* **`joke-app` / `sys-info-app` / `message-viewer-app**`: Containerized workload deployments showcasing standard Kubernetes primitives (Deployments, Services, ConfigMaps, Namespaces).

---

## 🚀 Getting Started with ArgoCD

### Prerequisites

* Kubernetes Cluster (v1.24+)
* `kubectl` configured with cluster access
* ArgoCD installed in the `argocd` namespace

### Registering an Application in ArgoCD

To deploy any of the applications (e.g., `joke-app`), define an ArgoCD `Application` manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: joke-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: '[https://github.com/sachinthokal/argocd-for-devops.git](https://github.com/sachinthokal/argocd-for-devops.git)'
    targetRevision: main
    path: joke-app
  destination:
    server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

```

Apply the application manifest:

```bash
kubectl apply -f application.yaml

```

---

## 🔄 Rollback & Self-Healing

* **Declarative Rollback:** To revert a deployment to a previously stable version, revert the corresponding commit in Git:

```bash
git revert <commit-hash>
git push origin main

```

* **Self-Healing:** ArgoCD actively detects manual drift or accidental changes made via `kubectl` and automatically reconciles the cluster back to the state declared in this repository.

---
