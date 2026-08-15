# GitOps Manifests with ArgoCD

This repository follows the **GitOps** methodology to manage cluster add-ons and application workloads on a Kubernetes (Kind) cluster using **Argo CD**.

---

## 📁 Repository Structure

```text
argocd-for-devops/
├── cluster-addons/
│   ├── calico/
│   │   ├── application.yaml      # ArgoCD Application for Calico CNI
│   │   └── calico.yaml           # Official Calico CNI Manifest (v3.28.0)
│   └── ingress-nginx/
│       ├── application.yaml      # ArgoCD Helm Application for NGINX Ingress
│       └── values.yaml           # Custom Helm values tailored for Kind
├── joke-app/
│   ├── application.yaml          # ArgoCD Application definition
│   └── deployment.yaml           # Workload manifests
├── sys-info-app/
│   ├── application.yaml          # ArgoCD Application definition
│   └── deployment.yaml           # Workload manifests
└── README.md

```

---

## 🚀 Architecture Overview

* **Infrastructure Provisioning**: Kind cluster and ArgoCD bootstrapping are managed via **Terraform**.
* **Networking (CNI)**: Custom **Calico CNI** (`192.168.0.0/16`) deployed via ArgoCD after disabling default Kind CNI.
* **Ingress Controller**: **Ingress-NGINX** bound to Kind control-plane node ports (`80`, `443`).
* **Workloads**: Microservices continuously synced and managed via ArgoCD.

---

## 🛠️ Deployment Instructions

### 1. Prerequisites

* Docker running locally
* `kubectl` installed and configured
* Kind cluster with ArgoCD provisioned via Terraform

### 2. Deploy Cluster Add-ons

#### A. Calico CNI (Required for Node Readiness)

```bash
kubectl apply -f [https://raw.githubusercontent.com/sachinthokal/argocd-for-devops/main/cluster-addons/calico/application.yaml](https://raw.githubusercontent.com/sachinthokal/argocd-for-devops/main/cluster-addons/calico/application.yaml)

```

Verify nodes are in `Ready` status:

```bash
kubectl get nodes

```

#### B. Ingress-NGINX

```bash
kubectl apply -f [https://raw.githubusercontent.com/sachinthokal/argocd-for-devops/main/cluster-addons/ingress-nginx/application.yaml](https://raw.githubusercontent.com/sachinthokal/argocd-for-devops/main/cluster-addons/ingress-nginx/application.yaml)

```

---

### 3. Deploy Workload Applications

Deploy apps directly through ArgoCD GitOps sync:

```bash
# Deploy Joke App
kubectl apply -f [https://raw.githubusercontent.com/sachinthokal/argocd-for-devops/main/joke-app/application.yaml](https://raw.githubusercontent.com/sachinthokal/argocd-for-devops/main/joke-app/application.yaml)

# Deploy System Info App
kubectl apply -f [https://raw.githubusercontent.com/sachinthokal/argocd-for-devops/main/sys-info-app/application.yaml](https://raw.githubusercontent.com/sachinthokal/argocd-for-devops/main/sys-info-app/application.yaml)

```

---

## 🔑 Accessing ArgoCD UI

1. Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

```

1. Open **[https://localhost:8085](https://localhost:8085)** in your browser.
2. Login using `admin` and the decoded password.

---
