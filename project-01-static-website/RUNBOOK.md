# Project 01 — Static Website Deployment Runbook 📘

**Environment:** RHEL 9.6 | Minikube v1.35.1 | MobaXterm (Windows VDI)  
**Concepts:** Pods, Deployments, Services, kubectl basics, Rolling Updates

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [GitHub Setup](#2-github-setup)
3. [Local Git Setup](#3-local-git-setup)
4. [Create Kubernetes Manifests](#4-create-kubernetes-manifests)
5. [Deploy to Kubernetes](#5-deploy-to-kubernetes)
6. [Verify & Access](#6-verify--access)
7. [Experiments](#7-experiments)
8. [Git Commit & Push](#8-git-commit--push)
9. [Cleanup](#9-cleanup)
10. [Key Concepts Summary](#10-key-concepts-summary)

---

## 1. Prerequisites

Verify your cluster is healthy:

```bash
kubectl get nodes
```

Expected:

```
NAME           STATUS   ROLES           AGE   VERSION
minikube       Ready    control-plane   Xd    v1.X.X
minikube-m02   Ready    <none>          Xd    v1.X.X
minikube-m03   Ready    <none>          Xd    v1.X.X
```

```bash
minikube status
```

Expected:

```
minikube: Running
cluster: Running
kubectl: Correctly Configured
```

---

## 2. GitHub Setup

> ⚠️ Do this once — before any other step.

### 2.1 Create the GitHub Repository

1. Go to [https://github.com](https://github.com) and log in
2. Click **"+"** → **"New repository"**
3. Fill in:

```
Repository name:  kubernetes-learning
Visibility:       Public (or Private)
Initialize with:  ✅ Add a README file   ← IMPORTANT
```

4. Click **"Create repository"**

> 💡 Initializing with a README creates the first commit and the `main` branch automatically. This avoids branch confusion later.

### 2.2 Create a Personal Access Token (PAT)

GitHub no longer accepts passwords for Git operations — you need a token.

1. Go to GitHub → top-right avatar → **Settings**
2. Scroll down → click **"Developer settings"** (bottom of left sidebar)
3. Click **"Personal access tokens"** → **"Tokens (classic)"**
4. Click **"Generate new token (classic)"**
5. Fill in:

```
Note:        kubernetes-learning
Expiration:  90 days (or your preference)
Scopes:      ✅ repo  (check the top-level box — selects all sub-scopes)
```

6. Click **"Generate token"**
7. **Copy the token immediately — you will NOT see it again!**

```bash
# Save it somewhere safe temporarily, e.g.:
ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

## 3. Local Git Setup

### 3.1 Clone the repository

```bash
cd ~
git clone https://github.com/<YOUR_USERNAME>/kubernetes-learning.git
cd kubernetes-learning
```

### 3.2 Configure Git credentials

```bash
# Store credentials so you're not prompted every time
git config --global credential.helper store

# Set your Git identity (if not already set)
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"

# Verify
git config --global --list
```

### 3.3 Embed your token in the remote URL

This is required for non-interactive authentication on a remote server.

```bash
git remote set-url origin https://<YOUR_USERNAME>:<YOUR_TOKEN>@github.com/<YOUR_USERNAME>/kubernetes-learning.git

# Example:
# git remote set-url origin https://paulekoppa:ghp_xxxxxxxxxxxx@github.com/paulekoppa/kubernetes-learning.git

# Verify
git remote -v
```

Expected:

```
origin  https://paulekoppa:ghp_xxxx...@github.com/paulekoppa/kubernetes-learning.git (fetch)
origin  https://paulekoppa:ghp_xxxx...@github.com/paulekoppa/kubernetes-learning.git (push)
```

### 3.4 Create your feature branch

```bash
# Always start from an up-to-date main
git checkout main
git pull origin main

# Create and switch to feature branch
git checkout -b feature/project-01-static-website

# Verify
git branch
```

Expected:

```
* feature/project-01-static-website
  main
```

---

## 4. Create Kubernetes Manifests

### 4.1 Create the project folder

```bash
mkdir -p project-01-static-website
cd project-01-static-website
```

### 4.2 Create deployment.yaml

```bash
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1          # API group that handles Deployments
kind: Deployment             # Resource type
metadata:
  name: static-website       # Name of this Deployment
  labels:
    app: static-website      # Label on the Deployment itself
spec:
  replicas: 3                # Keep 3 Pods running at all times
  selector:
    matchLabels:
      app: static-website    # Manages Pods that have THIS label
  template:                  # Blueprint for each Pod
    metadata:
      labels:
        app: static-website  # MUST match selector.matchLabels above
    spec:
      containers:
      - name: nginx
        image: nginx:1.25    # Always pin a version — never use 'latest'
        ports:
        - containerPort: 80
        resources:           # Always set resource limits (best practice)
          requests:
            memory: "64Mi"
            cpu: "100m"      # 100 millicores = 0.1 CPU
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

### 4.3 Create service.yaml

```bash
cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: static-website-svc
spec:
  type: NodePort             # Exposes the service on each Node's IP
  selector:
    app: static-website      # Routes traffic to Pods with this label
  ports:
  - port: 80                 # Port the Service listens on (inside cluster)
    targetPort: 80           # Port on the Pod to forward traffic to
    nodePort: 30080          # Port on the Node (range: 30000-32767)
EOF
```

### 4.4 Create README.md

```bash
cat > README.md << 'EOF'
# Project 01 - Static Website Deployment

Deploys a 3-replica nginx static website using a Kubernetes Deployment
and NodePort Service.

## Apply

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

## Access

```bash
# Port forward to access locally
kubectl port-forward svc/static-website-svc 8080:80 &

# Test
curl http://localhost:8080
```

## Concepts Covered

- Deployments and ReplicaSets
- NodePort Services
- Label selectors
- Self-healing
- Rolling updates
EOF
```

### 4.5 Verify all files exist

```bash
ls -la
```

Expected:

```
deployment.yaml
service.yaml
README.md
```

---

## 5. Deploy to Kubernetes

```bash
# Go back to repo root
cd ~/kubernetes-learning

# Apply both manifests
kubectl apply -f project-01-static-website/deployment.yaml
kubectl apply -f project-01-static-website/service.yaml
```

Expected:

```
deployment.apps/static-website created
service/static-website-svc created
```

Watch Pods come up:

```bash
kubectl get pods -w
```

Expected (wait until all show `Running`):

```
NAME                              READY   STATUS    RESTARTS   AGE
static-website-7875647d9f-8922m   1/1     Running   0          30s
static-website-7875647d9f-qk65v   1/1     Running   0          30s
static-website-7875647d9f-vdk2z   1/1     Running   0          30s
```

> Press `Ctrl+C` to exit the watch.

---

## 6. Verify & Access

### Inspect the deployment

```bash
kubectl describe deployment static-website

# See which Node each Pod landed on
kubectl get pods -o wide

# Check the service
kubectl get svc static-website-svc
```

### Access the website via port-forward

Since the Minikube IP (`192.168.49.2`) is not reachable from MobaXterm,
we use `port-forward` to tunnel traffic through localhost.

#### Option A — Background process (single terminal)

```bash
# Start port-forward in background
kubectl port-forward svc/static-website-svc 8080:80 &

# Test
curl http://localhost:8080

# Check HTTP status code only
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
# Expected: 200

# When done, kill the background port-forward
kill %1
```

#### Option B — Two terminals

```bash
# Terminal 1: start and leave running
kubectl port-forward svc/static-website-svc 8080:80

# Terminal 2: test
curl http://localhost:8080
```

---

## 7. Experiments

### 7.1 Self-Healing Test

```bash
# Get a pod name
kubectl get pods

# Delete one pod — Kubernetes should immediately recreate it
kubectl delete pod <POD_NAME>

# Watch it respawn (should be back within seconds)
kubectl get pods -w
```

> 💡 This proves the Deployment controller maintains desired state.

### 7.2 Scale Up and Down

```bash
# Scale to 5 replicas
kubectl scale deployment static-website --replicas=5
kubectl get pods -w

# Scale back down to 3
kubectl scale deployment static-website --replicas=3
kubectl get pods -w
```

### 7.3 Rolling Update (Zero Downtime)

```bash
# Update nginx version — pods are replaced one by one
kubectl set image deployment/static-website nginx=nginx:1.26

# Watch the rolling update
kubectl rollout status deployment/static-website

# View rollout history
kubectl rollout history deployment/static-website

# Rollback to previous version if needed
kubectl rollout undo deployment/static-website
```

---

## 8. Git Commit & Push

```bash
cd ~/kubernetes-learning

# Commit each file separately with meaningful messages
git add project-01-static-website/deployment.yaml
git commit -m "feat(p01): add nginx deployment with 3 replicas"

git add project-01-static-website/service.yaml
git commit -m "feat(p01): add NodePort service on port 30080"

git add project-01-static-website/README.md
git commit -m "docs(p01): add README with apply and access instructions"

# Verify commits
git log --oneline
```

### Push to GitHub

```bash
git push -u origin feature/project-01-static-website
```

### Open Pull Request on GitHub

1. Go to `https://github.com/<YOUR_USERNAME>/kubernetes-learning`
2. Click the **"Compare & pull request"** banner
3. Fill in:

```
Base:    main
Compare: feature/project-01-static-website
Title:   feat: Project 01 - Static Website Deployment

Description:
## What this PR does
- Deploys a 3-replica nginx static website
- Exposes it via NodePort Service on port 30080

## Manifests added
- deployment.yaml
- service.yaml
- README.md

## Tested
- [x] kubectl apply works cleanly
- [x] All 3 pods Running across nodes
- [x] curl http://localhost:8080 returns 200
- [x] Self-healing verified (pod deletion test)
- [x] Scaling verified (3 → 5 → 3 replicas)
- [x] Rolling update tested (nginx 1.25 → 1.26)
- [x] Rollback tested
```

4. Click **"Create pull request"**
5. Click **"Merge pull request"** → **"Confirm merge"**
6. Click **"Delete branch"**

### Sync local main

```bash
git checkout main
git pull origin main
git branch -d feature/project-01-static-website

# Verify
git branch
git log --oneline
```

---

## 9. Cleanup

```bash
kubectl delete -f project-01-static-website/

# Verify everything is gone
kubectl get deployments
kubectl get svc
kubectl get pods
```

---

## 10. Key Concepts Summary

| CONCEPT | WHAT IT DOES | WHY IT MATTERS |
|---|---|---|
| Deployment | Declares desired state (e.g. 3 replicas) | Self-healing, rolling updates |
| ReplicaSet | Created by Deployment — maintains Pod count | You rarely interact with it directly |
| Pod | Smallest deployable unit — runs your container | Ephemeral — never rely on a single Pod |
| Service | Stable network endpoint for Pods | Pods have dynamic IPs — Service abstracts that |
| NodePort | Exposes Service on a port on every Node | Simple external access for dev/testing |
| Label Selector | Links Services and Deployments to Pods | The glue that connects all resources |
| Resource Limits | CPU/memory boundaries per container | Prevents one Pod from starving others |
| Rolling Update | Replaces Pods gradually, not all at once | Zero downtime deployments |
| port-forward | Tunnels cluster traffic to localhost | Dev/debug tool — not for production |

---

## ✅ Project 1 Completion Checklist

- [ ] GitHub repo created with README (`main` branch initialized)
- [ ] PAT token created with `repo` scope
- [ ] Remote URL configured with token
- [ ] Feature branch created from `main`
- [ ] `deployment.yaml` applied successfully
- [ ] `service.yaml` applied successfully
- [ ] All 3 pods `Running` across 3 nodes
- [ ] `curl http://localhost:8080` returns nginx page
- [ ] Self-healing test passed
- [ ] Scaling test passed (3 → 5 → 3)
- [ ] Rolling update tested (nginx 1.25 → 1.26)
- [ ] Rollback tested
- [ ] All files committed with conventional commit messages
- [ ] PR merged into `main`
- [ ] Local feature branch deleted
- [ ] `kubectl delete` cleanup done
