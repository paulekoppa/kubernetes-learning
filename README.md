# Kubernetes Learning Journey 🚀

Hands-on Kubernetes projects with increasing complexity.


## Projects

| # | Project | Concepts |
|---|---|---|
| 01 | [Static Website Deployment](./project-1-static-website/) | Pods, Deployments, ReplicaSets, NodePort Services, Label Selectors, Self-Healing, Scaling, Rolling Updates, Rollbacks, Resource Requests and Limits, Port-Forward |
| 02 | [Multi-Tier Application](./project-2-multitier/) | Namespaces, ClusterIP Services, Kubernetes DNS, Nginx Reverse Proxy, ConfigMaps (Volume Mounts + Env Vars), Environment Variables, Multi-Tier Architecture, Load Balancing, Dependency Ordering, Rolling Restarts, Troubleshooting |
| 03 | [Stateful Applications](./project-3-stateful-applications/) | Persistent Volumes, Persistent Volume Claims, Storage Classes, Dynamic Provisioning, StatefulSets, Headless Services, volumeClaimTemplates, Stable Pod Identity, Stable Pod DNS, Data Persistence, Kubernetes Secrets, Least-Privilege Database Users, Stateless vs Stateful Architecture |
| 04 | [Observability and Resilience](./project-4-observability-resilience/) | Liveness Probes, Readiness Probes, Startup Probes, Probe Failure Simulations, Resource Requests and Limits, Metrics Server, kubectl top, HorizontalPodAutoscaler, CPU-Based Autoscaling, HPA Stress Test, HPA Scale-Down Stabilization, ConfigMap Live Reload, File vs Process Memory Distinction |
| 05 | [Networking — Ingress, TLS, and NetworkPolicy](./project-5-networking/) | Ingress Controller, Path-Based Routing, Ingress Resources, ingressClassName, rewrite-target, Host Header Routing, TLS Termination, Self-Signed Certificates, openssl, kubernetes.io/tls Secrets, HTTP to HTTPS Redirect, NetworkPolicy, Default-Deny Pattern, Least-Privilege Networking, namespaceSelector, DNS Egress Rules |


## Environment

- Minikube v1.35.1 (1 control plane + 2 worker nodes)
- RHEL 9.6


## Git Workflow

- Branch from main: git checkout -b feature/name
- Conventional commits: feat, docs, chore
- Push feature branch → open PR → merge → sync main → delete branch


## Cluster Hygiene

After deleting any namespace that used storage, always clean up
orphaned Persistent Volumes:

````bash
kubectl get pv | grep Released
kubectl delete pv $(kubectl get pv | grep Released | awk '{print $1}')
