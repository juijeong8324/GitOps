# GitOps

Kubernetes manifests for [WebGoDB](https://github.com/juijeong8324/WebGoDummy).  
Managed by Argo CD — any change pushed to this repository is automatically synced to the GKE cluster.

---

## Architecture

```
Developer pushes code to WebGoDummy repo
        ↓
GitHub Actions (CI)
  → docker build & push → Docker Hub (juijeong8324/web-go-db:<tag>)
  → Update image tag in this repo (04-deploy.yaml)
        ↓
Argo CD (CD)
  → Detects change in this repo
  → Automatically syncs to GKE cluster
```

---

## Infrastructure Overview

| Component | Detail                        |
| --------- | ----------------------------- |
| Cloud     | GCP (GKE)                     |
| Region    | asia-northeast3               |
| Ingress   | GCE Load Balancer             |
| DB        | PostgreSQL 15 (StatefulSet)   |
| Autoscale | HPA (CPU 50%, min 2 / max 10) |

---

## File Structure

```
GitOps/
├── 01-secret.yaml         # DB credentials (gitignored — apply manually)
├── 03-postgres.yaml       # ConfigMap + PVC + PostgreSQL StatefulSet + Service
├── 04-deploy.yaml         # Go app Deployment (initContainer + env injection)
├── 05-hpa.yaml            # Horizontal Pod Autoscaler
├── 06-ingress.yaml        # Ingress (GCE Load Balancer)
├── 07-networkpolicy.yaml  # NetworkPolicy (restrict DB access)
├── 08-service.yaml        # Go app Service (ClusterIP + BackendConfig)
└── 09-backendconfig.yaml  # GCE health check path (/healthz)
```

---

## Initial Deployment

### 1. Apply secret manually (not in Git)

```bash
kubectl apply -f 01-secret.yaml
```

### 2. Apply all manifests

```bash
kubectl apply -f 03-postgres.yaml
kubectl apply -f 04-deploy.yaml
kubectl apply -f 05-hpa.yaml
kubectl apply -f 06-ingress.yaml
kubectl apply -f 07-networkpolicy.yaml
kubectl apply -f 08-service.yaml
kubectl apply -f 09-backendconfig.yaml
```

---

## Argo CD Setup

### 1. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Connect this repository

Argo CD UI → Settings → Repositories → Connect Repo

```
Repository URL: https://github.com/juijeong8324/GitOps.git
```

### 3. Create Application

```
Application Name: deploy-web
Project:          default
Sync Policy:      Automatic
Repository URL:   https://github.com/juijeong8324/GitOps.git
Path:             .
Namespace:        default
```

---

## Network Policy

| Rule                | Detail                 |
| ------------------- | ---------------------- |
| Go app → PostgreSQL | Only port 5432 allowed |
| Ingress → Go app    | Only port 8080 allowed |
| Others → PostgreSQL | Blocked                |

---
