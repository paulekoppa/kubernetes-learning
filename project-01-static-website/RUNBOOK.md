# Project 01 — Static Website Deployment Runbook 📘

> **Environment:** RHEL 9.6 | Minikube v1.35.1 | MobaXterm (Windows VDI)
> **Concepts:** Pods, Deployments, Services, kubectl basics, Rolling Updates

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
11. [Project Completion Checklist](#11-project-completion-checklist)

---

## 1. Prerequisites

### Verify your cluster is healthy

\`\`\`bash
kubectl get nodes
\`\`\`

Expected output:

\`\`\`
NAME           STATUS   ROLES           AGE   VERSION
minikube       Ready    control-plane   Xd    v1.X.X
minikube-m02   Ready    <none>          Xd    v1.X.X
minikube-m03   Ready    <none>          Xd    v1.X.X
\`\`\`

\`\`\`bash
minikube status
\`\`\`

Expected output:

\`\`\`
minikube: Running
cluster: Running
kubectl: Correctly Configured
\`\`\`

---

## 2. GitHub Setup

> ⚠️ Do this once — before any other step.

### 2.1 Create the GitHub Repository

1. Go to https://github.com and log in
2. Click **"+"** → **"New repository"**
3. Fill in:

| Field | Value |
|-------|-------|
| Repository name | kubernetes-learning |
| Visibility | Public or Private |
| Initialize with | ✅ Add a README file |

4. Click **"Create repository"**

> 💡 Initializing with a README creates the first commit and the main branch
> automatically. This avoids branch confusion later.

---

### 2.2 Create a Personal Access Token (PAT)

> GitHub no longer accepts passwords for Git operations — you need a token.

1. Go to **GitHub → top-right avatar → Settings**
2. Scroll down → click **"Developer settings"** (bottom of left sidebar)
3. Click **"Personal access tokens"** → **"Tokens (classic)"**
4. Click **"Generate new token (classic)"**
5. Fill in:

| Field | Value |
|-------|-------|
| Note | kubernetes-learning |
| Expiration | 90 days (or your preference) |
| Scopes | ✅ repo (check top-level box) |

6. Click **"Generate token"**
7. **Copy the token immediately — you will NOT see it again!**

> 🔐 Save it somewhere safe temporarily.

---

## 3. Local Git Setup

### 3.1 Clone the repository

\`\`\`bash
cd ~
git clone https://github.com/<YOUR_USERNAME>/kubernetes-learning.git
cd kubernetes-learning
\`\`\`

### 3.2 Configure Git credentials

\`\`\`bash
# Store credentials so you are not prompted every time
git config --global credential.helper store

# Set your Git identity (if not already set)
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"

# Verify
git config --global --list
\`\`\`

### 3.3 Embed your token in the remote URL

\`\`\`bash
git remote set-url origin https://<YOUR_USERNAME>:<YOUR_TOKEN>@github.com/<YOUR_USERNAME>/kubernetes-learning.git

# Verify
git remote -v
\`\`\`

### 3.4 Create your feature branch

\`\`\`bash
git checkout main
git pull origin main
git checkout -b feature/project-01-static-website
git branch
\`\`\`

---

## 4. Create Kubernetes Manifests

### 4.1 Create the project folder

\`\`\`bash
mkdir -p project-01-static-website
cd project-01-static-website
\`\`\`

### 4.2 Create deployment.yaml

\`\`\`bash
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-website
  labels:
    app: static-website
spec:
  replicas: 3
  selector:
    matchLabels:
      app: static-website
  template:
    metadata:
      labels:
        app: static-website
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
\`\`\`

### 4.3 Create service.yaml

\`\`\`bash
cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: static-website-svc
spec:
  type: NodePort
  selector:
    app: static-website
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
EOF
\`\`\`

### 4.4 Verify all files exist

\`\`\`bash
ls -la
\`\`\`

---

## 5. Deploy to Kubernetes

\`\`\`bash
cd ~/kubernetes-learning

kubectl apply -f project-01-static-website/deployment.yaml
kubectl apply -f project-01-static-website/service.yaml

# Watch Pods come up
kubectl get pods -w
\`\`\`

Expected output:

\`\`\`
NAME                              READY   STATUS    RESTARTS   AGE
static-website-7875647d9f-8922m   1/1     Running   0          30s
static-website-7875647d9f-qk65v   1/1     Running   0          30s
static-website-7875647d9f-vdk2z   1/1     Running   0          30s
\`\`\`

---

## 6. Verify & Access

### Inspect resources

\`\`\`bash
kubectl describe deployment static-website
kubectl get pods -o wide
kubectl get svc static-website-svc
\`\`\`

### Access via port-forward

\`\`\`bash
# Start port-forward in background
kubectl port-forward svc/static-website-svc 8080:80 &

# Test
curl http://localhost:8080

# Check HTTP status code
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080

# Kill port-forward when done
kill %1
\`\`\`

---

## 7. Experiments

### 7.1 Self-Healing Test

\`\`\`bash
kubectl get pods
kubectl delete pod <POD_NAME>
kubectl get pods -w
\`\`\`

### 7.2 Scaling Test

\`\`\`bash
kubectl scale deployment static-website --replicas=5
kubectl get pods -w
kubectl scale deployment static-website --replicas=3
\`\`\`

### 7.3 Rolling Update Test

\`\`\`bash
kubectl set image deployment/static-website nginx=nginx:1.26
kubectl rollout status deployment/static-website
kubectl rollout history deployment/static-website
kubectl rollout undo deployment/static-website
\`\`\`

---

## 8. Git Commit & Push

\`\`\`bash
cd ~/kubernetes-learning

git add project-01-static-website/deployment.yaml
git commit -m "feat(p01): add nginx deployment with 3 replicas"

git add project-01-static-website/service.yaml
git commit -m "feat(p01): add NodePort service on port 30080"

git add project-01-static-website/README.md
git commit -m "docs(p01): add README with apply and access instructions"

git add project-01-static-website/RUNBOOK.md
git commit -m "docs(p01): add full runbook with setup and experiments"

git push -u origin feature/project-01-static-website
\`\`\`

### Pull Request Checklist

| Step | Action |
|------|--------|
| Base | main |
| Compare | feature/project-01-static-website |
| Title | feat: Project 01 - Static Website Deployment |
| Merge | Merge pull request → Confirm merge |
| Cleanup | Delete branch after merge |

### Sync local main

\`\`\`bash
git checkout main
git pull origin main
git branch -d feature/project-01-static-website
\`\`\`

---

## 9. Cleanup

\`\`\`bash
kubectl delete -f project-01-static-website/

kubectl get deployments
kubectl get svc
kubectl get pods
\`\`\`

---

## 10. Key Concepts Summary

| Concept | What it does | Why it matters |
|---------|-------------|----------------|
| **Deployment** | Declares desired state (e.g. 3 replicas) | Self-healing, rolling updates |
| **ReplicaSet** | Created by Deployment, maintains Pod count | You rarely interact with it directly |
| **Pod** | Smallest deployable unit — runs your container | Ephemeral — never rely on a single Pod |
| **Service** | Stable network endpoint for Pods | Pods have dynamic IPs — Service abstracts that |
| **NodePort** | Exposes Service on a port on every Node | Simple external access for dev/testing |
| **Label Selector** | Links Services and Deployments to Pods | The glue that connects all resources |
| **Resource Limits** | CPU/memory boundaries per container | Prevents one Pod from starving others |
| **Rolling Update** | Replaces Pods gradually, not all at once | Zero downtime deployments |
| **port-forward** | Tunnels cluster traffic to localhost | Dev/debug tool — not for production |

---

## 11. Project Completion Checklist

- [ ] GitHub repo created with README (main branch initialized)
- [ ] PAT token created with repo scope
- [ ] Remote URL configured with token
- [ ] Git identity configured (user.name and user.email)
- [ ] Feature branch created from main
- [ ] deployment.yaml created and applied successfully
- [ ] service.yaml created and applied successfully
- [ ] All 3 pods Running and spread across 3 nodes
- [ ] curl http://localhost:8080 returns nginx page (HTTP 200)
- [ ] Self-healing test passed (pod deleted and respawned)
- [ ] Scaling test passed (3 to 5 to 3 replicas)
- [ ] Rolling update tested (nginx 1.25 to 1.26)
- [ ] Rollback tested successfully
- [ ] All files committed with conventional commit messages
- [ ] PR opened with base: main and compare: feature branch
- [ ] PR merged into main
- [ ] Remote and local feature branch deleted
- [ ] Local main synced with git pull
- [ ] kubectl delete cleanup completed

---

*Kubernetes Learning Journey — Project 01 of 09*
