# Project 2 — Multi-Tier Application Deployment

## Kubernetes Hands-On Learning Runbook

| Field       | Detail                                   |
|-------------|------------------------------------------|
| Project     | 02 — Multi-Tier Application              |
| Environment | Minikube v1.35.1 (1 control + 2 workers) |
| OS          | Red Hat Enterprise Linux 9.6 (Plow)      |
| Namespace   | project-2                                |
| Date        | May 2026                                 |
| Status      | ✅ Complete                               |


## Table of Contents

1. [1. Overview](#1-overview)
2. [2. Architecture](#2-architecture)
3. [3. Concepts Learned](#3-concepts-learned)
4. [4. Prerequisites](#4-prerequisites)
5. [5. Repository Structure](#5-repository-structure)
6. [6. Deployment — Step by Step](#6-deployment--step-by-step)
7. [7. Verification](#7-verification)
8. [8. Accessing the Application](#8-accessing-the-application)
9. [9. Key Operations](#9-key-operations)
10. [10. Troubleshooting](#10-troubleshooting)
11. [11. Cleanup](#11-cleanup)
12. [12. Quick Reference Card](#12-quick-reference-card)


## 1. Overview

Project 2 builds a **multi-tier application** inside Kubernetes, demonstrating how real-world applications are structured with multiple components that communicate internally using Kubernetes-native networking.

### What We Built

* A **Node.js backend API** that returns JSON data configured via environment variables
* An **nginx frontend** that serves an HTML page and proxies API calls to the backend
* A **ClusterIP service** for internal pod-to-pod communication (backend)
* A **NodePort service** for external browser access (frontend)
* **ConfigMaps** for storing HTML, nginx config, and application configuration
* **Environment variables** injected into pods from ConfigMaps

### Key Principles Demonstrated

* Services as stable DNS endpoints (pods are ephemeral, services are not)
* Separation of concerns (config stored in ConfigMaps, not in images)
* Namespace isolation (all resources in `project-2`)
* Dependency ordering (ConfigMaps before Deployments)
* Zero-downtime updates via rolling restarts


## 2. Architecture

### Topology

```text
Your Browser (Windows VDI)
 │
 │ MobaXterm SSH Tunnel → localhost:30080
 │
 ▼
RHEL Sandbox
 │ kubectl port-forward svc/frontend 30080:80
 │
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ KUBERNETES CLUSTER (Minikube)        namespace: project-2       │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Frontend Service (NodePort)                            │     │
│  │ Type: NodePort   │ Port: 80   │ NodePort: 30080        │     │
│  └───────────────────────────┬────────────────────────────┘     │
│                              │                                  │
│                              ▼                                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Frontend Pod (nginx:alpine)                            │     │
│  │ ├── Serves index.html from ConfigMap volume mount      │     │
│  │ └── Proxies GET /api → http://backend:5000             │     │
│  └───────────────────────────┬────────────────────────────┘     │
│                              │ ClusterIP DNS                    │
│                              │ http://backend:5000              │
│                              ▼                                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Backend Service (ClusterIP)                            │     │
│  │ Type: ClusterIP  │ IP: 10.98.75.247  │ Port: 5000      │     │
│  │ Selector: app=backend                                  │     │
│  └──────────────┬─────────────────────┬───────────────────┘     │
│                 │                     │                         │
│                 ▼                     ▼                         │
│  ┌─────────────────────┐   ┌─────────────────────┐              │
│  │ Backend Pod 1       │   │ Backend Pod 2       │              │
│  │ node:18-alpine      │   │ node:18-alpine      │              │
│  │ Reads env vars      │   │ Reads env vars      │              │
│  │ from ConfigMap      │   │ from ConfigMap      │              │
│  └─────────────────────┘   └─────────────────────┘              │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ backend-config (ConfigMap)                             │     │
│  │ APP_MESSAGE           APP_ENV          APP_VERSION     │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ frontend-config (ConfigMap)                            │     │
│  │ index.html                         nginx.conf          │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### Service Type Decision

| Component | Service Type | Reason                                        |
|-----------|--------------|-----------------------------------------------|
| Backend   | ClusterIP    | Only reachable by frontend pods (internal)    |
| Frontend  | NodePort     | Must be reachable from outside the cluster    |

### Request Flow

| Step | From | To | Method |
|------|------|----|--------|
| 1 | Browser | Frontend Service | NodePort 30080 |
| 2 | Frontend Service | Nginx Pod | Port 80 |
| 3 | Nginx Pod (`/api`) | Backend Service | ClusterIP DNS `http://backend:5000` |
| 4 | Backend Service | Backend Pod | Load balanced across replicas |
| 5 | Backend Pod | Nginx Pod | JSON response |
| 6 | Nginx Pod | Browser | Proxied JSON response |


## 3. Concepts Learned

### 3.1 ClusterIP Service

* Internal-only service — not reachable from outside the cluster
* Provides a **stable virtual IP** and **DNS name** inside the cluster
* DNS name equals the service `metadata.name`

| DNS Format | Example |
|------------|---------|
| Short name (same namespace) | `http://backend:5000` |
| Namespace-qualified | `http://backend.project-2:5000` |
| Fully qualified | `http://backend.project-2.svc.cluster.local:5000` |

* Label selector dynamically routes to matching healthy pods
* Automatically load-balances across all matching pod replicas

### 3.2 Namespace Isolation

* Logical separation of resources within the same cluster
* Prevents name collisions between projects
* Scoped DNS: short names work within the same namespace
* **Must be created before** any resources that reference it
* Easy bulk cleanup: delete namespace → all resources inside are deleted
* `kube-root-ca.crt` ConfigMap is auto-created in every namespace by Kubernetes — do not delete it

### 3.3 ConfigMap

* Stores non-sensitive configuration data separate from container images
* Two usage patterns:

| Pattern | How | Used For |
|---------|-----|----------|
| Volume mount | Files injected into container filesystem | HTML, nginx.conf |
| Environment variable | Key-value pairs as env vars | APP_MESSAGE, APP_ENV |

* Changes require pod restart to take effect (`kubectl rollout restart`)
* No Docker image rebuild needed when configuration changes

### 3.4 Environment Variables

* Injected into containers at runtime by Kubernetes
* Three sources:

| Source | YAML keyword | Use case |
|--------|-------------|----------|
| Direct value | `value:` | Simple static values |
| From ConfigMap | `configMapKeyRef` | Non-sensitive config |
| From Secret | `secretKeyRef` | Passwords, tokens, API keys |

* Read in Node.js via `process.env.VARIABLE_NAME`
* Fallback pattern: `process.env.VAR || 'default'`
* `HOSTNAME` is automatically set by Kubernetes to the pod name

### 3.5 Nginx Reverse Proxy

* Serves static files (HTML/CSS/JS) to the browser
* Proxies `/api` requests internally to the backend ClusterIP service
* Solves the browser DNS problem:

| Actor | Location | Can resolve cluster DNS? |
|-------|----------|--------------------------|
| Browser JavaScript | Your laptop | ❌ No |
| Nginx pod | Inside cluster | ✅ Yes |

### 3.6 Volume Mounts

Analogous to Linux mount points:

| Linux concept | Kubernetes equivalent |
|---------------|----------------------|
| Source share (NFS export) | `volumes:` block (defines source) |
| Mount point (`/mnt/data`) | `volumeMounts:` block (defines destination) |
| `mount` command | Shared `name:` field linking both |

* ConfigMap keys become filenames inside the mounted directory

### 3.7 Dependency Ordering

Always apply resources in this order:

1. Namespace ← everything else lives inside it
2. ConfigMaps ← pods depend on these
3. Secrets ← pods depend on these
4. Deployments ← depends on ConfigMaps + Secrets
5. Services ← depends on Deployments existing

> ⚠️ Wrong order → `CreateContainerConfigError`

### 3.8 Troubleshooting Pattern

* **Pod won't start?**
  * `kubectl describe pod <name> -n <namespace>`
  * Read Events section → identifies root cause
* **Pod running but wrong behaviour?**
  * `kubectl logs <pod> -n <namespace>`
  * `kubectl exec <pod> -n <namespace> -- env | grep <VAR>`


## 4. Prerequisites

* Minikube running with at least 2 worker nodes
* `kubectl` configured and pointing to Minikube
* Git configured with GitHub Personal Access Token in remote URL
* MobaXterm with SSH tunnel capability (for browser access)

### Verify Cluster Health

```bash
kubectl get nodes
```

Expected output:

```text
NAME           STATUS   ROLES           AGE
minikube       Ready    control-plane   Xd
minikube-m02   Ready    <none>          Xd
minikube-m03   Ready    <none>          Xd
```


## 5. Repository Structure

```text
kubernetes-learning/
├── README.md
├── project-01-static-website/
│   └── RUNBOOK.md
└── project-2-multitier/
    ├── RUNBOOK.md                  ← this file
    ├── namespace.yaml              ← project-2 namespace
    ├── backend-configmap.yaml      ← APP_MESSAGE, APP_ENV, APP_VERSION
    ├── backend-deployment.yaml     ← Node.js API, 2 replicas, env vars
    ├── backend-service.yaml        ← ClusterIP on port 5000
    ├── frontend-configmap.yaml     ← index.html + nginx.conf
    ├── frontend-deployment.yaml    ← nginx with ConfigMap volume mounts
    └── frontend-service.yaml       ← NodePort on port 30080
```


## 6. Deployment — Step by Step

> ⚠️ **Critical:** Always follow this order. ConfigMaps must be applied before Deployments. Namespace must exist before everything else.

### 6.1 Git Setup

```bash
git checkout main
git pull origin main
git checkout -b feature/<name>
git branch
```

### 6.2 Create the Namespace

```bash
kubectl apply -f project-2-multitier/namespace.yaml
kubectl get namespaces | grep project-2
```

Expected:

```text
project-2   Active   Xs
```

### 6.3 Apply the Backend ConfigMap

> ⚠️ Must be applied before the backend deployment.

```bash
kubectl apply -f project-2-multitier/backend-configmap.yaml
kubectl get configmap -n project-2
```

Expected:

```text
NAME               DATA   AGE
backend-config     3      Xs
kube-root-ca.crt   1      Xd
```

### 6.4 Apply the Backend Deployment

```bash
kubectl apply -f project-2-multitier/backend-deployment.yaml
kubectl get pods -n project-2 -w
```

Expected:

```text
NAME                      READY   STATUS    RESTARTS   AGE
backend-XXXXXXXXX-XXXXX   1/1     Running   0          Xs
backend-XXXXXXXXX-XXXXX   1/1     Running   0          Xs
```

### 6.5 Apply the Backend ClusterIP Service

```bash
kubectl apply -f project-2-multitier/backend-service.yaml
kubectl get service -n project-2
kubectl describe service backend -n project-2 | grep Endpoints
```

Expected:

```text
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
backend   ClusterIP   10.98.75.247   <none>        5000/TCP   Xs

Endpoints: 10.x.x.x:5000,10.x.x.x:5000
```

✅ Two endpoint IPs = label selector found both backend pods.

Inter-pod DNS test:

```bash
kubectl run curl-test \
  --image=curlimages/curl:latest \
  --rm --restart=Never \
  -n project-2 -it \
  -- curl http://backend:5000
```

Expected:

```text
{
  "message": "Hello from the Backend API - Configured via ConfigMap!",
  "environment": "development",
  "version": "2.0.0",
  "pod": "backend-XXXXXXXXX-XXXXX",
  "timestamp": "2026-XX-XXTXX:XX:XX.XXXZ"
}
```

### 6.6 Apply the Frontend ConfigMap

> ⚠️ Must be applied before the frontend deployment.

```bash
kubectl apply -f project-2-multitier/frontend-configmap.yaml
kubectl get configmap -n project-2
```

Expected:

```text
NAME               DATA   AGE
backend-config     3      Xm
frontend-config    2      Xs
kube-root-ca.crt   1      Xd
```

### 6.7 Apply the Frontend Deployment

```bash
kubectl apply -f project-2-multitier/frontend-deployment.yaml
kubectl get pods -n project-2
kubectl logs -n project-2 -l app=frontend
```

Expected logs (no errors):

```text
nginx: worker process started
nginx: worker process started
```

### 6.8 Apply the Frontend NodePort Service

```bash
kubectl apply -f project-2-multitier/frontend-service.yaml
kubectl get service -n project-2
```

Expected:

```text
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
backend    ClusterIP   10.98.75.247     <none>        5000/TCP       Xm
frontend   NodePort    10.106.149.119   <none>        80:30080/TCP   Xs
```


## 7. Verification

### All Pods Healthy

```bash
kubectl get pods -n project-2
```

Expected:

```text
NAME                       READY   STATUS    RESTARTS   AGE
backend-XXXXXXXXX-XXXXX    1/1     Running   0          Xm
backend-XXXXXXXXX-XXXXX    1/1     Running   0          Xm
frontend-XXXXXXXXX-XXXXX   1/1     Running   0          Xm
```

### Environment Variables Injected Correctly

```bash
kubectl exec -n project-2 <backend-pod-name> -- env | grep APP
```

Expected:

```text
APP_MESSAGE=Hello from the Backend API - Configured via ConfigMap!
APP_ENV=development
APP_VERSION=2.0.0
```

### Backend Logs Confirm Configuration

```bash
kubectl logs -n project-2 -l app=backend
```

Expected:

```text
Backend API v2.0.0 running on port 5000
Environment: development
Message configured as: Hello from the Backend API - Configured via ConfigMap!
```

### Load Balancing Verification

Run the curl test twice — the pod field in the JSON response should alternate between the two backend pod names, confirming ClusterIP is load balancing across replicas.


## 8. Accessing the Application

### Step 1 — Start Port-Forward

> ⚠️ Keep this terminal open. Port-forward stops when the terminal closes. Open a second terminal for all other commands.

```bash
kubectl port-forward \
  -n project-2 \
  svc/frontend \
  30080:80 \
  --address=0.0.0.0
```

Expected output:

```text
Forwarding from 0.0.0.0:30080 -> 80
Forwarding from [::]:30080 -> 80
```

### Step 2 — Configure MobaXterm SSH Tunnel

| Field | Value |
|-------|-------|
| Tunnel type | Local port forwarding |
| Local port | 30080 |
| Remote host | localhost |
| Remote port | 30080 |
| SSH server | Your RHEL sandbox IP |

### Step 3 — Open Browser

Navigate to `http://localhost:30080`

### Step 4 — Test the Application

Click the **Call Backend API** button.

Expected response fields:

| Field | Source |
|-------|--------|
| MESSAGE | `APP_MESSAGE` env var from `backend-config` ConfigMap |
| ENVIRONMENT | `APP_ENV` env var from `backend-config` ConfigMap |
| VERSION | `APP_VERSION` env var from `backend-config` ConfigMap |
| BACKEND POD | `HOSTNAME` auto-injected by Kubernetes |
| TIMESTAMP | Generated by backend Node.js at request time |

Environment badge colors:

| Value | Badge Color |
|-------|-------------|
| `development` | 🟠 Orange |
| `staging` | 🟣 Purple |
| `production` | 🟢 Green |

💡 **Load balancing test:** Click the button multiple times. The BACKEND POD field alternates between the two pod names — proving ClusterIP is distributing requests across replicas live.


## 9. Key Operations

### Update Application Configuration (No Image Rebuild)

```bash
kubectl edit configmap backend-config -n project-2
# Change APP_ENV: "development" → "production"
# Save and exit: :wq

kubectl rollout restart deployment/backend -n project-2
kubectl rollout status deployment/backend -n project-2
kubectl exec -n project-2 <pod-name> -- env | grep APP_ENV
```

### Update Frontend HTML or Nginx Config

```bash
kubectl apply -f project-2-multitier/frontend-configmap.yaml
kubectl rollout restart deployment/frontend -n project-2
kubectl rollout status deployment/frontend -n project-2
```

### Scale Backend Replicas

```bash
kubectl scale deployment/backend --replicas=3 -n project-2
kubectl get pods -n project-2
kubectl describe service backend -n project-2 | grep Endpoints
```

### Check Rollout History

```bash
kubectl rollout history deployment/backend -n project-2
kubectl rollout history deployment/frontend -n project-2
```

### Rollback a Deployment

```bash
kubectl rollout undo deployment/backend -n project-2
kubectl rollout status deployment/backend -n project-2
```

### View All Resources in Namespace

```bash
kubectl get all -n project-2
```


## 10. Troubleshooting

### Issue: CreateContainerConfigError

**Symptom:**

```text
NAME          READY   STATUS                       RESTARTS
backend-XXX   0/1     CreateContainerConfigError   X
```

**Cause:** A ConfigMap referenced by the deployment does not exist yet.

**Diagnose:**

```bash
kubectl describe pod <pod-name> -n project-2
# Read Events section:
# Error: configmap "backend-config" not found
```

**Fix:**

```bash
kubectl apply -f project-2-multitier/backend-configmap.yaml
kubectl get pods -n project-2 -w
```
*Pod self-heals automatically once the ConfigMap exists.*

### Issue: Service Endpoints Show `<none>`

**Cause:** Label selector on the service does not match labels on the pods.

**Diagnose:**

```bash
kubectl get service backend -n project-2 -o yaml | grep -A2 selector
kubectl get pods -n project-2 --show-labels
```

**Fix:** Ensure `spec.selector` in the service exactly matches `metadata.labels` in the pod template of the deployment.

### Issue: Frontend Shows Error on API Call

**Diagnose:**

```bash
kubectl exec -n project-2 <frontend-pod> \
  -- cat /etc/nginx/conf.d/default.conf

kubectl exec -n project-2 <frontend-pod> \
  -- wget -qO- http://backend:5000

kubectl get pods -n project-2 -l app=backend
```

### Issue: ConfigMap Update Not Reflected in Pod

**Cause:** ConfigMaps do not auto-update running pods.

**Fix:**

```bash
kubectl rollout restart deployment/<name> -n project-2
kubectl rollout status deployment/<name> -n project-2
```

### Issue: Old Errors Still Visible in kubectl describe

**Cause:** The Events section is a historical record, not live status. Events persist for approximately 1 hour regardless of current pod health.

**Resolution:** Always trust the STATUS column:

```bash
kubectl get pods -n project-2
# 1/1 Running = healthy, regardless of past events in describe
```


## 11. Cleanup

### Delete All Project-2 Resources (One Command)

```bash
kubectl delete namespace project-2
kubectl get all -n project-2
```
*Deleting the namespace removes everything inside it. Error from server (NotFound) after verify command is expected* ✅

### Delete Individual Resources (Selective)

```bash
kubectl delete -f project-2-multitier/frontend-service.yaml
kubectl delete -f project-2-multitier/frontend-deployment.yaml
kubectl delete -f project-2-multitier/frontend-configmap.yaml
kubectl delete -f project-2-multitier/backend-service.yaml
kubectl delete -f project-2-multitier/backend-deployment.yaml
kubectl delete -f project-2-multitier/backend-configmap.yaml
kubectl delete -f project-2-multitier/namespace.yaml
```

### Stop Port-Forward

```bash
Ctrl+C  # (in the terminal running port-forward)
```


## 12. Quick Reference Card

### Cluster Info

| Item | Value |
|------|-------|
| Namespace | `project-2` |
| Backend port | `5000` (ClusterIP, internal only) |
| Frontend port | `30080` (NodePort, external) |
| Backend DNS | `http://backend:5000` (from inside cluster) |
| Browser URL | `http://localhost:30080` (via port-forward + SSH tunnel) |

### ConfigMaps Summary

| Name | Keys | Consumed As |
|------|------|-------------|
| `backend-config` | `APP_MESSAGE`, `APP_ENV`, `APP_VERSION` | Environment variables |
| `frontend-config` | `index.html`, `nginx.conf` | Volume mounts |

### Most Used Commands

```bash
# View all resources in namespace
kubectl get all -n project-2

# Watch pods in real time
kubectl get pods -n project-2 -w

# View backend logs
kubectl logs -n project-2 -l app=backend

# View frontend logs
kubectl logs -n project-2 -l app=frontend

# Check env vars inside a pod
kubectl exec -n project-2 <pod> -- env | grep APP

# Inspect service endpoints
kubectl describe service backend -n project-2

# Live config update + rolling restart
kubectl edit configmap backend-config -n project-2
kubectl rollout restart deployment/backend -n project-2

# Restart frontend to pick up ConfigMap changes
kubectl rollout restart deployment/frontend -n project-2

# Inter-pod DNS test
kubectl run curl-test \
  --image=curlimages/curl:latest \
  --rm --restart=Never \
  -n project-2 -it \
  -- curl http://backend:5000
```

> **Note:** Runbook authored during Kubernetes hands-on learning — Project 2 of series.
> Environment: Minikube on RHEL 9.6 | Access: Windows VDI → MobaXterm → RHEL sandbox
