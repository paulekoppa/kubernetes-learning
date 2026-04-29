# Project 01 - Static Website Deployment

Deploys a 3-replica nginx static website using a Kubernetes Deployment and NodePort Service.

## Apply
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```bash

## Access
```bash
# Port forward to access locally
kubectl port-forward svc/static-website-svc 8080:80

# Test
curl http://localhost:8080
```bash

## Concepts Covered
Deployments and ReplicaSets
NodePort Services
Label selectors
Self-healing
Rolling updates
