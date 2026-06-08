# Project 5 — Kubernetes Networking: Ingress, TLS, and NetworkPolicy


---


## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Project Structure](#project-structure)
* [Step 1 — Enable the Nginx Ingress Controller](#step-1--enable-the-nginx-ingress-controller)
* [Step 2 — Deploy the Application](#step-2--deploy-the-application)
* [Step 3 — Create the Ingress Resource](#step-3--create-the-ingress-resource)
* [Step 4 — Test HTTP Routing](#step-4--test-http-routing)
* [Step 5 — Generate a Self-Signed TLS Certificate](#step-5--generate-a-self-signed-tls-certificate)
* [Step 6 — Create the TLS Secret](#step-6--create-the-tls-secret)
* [Step 7 — Add TLS Termination to the Ingress](#step-7--add-tls-termination-to-the-ingress)
* [Step 8 — Test HTTPS Access](#step-8--test-https-access)
* [Step 9 — Apply Default-Deny NetworkPolicy](#step-9--apply-default-deny-networkpolicy)
* [Step 10 — Apply Targeted Allow Policies](#step-10--apply-targeted-allow-policies)
* [Step 11 — Stress-Test the NetworkPolicy](#step-11--stress-test-the-networkpolicy)
* [NetworkPolicy Reference — firewalld Comparison](#networkpolicy-reference--firewalld-comparison)
* [Secret Discovery Reference](#secret-discovery-reference)
* [.gitignore Best Practices](#gitignore-best-practices)
* [Concepts Learned](#concepts-learned)
* [Teardown](#teardown)


---


## Overview

This project covers Kubernetes networking from the cluster edge to pod-level micro-segmentation. It builds a two-tier application (frontend and backend) exposed via an Nginx Ingress Controller with HTTPS termination and locked down with NetworkPolicy using a default-deny whitelist model.

| Area | What was built |
|---|---|
| Ingress Controller | Nginx addon enabled in Minikube |
| Routing | Path-based routing — `/` to frontend, `/api` to backend |
| TLS | Self-signed certificate, `kubernetes.io/tls` Secret, HTTPS termination |
| HTTP redirect | 308 Permanent Redirect from HTTP to HTTPS |
| NetworkPolicy | Default-deny baseline with targeted allow rules |
| Security posture | Least-privilege — only Ingress Controller can reach pods |


---


## Architecture

```text
Browser
 │
 │ HTTPS port 443
 ▼
Ingress Controller Pod (ingress-nginx namespace)
 │ TLS terminates here — traffic decrypted
 │
 │ plain HTTP port 80
 │
 ├── path: /     --> frontend-service --> frontend pods (x2)
 └── path: /api  --> backend-service  --> backend pods (x2)

NetworkPolicy enforcement (project-5 namespace):

default-deny-all  blocks all ingress and egress by default
allow-frontend    ingress-nginx --> frontend:80, egress --> kube-dns:53
allow-backend     ingress-nginx --> backend:80, egress --> kube-dns:53
frontend --> backend BLOCKED (no direct pod-to-pod traffic allowed)
pods --> internet BLOCKED
```


---


## Prerequisites

* Minikube running with at least 2 worker nodes
* `kubectl` configured and pointing to Minikube
* `openssl` installed on the host machine
* Entry added to `/etc/hosts`: `192.168.49.2  project5.local`


---


## Project Structure

```text
project-5-networking/
├── .gitignore                      # excludes tls.key and tls.crt from Git
├── frontend-configmap.yaml         # HTML content for frontend nginx
├── frontend-deployment.yaml        # frontend Deployment + ClusterIP Service
├── backend-configmap.yaml          # JSON content for backend nginx
├── backend-deployment.yaml         # backend Deployment + ClusterIP Service
├── ingress.yaml                    # Ingress resource with TLS and path routing
├── networkpolicy-default-deny.yaml # blocks all traffic in namespace
├── networkpolicy-frontend.yaml     # targeted allow rules for frontend pods
├── networkpolicy-backend.yaml      # targeted allow rules for backend pods
└── RUNBOOK.md                      # this file
```

> **Note:** `tls.key` and `tls.crt` are generated locally and excluded from Git via `.gitignore`.


---


## Step 1 — Enable the Nginx Ingress Controller

The Ingress Controller is a running pod that reads Ingress Resource YAML rules and performs the actual HTTP/HTTPS routing. Minikube ships it as a built-in addon.

```bash
minikube addons enable ingress
```

Verify the controller is running:

```bash
kubectl get pods -n ingress-nginx
kubectl get namespace ingress-nginx --show-labels
```

Expected pod states:

| Pod | Expected status |
|---|---|
| `ingress-nginx-controller` | Running |
| `ingress-nginx-admission-create` | Completed |
| `ingress-nginx-admission-patch` | Completed |

The two `Completed` pods are one-time Job pods that set up TLS webhooks at install time. They are not crashed — they finished successfully.

### Ingress Controller concept

* **Without Ingress:** Browser --> NodePort (awkward port) --> pod
* **With Ingress:** Browser --> Ingress Controller (port 80/443) --> ClusterIP --> pod

The Ingress Controller is the professional front desk — one entry point that routes traffic to the right internal service based on hostname and path rules.


---


## Step 2 — Deploy the Application

Create the namespace first, then apply ConfigMaps before Deployments. ConfigMaps must exist before the pods that mount them are scheduled.

```bash
kubectl create namespace project-5
kubectl apply -f frontend-configmap.yaml
kubectl apply -f backend-configmap.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f backend-deployment.yaml
```

Verify all resources are healthy:

Expected state:

| Resource | Count | Status |
|---|---|---|
| frontend pods | 2/2 | Running |
| backend pods | 2/2 | Running |
| frontend-service | 1 | ClusterIP |
| backend-service | 1 | ClusterIP |

### Why ClusterIP and not NodePort

NodePort was used in Project 1 because there was no Ingress. Now that we have an Ingress Controller, ClusterIP is the correct service type. The Ingress Controller handles all external traffic — backend services only need to be reachable internally.


---


## Step 3 — Create the Ingress Resource

The Ingress Resource is a YAML object that defines routing rules. It does not do the routing itself — the Ingress Controller reads these rules and enforces them.

```bash
kubectl apply -f ingress.yaml --dry-run=client
kubectl apply -f ingress.yaml
```

Verify routing rules:

```bash
kubectl get ingress -n project-5
kubectl describe ingress project5-ingress -n project-5
```

Expected output from describe:

```text
Rules:
  Host            Path  Backends
  ----            ----  --------
  project5.local
                  /     frontend-service:80
                  /api  backend-service:80
```

### Path types

| pathType | Behaviour |
|---|---|
| `Exact` | Matches the path character for character only |
| `Prefix` | Matches the path and anything that starts with it |
| `ImplementationSpecific` | Left to the Ingress Controller to decide |

`Prefix` is used here — `/api` also matches `/api/users`, `/api/health`, etc.

### rewrite-target annotation

Without `rewrite-target`, a request to `/api` is forwarded as `/api` to the backend pod. The backend nginx only serves content at `/`, so it returns 404. With `rewrite-target: /`, the path is rewritten to `/` before forwarding.

### ingressClassName vs deprecated annotation

Modern Kubernetes (1.18+) uses `spec.ingressClassName` in the YAML spec. The old `kubernetes.io/ingress.class` annotation is deprecated and produces a warning. Always use `ingressClassName` in new manifests.


---


## Step 4 — Test HTTP Routing

Add the hostname to `/etc/hosts` so the OS resolves it locally without DNS:

```bash
echo "192.168.49.2 project5.local" | sudo tee -a /etc/hosts
```

Test path routing:

```bash
curl [http://project5.local/](http://project5.local/)
curl [http://project5.local/api](http://project5.local/api)
```

Prove the Host header is required:

```bash
curl [http://192.168.49.2/](http://192.168.49.2/)
```

Expected results:

| Request | Expected response |
|---|---|
| `curl http://project5.local/` | Frontend HTML |
| `curl http://project5.local/api` | Backend JSON |
| `curl http://192.168.49.2/` | 404 Not Found |

### Why the raw IP returns 404

The Ingress Controller routes based on the HTTP Host header, not the IP address. When you curl the raw IP, no Host header is set, so no rule matches and the request falls through to the default 404 backend. This proves routing is hostname-driven, not IP-driven.


---


## Step 5 — Generate a Self-Signed TLS Certificate

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=project5.local/O=Kubernetes-Learning"
```

Verify the certificate:

```bash
openssl x509 -in tls.crt -text -noout | grep -E "Subject:|Not After|Issuer:"
```

Expected output:

```text
Issuer: CN=project5.local, O=Kubernetes-Learning
Not After : <date 365 days from generation>
Subject: CN=project5.local, O=Kubernetes-Learning
```

### Self-signed vs CA-signed certificates

| Property | Self-signed | CA-signed (Let's Encrypt) |
|---|---|---|
| Issuer | Same as Subject | Trusted third party |
| Browser trust | Warning shown | Trusted automatically |
| Encryption strength | Identical | Identical |
| Use case | Local development | Production |

The Issuer and Subject being identical is the hallmark of a self-signed certificate. In production, cert-manager automates certificate issuance from Let's Encrypt.

### openssl flag reference

| Flag | Meaning |
|---|---|
| `-x509` | Output a self-signed certificate directly |
| `-nodes` | No passphrase on the private key |
| `-days 365` | Certificate validity period |
| `-newkey rsa:2048` | Generate a new 2048-bit RSA key pair |
| `-keyout tls.key` | Write private key to this file |
| `-out tls.crt` | Write certificate to this file |
| `-subj` | Certificate subject fields — avoids interactive prompts |


---


## Step 6 — Create the TLS Secret

```bash
kubectl create secret tls project5-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n project-5
```

Verify the Secret:

```bash
kubectl get secret project5-tls -n project-5
kubectl describe secret project5-tls -n project-5
```

Expected:

```text
NAME           TYPE                DATA   AGE
project5-tls   kubernetes.io/tls   2      Xs
```

### Secret types comparison

| Type | Used for | Required keys |
|---|---|---|
| `Opaque` | Passwords, tokens, connection strings | Any names you choose |
| `kubernetes.io/tls` | TLS certificates | Must be `tls.crt` and `tls.key` exactly |

The Ingress Controller specifically looks for the `kubernetes.io/tls` type and expects those exact key names. An `Opaque` Secret with the same data would not be recognised.

### How to discover Secrets in any namespace

```bash
# List all Secrets in a namespace
kubectl get secrets -n project-5

# List all Secrets across all namespaces
kubectl get secrets -A

# Filter by type
kubectl get secrets -A --field-selector type=kubernetes.io/tls

# Inspect without exposing values
kubectl describe secret project5-tls -n project-5

# View base64-encoded values for debugging
kubectl get secret project5-tls -n project-5 -o yaml
```

### Secret naming convention

| Pattern | Example |
|---|---|
| `purpose-type` | `project5-tls`, `db-credentials`, `api-token` |
| `app-purpose` | `frontend-tls`, `mysql-secret`, `backend-api-key` |

The Secret name must match exactly what is referenced in the Ingress `spec.tls.secretName` field.


---


## Step 7 — Add TLS Termination to the Ingress

Update `ingress.yaml` to add the `tls` block and `ssl-redirect` annotation, then apply:

```bash
kubectl apply -f ingress.yaml --dry-run=client
kubectl apply -f ingress.yaml
```

Verify TLS is wired:

```bash
kubectl get ingress -n project-5
kubectl describe ingress project5-ingress -n project-5
```

Expected in `get` output:

```text
NAME               CLASS   HOSTS            ADDRESS        PORTS     AGE
project5-ingress   nginx   project5.local   192.168.49.2   80, 443   Xm
```

Expected in `describe` output:

```text
TLS:
  project5-tls terminates project5.local
```

### Why port 443 is not in the NetworkPolicy

TLS terminates at the Ingress Controller — not at the application pods. By the time traffic reaches frontend or backend pods it is already decrypted and forwarded as plain HTTP on port 80. The pods never see port 443.

| Hop | Protocol | Port |
|---|---|---|
| Browser to Ingress Controller | HTTPS | 443 |
| Ingress Controller to frontend pod | HTTP | 80 |
| Ingress Controller to backend pod | HTTP | 80 |

NetworkPolicy port numbers must match the port the receiving pod is listening on, not the port the original client used.


---


## Step 8 — Test HTTPS Access

Test HTTPS frontend:
```bash
curl -k [https://project5.local/](https://project5.local/)
```

Test HTTPS backend:
```bash
curl -k [https://project5.local/api](https://project5.local/api)
```

Prove HTTP redirects to HTTPS (308 Permanent Redirect):
```bash
curl -v [http://project5.local/](http://project5.local/) 2>&1 | grep -E "< HTTP|Location"
```

Inspect the certificate served by the Ingress Controller:
```bash
echo | openssl s_client -connect 192.168.49.2:443 \
  -servername project5.local 2>/dev/null | \
  openssl x509 -noout -text | grep -E "Subject:|Not After|Issuer:"
```

Expected results:

| Test | Expected result |
|---|---|
| HTTPS frontend | Frontend HTML over port 443 |
| HTTPS backend | Backend JSON over port 443 |
| HTTP redirect | 308 Permanent Redirect to `https://project5.local` |
| Certificate | Issuer and Subject both show `project5.local` |

### Why curl -k is needed

The `-k` flag skips certificate chain verification. Our cert is self-signed so no trusted CA vouches for it. The traffic is still fully encrypted — we are only bypassing the identity verification step. In production with a CA-signed certificate, `-k` would never be needed.


---


## Step 9 — Apply Default-Deny NetworkPolicy

Before applying the policy, prove that unrestricted access exists by exec-ing into a frontend pod and reaching the backend directly:

```bash
kubectl exec -it -n project-5 \
  $(kubectl get pod -n project-5 -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  -- wget -qO- http://backend-service/
```

This should succeed (returns backend JSON) — proving unrestricted pod-to-pod access.

Now apply the default-deny policy:

```bash
kubectl apply -f networkpolicy-default-deny.yaml
```

Verify the application is now broken:

```bash
curl -k --max-time 5 [https://project5.local/](https://project5.local/)
```

Expected: `curl timeout` — confirming the deny is active.

### How default-deny works

An empty `podSelector` with no ingress or egress rules blocks all traffic.

```yaml
  podSelector: {}  # applies to ALL pods in namespace
  policyTypes:
  - Ingress        # declared but no rules = deny all inbound
  - Egress         # declared but no rules = deny all outbound
```

### Why DNS breaks too

The Egress deny also blocks pods from reaching kube-dns on port 53. Without DNS, service name resolution fails immediately — wget shows `bad address` instead of a connection timeout. This is why the allow policies must explicitly permit egress to kube-dns port 53.


---


## Step 10 — Apply Targeted Allow Policies

```bash
kubectl apply -f networkpolicy-frontend.yaml
kubectl apply -f networkpolicy-backend.yaml
```

Verify all three policies exist:

```bash
kubectl get networkpolicies -n project-5
```

Expected:

```text
NAME               POD-SELECTOR   AGE
default-deny-all   <none>         Xm
allow-frontend     app=frontend   Xs
allow-backend      app=backend    Xs
```

Verify the application is restored:

```bash
curl -k --max-time 5 [https://project5.local/](https://project5.local/)
curl -k --max-time 5 [https://project5.local/api](https://project5.local/api)
```

### How NetworkPolicy rules layer

NetworkPolicy is additive — multiple policies targeting the same pod combine with OR logic. If any policy allows a connection it is permitted. The deny-all policy has no rules so nothing is allowed. Each allow policy punches a specific hole through the wall.

### Allowed traffic paths after all policies are applied

| Source | Destination | Port | Allowed |
|---|---|---|---|
| ingress-nginx pods | frontend pods | 80 TCP | Yes |
| ingress-nginx pods | backend pods | 80 TCP | Yes |
| frontend pods | kube-dns | 53 UDP+TCP | Yes |
| backend pods | kube-dns | 53 UDP+TCP | Yes |
| frontend pods | backend pods | any | No |
| backend pods | frontend pods | any | No |
| any pod | internet | any | No |


---


## Step 11 — Stress-Test the NetworkPolicy

Run all four tests to confirm only allowed paths work:

**Test 1 — frontend cannot reach backend directly (should timeout)**
```bash
kubectl exec -it -n project-5 \
  $(kubectl get pod -n project-5 -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  -- wget -qO- --timeout=5 http://backend-service/
```

**Test 2 — frontend cannot reach the internet (should timeout)**
```bash
kubectl exec -it -n project-5 \
  $(kubectl get pod -n project-5 -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  -- wget -qO- --timeout=5 [http://example.com/](http://example.com/)
```

**Test 3 — legitimate HTTPS frontend still works**
```bash
curl -k --max-time 5 [https://project5.local/](https://project5.local/)
```

**Test 4 — legitimate HTTPS backend still works**
```bash
curl -k --max-time 5 [https://project5.local/api](https://project5.local/api)
```

Expected results:

| Test | Expected |
|---|---|
| frontend to backend direct | wget download timed out |
| frontend to internet | wget download timed out |
| HTTPS frontend via Ingress | Frontend HTML returned |
| HTTPS backend via Ingress | Backend JSON returned |


---


## NetworkPolicy Reference — firewalld Comparison

For engineers familiar with Linux firewalld, NetworkPolicy maps as follows:

| firewalld | Kubernetes NetworkPolicy |
|---|---|
| `systemctl stop firewalld` | `kubectl delete networkpolicy default-deny-all -n namespace` |
| `systemctl start firewalld` | `kubectl apply -f networkpolicy-default-deny.yaml` |
| `firewall-cmd --add-port=80/tcp` | `kubectl apply a new NetworkPolicy YAML` |
| `firewall-cmd --remove-port=80/tcp` | `kubectl delete networkpolicy policy-name -n namespace` |
| `firewall-cmd --list-all` | `kubectl get networkpolicies -n namespace` |
| `firewall-cmd --info-zone` | `kubectl describe networkpolicy name -n namespace` |

### Practical NetworkPolicy management commands

```bash
# Remove default-deny to allow all traffic temporarily
kubectl delete networkpolicy default-deny-all -n project-5

# Re-apply default-deny to restore blocking
kubectl apply -f networkpolicy-default-deny.yaml

# Remove one specific allow policy
kubectl delete networkpolicy allow-frontend -n project-5

# Remove all NetworkPolicies at once (full open access)
kubectl delete networkpolicies --all -n project-5

# List all policies across all namespaces
kubectl get networkpolicies -A
```

### Key differences from firewalld

| Property | firewalld | NetworkPolicy |
|---|---|---|
| Scope | Entire machine | Namespace-scoped |
| Default with no rules | Depends on zone | Allow all |
| Default with deny policy | Blocks traffic | Blocks traffic |
| Rule engine | Zone and service based | Label selector based |

If no NetworkPolicy exists in a namespace, Kubernetes defaults to allow all. It is only when at least one policy exists that traffic starts being evaluated. This is why the `default-deny-all` pattern is the production standard — you explicitly declare deny-all so the safe default is enforced from the start.

Deleting a policy in `project-5` has zero effect on `project-3`, `project-4`, or `ingress-nginx`. Each namespace is its own firewall domain.


---


## Secret Discovery Reference

```bash
# List all Secrets in a specific namespace
kubectl get secrets -n project-5

# List all Secrets across every namespace
kubectl get secrets -A

# Filter by Secret type
kubectl get secrets -A --field-selector type=kubernetes.io/tls
kubectl get secrets -n project-5 --field-selector type=Opaque

# Inspect a Secret without exposing values
kubectl describe secret project5-tls -n project-5

# View base64-encoded values for debugging
kubectl get secret project5-tls -n project-5 -o yaml
```


---


## .gitignore Best Practices

A `.gitignore` file should be created in every project directory. Private keys and sensitive files must never be committed to version control.

### Recommended .gitignore for Kubernetes projects

```text
# TLS private key — never commit to version control
# Anyone with this file can impersonate your server
tls.key

# TLS certificate — excluded for cleanliness
# In production managed by cert-manager or Vault
tls.crt

# Kubernetes Secret YAML files containing plaintext credentials
# If you ever dump a Secret to YAML for debugging, exclude it
*-secret.yaml
*-credentials.yaml

# Local environment files
.env
.env.local

# Editor and OS noise
.DS_Store
.idea/
*.swp
*.swo
Thumbs.db
```

### Pattern reference

| Pattern | What it excludes |
|---|---|
| `tls.key` | Exact filename match |
| `*.key` | All files ending in `.key` |
| `*-secret.yaml` | Any YAML file ending in `-secret.yaml` |
| `secrets/` | Entire directory named secrets |
| `!public.crt` | Exception — do not exclude this specific file |

### Verify .gitignore is working before every commit

```bash
# Confirm a specific file is being ignored
git check-ignore -v project-5-networking/tls.key

# See all ignored files in a directory
git status --ignored project-5-networking/
```


---


## Concepts Learned

| Concept | What was learned |
|---|---|
| Ingress Controller | Running pod that reads Ingress rules and performs routing |
| Ingress Resource | YAML object declaring hostname and path routing rules |
| `ingressClassName` | Modern field (1.18+) replacing deprecated annotation |
| Path-based routing | Single hostname, multiple paths to different services |
| `rewrite-target` annotation | Strips path prefix before forwarding to backend pod |
| Host header | HTTP header the Ingress Controller reads to select routing rules |
| TLS termination | HTTPS decrypted at Ingress edge — backends receive plain HTTP |
| Self-signed certificate | Cert signed by itself — same encryption, no CA trust |
| `kubernetes.io/tls` Secret | Purpose-built Secret type for TLS — requires `tls.crt` and `tls.key` keys |
| `ssl-redirect` | Forces HTTP to HTTPS with 308 Permanent Redirect |
| `curl -k` | Skips certificate chain verification for self-signed certs |
| NetworkPolicy | Pod-level firewall rules using label selectors |
| Default-deny pattern | Block all traffic first, then selectively allow |
| Additive policy model | Multiple policies combine with OR logic |
| `namespaceSelector` | Selects pods from a specific namespace in NetworkPolicy rules |
| DNS egress rule | Pods need explicit egress to `kube-dns:53` for name resolution |
| Least privilege networking | Only the exact required traffic paths are opened |


---


## Teardown

Delete all application resources:
```bash
kubectl delete namespace project-5
```

Verify namespace is gone:
```bash
kubectl get namespace project-5
```

Remove the `/etc/hosts` entry:
```bash
sudo sed -i '/project5.local/d' /etc/hosts
```

Verify removal:
```bash
cat /etc/hosts | grep project5
```

Optionally disable the Ingress addon:
```bash
minikube addons disable ingress
```

> **Note:** Deleting the namespace removes all resources inside it including Deployments, Services, ConfigMaps, Secrets, and NetworkPolicies. The `tls.key` and `tls.crt` files on the host filesystem must be deleted manually if no longer needed.

> **Note:** Runbook authored during Kubernetes hands-on learning — Project 5 of series.
> Environment: Minikube on RHEL 9.6 | Access: Windows VDI → MobaXterm → RHEL sandbox
