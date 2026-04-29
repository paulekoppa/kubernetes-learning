# Project 01 - Static Website Deployment

Deploys a 3-replica nginx static website using a Kubernetes Deployment and NodePort Service.

## Apply

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml


## Access

# Port forward to access locally
kubectl port-forward svc/static-website-svc 8080:80

# Test
curl http://localhost:8080


## Concepts Covered
- Deployments and ReplicaSets
- NodePort Services
- Label selectors
- Self-healing
- Rolling updates
