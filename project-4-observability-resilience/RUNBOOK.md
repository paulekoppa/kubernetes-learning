# Project 4 — Observability and Resilience


---


## Overview

This runbook documents Project 4 of the Kubernetes hands-on learning journey. The project builds a 2-tier application (Nginx + Node.js) and hardens it with production-grade observability and resilience patterns.


---


## What You Will Learn

* Liveness, Readiness, and Startup Probes with live failure simulations
* Resource Requests and Limits for all containers
* Metrics Server enabling `kubectl top` and HPA
* HorizontalPodAutoscaler with a real CPU stress test
* Live ConfigMap file reload versus application restart behaviour


---


## Architecture

```text
Browser
└── NodePort (30400)
    └── Nginx frontend (port 80)
        └── ClusterIP backend-service (port 3000)
            └── Node.js backend (port 3000)
```

### Traffic Flow

Browser → `frontend-service:30400` → Nginx pod:80 → `backend-service:3000` → Node.js pod:3000

### Directory Structure

```text
project-4-observability-resilience/
├── namespace.yaml
├── configmap.yaml
├── backend-deployment.yaml
├── frontend-deployment.yaml
├── hpa.yaml
└── RUNBOOK.md
```


---


## Prerequisites

* Minikube running with at least 2 worker nodes
* `kubectl` configured and pointing to minikube context
* Metrics Server addon enabled


---


## Environment

| Item | Value |
| --- | --- |
| OS | Red Hat Enterprise Linux 9.6 |
| Kubernetes | Minikube v1.35.1 |
| Nodes | 1 control plane + 2 workers |
| Namespace | `project-4` |
| Frontend URL | `http://192.168.49.2:30400` |
| Node.js version | 18 Alpine |
| Nginx version | Alpine |


---


## Step 1 — Git Setup

```bash
cd ~/kubernetes-learning
git checkout main
git pull origin main
git checkout -b feature/project-4-observability-resilience
mkdir project-4-observability-resilience
cd project-4-observability-resilience
```


---


## Step 2 — Namespace

### Create namespace.yaml

```bash
kubectl apply --dry-run=client -f namespace.yaml
kubectl apply -f namespace.yaml
kubectl get namespace project-4
```

### Set default namespace for session

```bash
kubectl config set-context --current --namespace=project-4
kubectl config view --minify | grep namespace
```


---


## Step 3 — ConfigMaps

Two ConfigMaps in one file separated by `---`

| ConfigMap | Purpose |
| --- | --- |
| `backend-code` | Holds `server.js` mounted as `/app/server.js` |
| `nginx-config` | Holds `nginx.conf` mounted as `/etc/nginx/nginx.conf` |

### Node.js Endpoints

| Endpoint | Purpose |
| --- | --- |
| `/` | Main app response — returns hostname and version |
| `/health/live` | Liveness probe target — always returns 200 |
| `/health/ready` | Readiness probe target — returns 200 or 503 based on isReady flag |
| `/toggleready` | Flips isReady flag — used for readiness probe simulation |
| `/crash` | Calls `process.exit(1)` — used for liveness probe simulation |
| `/load` | Burns CPU for 500ms — used for HPA stress test |

### Apply

```bash
kubectl apply --dry-run=client -f configmap.yaml
kubectl apply -f configmap.yaml
kubectl get configmaps
```


---


## Step 4 — Backend Deployment and Service

### Resources

| Setting | Value |
| --- | --- |
| Image | `node:18-alpine` |
| Command | `node /app/server.js` |
| Container port | `3000` |
| Service type | `ClusterIP` |
| Service port | `3000` |

### Apply

```bash
kubectl apply --dry-run=client -f backend-deployment.yaml
kubectl apply -f backend-deployment.yaml
kubectl get pods -l app=backend -w
kubectl logs -l app=backend
kubectl get service backend-service
```


---


## Step 5 — Frontend Deployment and Service

### Resources

| Setting | Value |
| --- | --- |
| Image | `nginx:alpine` |
| Container port | `80` |
| Service type | `NodePort` |
| NodePort | `30400` |
| Config mount | `/etc/nginx/nginx.conf` via subPath |

### Apply

```bash
kubectl apply --dry-run=client -f frontend-deployment.yaml
kubectl apply -f frontend-deployment.yaml
kubectl get pods -w
minikube service frontend-service -n project-4 --url
```

### Verify full stack

```bash
curl [http://192.168.49.2:30400/](http://192.168.49.2:30400/)
curl [http://192.168.49.2:30400/health/live](http://192.168.49.2:30400/health/live)
curl [http://192.168.49.2:30400/health/ready](http://192.168.49.2:30400/health/ready)
```


---


## Step 6 — Probes

### Probe Types

| Probe | Question Kubernetes asks | Action on failure |
| --- | --- | --- |
| Startup | Has the app finished starting? | Restart container |
| Liveness | Is the app still functional? | Restart container |
| Readiness | Should this pod receive traffic? | Remove from Service endpoints |

### Probe Configuration

**Backend**

| Probe | Endpoint | Period | Failure Threshold | Timeout |
| --- | --- | --- | --- | --- |
| Startup | `/health/live` | 5s | 12 (60s grace) | 3s |
| Liveness | `/health/live` | 10s | 3 | 3s |
| Readiness | `/health/ready` | 10s | 3 | 3s |

**Frontend**

| Probe | Endpoint | Period | Failure Threshold | Timeout |
| --- | --- | --- | --- | --- |
| Startup | `/nginx-health` | 5s | 6 (30s grace) | 3s |
| Liveness | `/nginx-health` | 15s | 3 | 3s |
| Readiness | `/nginx-health` | 10s | 3 | 3s |

### Verify probes registered

```bash
kubectl describe pod -l app=backend | grep -A 3 "Liveness\|Readiness\|Startup"
kubectl describe pod -l app=frontend | grep -A 3 "Liveness\|Readiness\|Startup"
```


---


## Step 7 — Probe Failure Simulations

### Simulation 1 — Readiness Probe Failure

Purpose: prove pod is removed from Service endpoints without being restarted.

Toggle readiness OFF:
```bash
curl [http://192.168.49.2:30400/toggleready](http://192.168.49.2:30400/toggleready)
```

Watch pod go to 0/1 (removed from endpoints — NOT restarted):
```bash
kubectl get pods -l app=backend -w
```

When pod is 0/1, Service blocks access — use exec to toggle back:
```bash
kubectl exec -it <pod-name> -- wget -qO- http://localhost:3000/toggleready
```

Confirm recovery — RESTARTS must still be 0:
```bash
kubectl get pods -l app=backend
```

### Key lesson — Readiness simulation

The pod STATUS stays Running throughout. Only READY changes from 1/1 to 0/1. RESTARTS stays at 0. The pod was never killed — just removed from rotation. When the pod is 0/1 you cannot reach it through the Service. Use `kubectl exec` to bypass the Service and talk directly to the pod.

### Simulation 2 — Liveness Probe Failure

Purpose: prove Kubernetes automatically restarts a crashed container.

Crash the Node.js process intentionally:
```bash
curl [http://192.168.49.2:30400/crash](http://192.168.49.2:30400/crash)
```

Watch the restart happen in real time:
```bash
kubectl get pods -l app=backend -w
```

Confirm exit code 1 was recorded:
```bash
kubectl describe pod -l app=backend | grep -A 10 "Last State"
```

Confirm app recovered — RESTARTS now shows 1:
```bash
kubectl get pods -l app=backend
```

### Key lesson — Liveness simulation

Exit Code 1 proves `process.exit(1)` was the cause. RESTARTS increments to 1 — Kubernetes detected and fixed it automatically. Zero human intervention required for recovery.


---


## Step 8 — Resource Requests and Limits

### Why resources are required for HPA

HPA calculates CPU usage as a percentage of the request value. Without a request defined HPA has no baseline and will not scale.

### Values applied

| Container | CPU Request | CPU Limit | Memory Request | Memory Limit |
| --- | --- | --- | --- | --- |
| backend | 100m | 300m | 128Mi | 256Mi |
| frontend | 50m | 150m | 64Mi | 128Mi |

### CPU and Memory units

| Unit | Meaning |
| --- | --- |
| `100m` | 100 millicores = 0.1 CPU core |
| `300m` | 300 millicores = 0.3 CPU core |
| `128Mi` | 128 Mebibytes |
| `256Mi` | 256 Mebibytes |

### Behaviour on limit exceeded

| Resource | Behaviour |
| --- | --- |
| CPU limit exceeded | Container throttled — slowed down, not killed |
| Memory limit exceeded | Container OOMKilled — killed and restarted |

### Verify

```bash
kubectl describe pod -l app=backend | grep -A 6 "Limits\|Requests"
kubectl describe pod -l app=frontend | grep -A 6 "Limits\|Requests"
```


---


## Step 9 — Metrics Server

### What the Metrics Server does

Collects CPU and memory usage from every node and pod. Exposes the data through the Kubernetes API. Powers both `kubectl top` and HPA decisions.

### Enable

```bash
minikube addons enable metrics-server
kubectl get pods -n kube-system | grep metrics-server
```

### Verify

```bash
kubectl top nodes
kubectl top pods
```

### Note

If `kubectl top` returns "Metrics API not available" wait 60 seconds. The Metrics Server needs a few collection cycles to stabilise.


---


## Step 10 — HorizontalPodAutoscaler

### How HPA works

Every 15 seconds HPA queries the Metrics Server for average CPU usage across all backend pods. It compares actual usage against the target percentage of the CPU request value.

Scale up formula:

`desired replicas = ceil( current replicas x ( current CPU% / target CPU% ) )`

Example: 1 pod at 200% of request → `ceil( 1 x 200/50 )` = 4 pods

Scale down has a 5 minute stabilization window to prevent flapping.

### Configuration

| Parameter | Value | Reason |
| --- | --- | --- |
| `minReplicas` | 1 | Always keep at least one pod running |
| `maxReplicas` | 5 | Protect cluster from unbounded scaling |
| `targetCPUUtilizationPercentage` | 50% | Scale before pods are overwhelmed |

### Apply

```bash
kubectl apply --dry-run=client -f hpa.yaml
kubectl apply -f hpa.yaml
kubectl get hpa
kubectl describe hpa backend-hpa
```

### Reading the TARGETS column

`cpu: 1%/50%` means current average CPU is 1% of request, target is 50%


---


## Step 11 — HPA Stress Test

### Setup — open three terminals

Terminal 1 watches HPA decisions:
```bash
kubectl get hpa backend-hpa --watch
```

Terminal 2 watches pods scaling:
```bash
kubectl get pods -l app=backend --watch
```

Terminal 3 runs the load generator:
```bash
kubectl run load-generator \
  --image=busybox:1.28 \
  --restart=Never \
  --rm \
  -it \
  -- /bin/sh -c "while true; do wget -q -O- http://backend-service:3000/load > /dev/null; done"
```

### Stop the load

Press `Ctrl+C` in Terminal 3 to stop the load generator. The pod is automatically deleted because of `--rm`. HPA will scale back down to 1 replica after the 5 minute stabilization window.

### View full scaling history

```bash
kubectl describe hpa backend-hpa | tail -30
```

### Expected Events output

```text
Normal SuccessfulRescale horizontal-pod-autoscaler New size: 5; reason: cpu resource utilization above target
Normal SuccessfulRescale horizontal-pod-autoscaler New size: 1; reason: All metrics below target
```


---


## Step 12 — ConfigMap Live Reload

### How ConfigMap volume updates work

When a ConfigMap is mounted as a volume Kubernetes automatically updates the file on disk inside the container within 30 to 60 seconds. The running application process is not restarted.

### Linux analogy

| Linux world | Kubernetes world |
| --- | --- |
| Edit `/etc/app/config.conf` | `kubectl apply -f configmap.yaml` |
| `systemctl restart app` | `kubectl rollout restart deployment/backend` |
| `systemctl reload nginx` | `kubectl exec -- nginx -s reload` |

### Demonstration steps

**Step 1** — Apply updated ConfigMap (version 1.0.0 → 2.0.0)
```bash
kubectl apply -f configmap.yaml
```

**Step 2** — Confirm app still serves old version (in-memory process unchanged)
```bash
curl [http://192.168.49.2:30400/](http://192.168.49.2:30400/)
```

**Step 3** — Watch the file update on disk inside the container
```bash
kubectl exec -it <pod-name> \
  -- /bin/sh -c "while true; do grep version /app/server.js; sleep 5; done"
```

**Step 4** — File shows 2.0.0 on disk but curl still returns 1.0.0 (Node.js read server.js once at startup — does not watch for changes)

**Step 5** — Rolling restart to load the new code into memory
```bash
kubectl rollout restart deployment/backend
kubectl get pods -l app=backend -w
```

**Step 6** — Confirm app now serves new version
```bash
curl [http://192.168.49.2:30400/](http://192.168.49.2:30400/)
```

### Key lesson

Kubernetes updates the file on disk automatically. The running process only picks up the change after a restart. Applications with file watchers or config polling can reload without restart. Node.js loading code at startup always requires a rollout restart.


---


## Useful Commands Reference

### General health check

```bash
kubectl get all -n project-4
kubectl get pods -w
kubectl top pods
kubectl top nodes
```

### Probe inspection

```bash
kubectl describe pod -l app=backend | grep -A 3 "Liveness\|Readiness\|Startup"
kubectl describe pod -l app=frontend | grep -A 3 "Liveness\|Readiness\|Startup"
```

### HPA inspection

```bash
kubectl get hpa
kubectl describe hpa backend-hpa
kubectl get hpa backend-hpa --watch
```

### Logs

```bash
kubectl logs -l app=backend
kubectl logs -l app=frontend
kubectl logs -l app=backend --previous
```

### Exec into pod

```bash
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- wget -qO- http://localhost:3000/health/live
```

### Rollout commands

```bash
kubectl rollout restart deployment/backend
kubectl rollout restart deployment/frontend
kubectl rollout status deployment/backend
kubectl rollout history deployment/backend
```

### Cleanup

```bash
kubectl delete namespace project-4
kubectl config set-context --current --namespace=default
```


---


## Key Concepts Summary

| Concept | What it does | Why it matters |
| --- | --- | --- |
| Liveness Probe | Restarts container when app is unhealthy | Self-healing — no manual intervention needed |
| Readiness Probe | Removes pod from Service when not ready | Zero-downtime deployments and safe rollouts |
| Startup Probe | Protects slow-starting apps from premature kills | Prevents liveness from killing app during boot |
| Resource Requests | Guaranteed minimum CPU and memory | Scheduler places pods on nodes with room |
| Resource Limits | Hard ceiling on CPU and memory | Prevents noisy neighbour problem |
| Metrics Server | Collects real-time CPU and memory data | Powers HPA and `kubectl top` |
| HPA | Auto-scales replica count based on CPU | Handles load automatically without human action |
| ConfigMap Volume | Mounts config as files inside container | Decouples configuration from container image |


---


## Lessons Learned

### Readiness probe blocks Service access

When a pod is 0/1 (removed from endpoints) you cannot reach it through the Service. Use `kubectl exec` to bypass the Service and talk directly to the pod on localhost.

### HPA scale-down is intentionally slow

The 5 minute stabilization window prevents flapping. Scale-up is fast and aggressive. Scale-down is deliberate and gradual. This is by design — scaling down too fast wastes the work of scaling up.

### ConfigMap file update vs process reload

`kubectl apply` updates the file on disk within 30 to 60 seconds. The running process is not affected until it is restarted. This mirrors the Linux pattern of editing a config file versus restarting the service that reads it.

### kill -9 PID 1 may not work in all container runtimes

In Minikube with containerd the container runtime can intercept signals sent to PID 1 from outside. Use `process.exit()` from within the application or a `/crash` endpoint for reliable crash simulation.

### Resources must be set before HPA

HPA calculates CPU as a percentage of the request value. If no request is defined HPA cannot calculate utilisation and will report unknown in the TARGETS column.

> **Note:** Runbook authored during Kubernetes hands-on learning — Project 4 of series. Environment: Minikube on RHEL 9.6 | Access: Windows VDI → MobaXterm → RHEL sandbox
