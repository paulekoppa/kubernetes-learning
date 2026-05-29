# Project 3 - Stateful Applications with Kubernetes

## Overview

This project adds a persistent database tier to the multi-tier application built in Project 2. It demonstrates Persistent Volumes, Persistent Volume Claims, StatefulSets, Kubernetes Secrets, and proves that data survives pod deletion and rescheduling.


---


## Environment

| Property | Value |
|----------|-------|
| OS | Red Hat Enterprise Linux 9.6 (Plow) |
| Kubernetes | Minikube v1.35.1 |
| Cluster | 1 control plane + 2 worker nodes |
| Namespace | project-3 |
| GitHub Repo | https://github.com/paulekoppa/kubernetes-learning |


---


## Architecture

```text
[ Browser ]
 │ NodePort 30080
 ▼
[ Nginx Frontend ] Deployment, 1 replica
 │ /api/* proxied to backend
 ▼
[ Node.js Backend API ] Deployment, 2 replicas
 │ queries appdb via appuser
 ▼
[ MySQL 8.0 StatefulSet ] StatefulSet, 1 replica (mysql-0)
 │ /var/lib/mysql mounted from PVC
 ▼
[ PVC mysql-data-mysql-0 ] 1Gi RWO, auto-created by StorageClass
 │
 ▼
[ PV (auto) ] minikube-hostpath, dynamically provisioned
```


---


## Concepts Learned

### Stateless vs Stateful Applications

Stateless applications hold no data between requests. Any replica is identical to any other. Pod deletion causes no data loss. Deployments are the correct controller.

Stateful applications write data to disk that must survive pod restarts. Pod identity matters. StatefulSets are the correct controller.

Use **stateless** when:
- The pod holds no user data (nginx, REST API, reverse proxy)
- Any replica can handle any request equally
- Horizontal scaling is needed without coordination

Use **stateful** when:
- The pod writes data that must persist (databases, message queues)
- Each instance needs its own dedicated storage
- Pod identity and stable network address are required

### Persistent Volumes and Persistent Volume Claims

A **Persistent Volume (PV)** is the actual storage asset in the cluster. It exists independently of any pod or namespace. Analogous to a Physical Volume in LVM.

A **Persistent Volume Claim (PVC)** is a pod's request for storage. It binds to a matching PV. Analogous to a Logical Volume in LVM.

| LVM Concept | Kubernetes Equivalent |
|-------------|----------------------|
| Physical Volume (PV) | Persistent Volume (PV) |
| Volume Group (VG) | Storage Class |
| Logical Volume (LV) | Persistent Volume Claim (PVC) |
| Mount point | `volumeMounts` in pod spec |

Pods reference PVCs, never PVs directly. This decouples application configuration from infrastructure details.

### Storage Classes and Dynamic Provisioning

A Storage Class is a template and provisioner that automatically creates PVs when a PVC is submitted. This is dynamic provisioning.

Minikube ships with a default Storage Class named `standard` using the `minikube-hostpath` provisioner. No manual PV creation is needed.

Check the available Storage Classes:

```bash
kubectl get storageclass
kubectl describe storageclass standard
```

### StatefulSets vs Deployments

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random hash suffix | Stable ordinal (`mysql-0`, `mysql-1`) |
| Pod identity | Interchangeable | Unique and permanent |
| DNS per pod | No | Yes, via Headless Service |
| Storage per pod | Shared or none | Dedicated PVC per pod |
| PVC on pod delete | Deleted with pod | Survives, re-attaches |
| Startup order | All at once | Sequential 0 to N |
| Shutdown order | Any order | Reverse sequential |
| Use case | Stateless apps | Stateful apps |

StatefulSets require a Headless Service (`clusterIP: None`) to give each pod a stable DNS entry:

```text
mysql-0.mysql.project-3.svc.cluster.local
```

The `volumeClaimTemplates` section auto-creates one PVC per pod:
- Template name: `mysql-data`
- Pod `mysql-0`  ->  PVC `mysql-data-mysql-0` (never deleted on pod removal)
- Pod `mysql-1`  ->  PVC `mysql-data-mysql-1` (never deleted on pod removal)

### Kubernetes Secrets

Secrets store sensitive data separately from application configuration. Values are base64-encoded, not encrypted. Base64 is for safe transport of binary data, not for security.

| What Secrets Do Well | What Secrets Do Not Do |
|---------------------|----------------------|
| Keep values out of pod specs | Encrypt data at rest by default |
| Hide values in `kubectl describe` | Prevent cluster admins reading them |
| Enable RBAC-based access control | Replace a proper secrets manager |
| Separate sensitive config from code | Protect against compromised nodes |

> **Note:** In production, use Sealed Secrets or HashiCorp Vault for true encryption and audit trails. Secrets are the floor, not the ceiling.

Encode a value for a Secret:

```bash
echo -n 'mypassword' | base64
```

Decode a Secret value:

```bash
echo 'bXlwYXNzd29yZA==' | base64 --decode
```


---


## File Reference

| File | Purpose |
|------|---------|
| `namespace.yaml` | Creates the project-3 namespace |
| `mysql-secret.yaml` | Stores all MySQL credentials as base64-encoded Secret |
| `mysql-service.yaml` | Headless Service for stable per-pod DNS |
| `mysql-statefulset.yaml` | MySQL 8.0 StatefulSet with volumeClaimTemplate |
| `backend-configmap.yaml` | Node.js REST API application code |
| `backend-deployment.yaml` | Backend Deployment, 2 replicas, initContainer for npm |
| `backend-service.yaml` | ClusterIP Service for backend on port 3000 |
| `frontend-configmap.yaml` | Nginx config and HTML UI |
| `frontend-deployment.yaml` | Nginx frontend Deployment, 1 replica |
| `frontend-service.yaml` | NodePort Service on port 30080 |


---


## Deployment Order

Dependencies must be applied in this order:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-statefulset.yaml
kubectl apply -f backend-configmap.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-configmap.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```

Wait for MySQL to be ready before the backend starts:

```bash
kubectl wait --for=condition=ready pod/mysql-0 -n project-3 --timeout=120s
```


---


## Verification Commands

**Check all resources:**

```bash
kubectl get all -n project-3
kubectl get pvc -n project-3
kubectl get pv
kubectl get secret -n project-3
```

**Check StorageClass:**

```bash
kubectl get storageclass
```

**Test backend health:**

```bash
kubectl exec -n project-3 deploy/frontend -- \
  wget -qO- http://backend:3000/api/health
```

**Test users endpoint:**

```bash
kubectl exec -n project-3 deploy/frontend -- \
  wget -qO- http://backend:3000/api/users
```

**Add a user:**

```bash
kubectl exec -n project-3 deploy/frontend -- \
  wget -qO- --post-data='{"name":"Alice"}' \
  --header='Content-Type: application/json' \
  http://backend:3000/api/users
```

**Connect to MySQL directly:**

```bash
kubectl exec -it mysql-0 -n project-3 -- mysql -u root -prootpassword
```

**Confirm Secret values are masked:**

```bash
kubectl describe pod -n project-3 -l app=backend | grep -A2 "MYSQL_"
```

**Open frontend in browser:**

```bash
kubectl port-forward -n project-3 service/frontend 8080:80
# Then open http://localhost:8080
```


---


## Persistence Test

This test proves data survives pod deletion.

**Step 1 - Record current data:**

```bash
kubectl exec -n project-3 deploy/frontend -- \
  wget -qO- http://backend:3000/api/users
```

**Step 2 - Delete the MySQL pod:**

```bash
kubectl delete pod mysql-0 -n project-3
```

**Step 3 - Watch StatefulSet recreate it:**

```bash
kubectl get pods -n project-3 -w
```

**Step 4 - Confirm PVC was never deleted:**

```bash
kubectl get pvc -n project-3
# AGE of PVC will be much older than AGE of the new pod
```

**Step 5 - Confirm data is intact:**

```bash
kubectl exec -n project-3 deploy/frontend -- \
  wget -qO- http://backend:3000/api/users
```

*All rows will be present. The pod was replaced. The data was not.*


---


## Troubleshooting

**Pod stuck in Pending:**

```bash
kubectl describe pod <pod-name> -n project-3
# Check Events section for scheduling or PVC binding errors
```

**PVC stuck in Pending:**

```bash
kubectl describe pvc <pvc-name> -n project-3
# Check StorageClass exists and provisioner is running
kubectl get storageclass
```

**Backend cannot connect to MySQL:**

```bash
# Check MySQL is ready
kubectl get pod mysql-0 -n project-3

# Check DNS resolution from backend pod
kubectl exec -n project-3 deploy/backend -- \
  nslookup mysql-0.mysql.project-3.svc.cluster.local

# Check Secret keys exist
kubectl describe secret mysql-secret -n project-3
```

**YAML apply fails with token error:**

```bash
# Check for tab characters - YAML requires spaces only
cat -A <filename>.yaml | grep -P "\t"

# Always dry-run before applying
kubectl apply --dry-run=client -f <filename>.yaml
```

**CreateContainerConfigError:**

```bash
kubectl describe pod <pod-name> -n project-3
# Usually means a referenced Secret or ConfigMap does not exist
# Apply the Secret or ConfigMap first, then apply the Deployment
```


---


## MySQL User Management

The application connects as `appuser`, not root. This follows the principle of least privilege.

Create the application user:

```bash
kubectl exec -it mysql-0 -n project-3 -- mysql -u root -prootpassword
```

Run the following SQL commands:

```sql
CREATE USER 'appuser'@'%' IDENTIFIED BY 'apppassword';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE ON appdb.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
SELECT user, host FROM mysql.user WHERE user = 'appuser';
EXIT;
```

Verify grants:

```sql
SHOW GRANTS FOR 'appuser'@'%';
```


---


## Cluster Cleanup

**Remove all Project 3 resources:**

```bash
# Delete the namespace - cascades to all pods, services,
# deployments, statefulsets, and PVCs inside it
kubectl delete namespace project-3
```

PVs are cluster-scoped and are not deleted with the namespace. After deleting the namespace, clean up Released PVs:

```bash
kubectl get pv | grep Released
kubectl delete pv $(kubectl get pv | grep Released | awk '{print $1}')
```

*Always run this check after deleting any namespace that used storage. Apply this habit to all current and future projects.*

**Verify the cluster is clean:**

```bash
kubectl get pv
kubectl get namespace
```


---


## Git Workflow Used

```bash
# Start from main
git checkout main
git pull
git checkout -b feature/project-3-stateful-applications

# Commit incrementally throughout the project
git add project-3-stateful-applications/
git commit -m "feat: description"
git push -u origin feature/project-3-stateful-applications

# At project end: open PR on GitHub, merge, delete branch, sync main
git checkout main
git pull
git branch -d feature/project-3-stateful-applications
```


---


## Key Takeaways

| Concept | One-Line Summary |
|---------|-----------------|
| Ephemeral filesystem | Container storage is destroyed when the pod dies |
| Persistent Volume | The actual storage asset, lives independently of pods |
| Persistent Volume Claim | The pod's request for storage, binds to a PV |
| Storage Class | Automatically creates PVs when a PVC is submitted |
| StatefulSet | Like a Deployment but pods have stable names and storage |
| Headless Service | Gives each StatefulSet pod its own DNS address |
| `volumeClaimTemplates` | Auto-creates one dedicated PVC per StatefulSet pod |
| Secret | Separates sensitive config from application manifests |
| Least privilege | App connects as `appuser`, not root |
| Persistence proof | Pod age resets on recreation, PVC age does not |

> **Note:** Runbook authored during Kubernetes hands-on learning — Project 3 of series.
> Environment: Minikube on RHEL 9.6 | Access: Windows VDI → MobaXterm → RHEL sandbox
