# GitOps Quick Start Guide

This guide will help you get started with the Gearify GitOps setup.

---

## Prerequisites

Before starting, ensure you have:

- [ ] EKS cluster deployed (via gearify-infrastructure)
- [ ] kubectl configured to access the cluster
- [ ] Helm 3.x installed
- [ ] AWS CLI configured

### Verify Prerequisites

```bash
# Check kubectl connection
kubectl cluster-info

# Check Helm version
helm version

# Check AWS CLI
aws sts get-caller-identity
```

---

## Step 1: Clone the Repository

```bash
git clone https://github.com/your-org/gearify-gitops.git
cd gearify-gitops
```

---

## Step 2: Validate Helm Charts

```bash
# Validate the base chart
cd charts/gearify-service
helm lint .

# Preview what will be generated for a service
helm template catalog-svc . \
  -f ../../apps/overlays/dev/catalog-svc/values.yaml
```

---

## Step 3: Bootstrap ArgoCD

```bash
# Create argocd namespace and install ArgoCD
kubectl apply -k bootstrap/argocd/

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=300s

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo  # New line after password
```

---

## Step 4: Access ArgoCD UI

```bash
# Port-forward to access locally
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 in your browser:
- **Username:** admin
- **Password:** (from Step 3)

---

## Step 5: Apply Cluster Infrastructure

```bash
# Apply namespaces, quotas, and network policies
kubectl apply -k infrastructure/base/
```

This creates:
- `gearify-dev` namespace
- Resource quotas
- Network policies

---

## Step 6: Install External Secrets Operator

```bash
# Add Helm repo
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Install operator
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace

# Verify installation
kubectl get pods -n external-secrets
```

---

## Step 7: Apply ClusterSecretStore

```bash
# This connects External Secrets to AWS Secrets Manager
kubectl apply -f infrastructure/base/cluster-secret-store.yaml
```

---

## Step 8: Apply ArgoCD Projects

```bash
kubectl apply -f argocd/projects/
```

---

## Step 9: Deploy Services with ApplicationSet

```bash
kubectl apply -f argocd/applicationsets/
```

This single command deploys ALL 13 services!

---

## Step 10: Verify Deployment

### In ArgoCD UI

1. Open https://localhost:8080
2. You should see all services listed
3. Each should show "Synced" and "Healthy"

### Via kubectl

```bash
# Check all pods
kubectl get pods -n gearify-dev

# Check services
kubectl get svc -n gearify-dev

# Check a specific deployment
kubectl describe deployment catalog-svc -n gearify-dev
```

---

## Common Tasks

### Update a Service Image

1. Edit the values file:
   ```bash
   cd apps/overlays/dev/catalog-svc
   # Edit values.yaml, change image.tag
   ```

2. Commit and push:
   ```bash
   git add values.yaml
   git commit -m "Update catalog-svc to v1.2.3"
   git push
   ```

3. ArgoCD automatically syncs (for dev environment)

### Add a New Service

See [Adding a New Service](README.md#adding-a-new-service)

### Check Logs

```bash
kubectl logs -f deployment/catalog-svc -n gearify-dev
```

### Port-Forward for Testing

```bash
kubectl port-forward svc/catalog-svc 5001:5001 -n gearify-dev
# Now access http://localhost:5001
```

---

## Troubleshooting

### ArgoCD Application Not Syncing

```bash
# Check ArgoCD logs
kubectl logs -f deployment/argocd-application-controller -n argocd

# Force sync
argocd app sync catalog-svc-dev --force
```

### Pod Stuck in Pending

```bash
# Check events
kubectl describe pod -l app=catalog-svc -n gearify-dev

# Usually means: insufficient resources or missing secrets
```

### External Secrets Not Working

```bash
# Check ExternalSecret status
kubectl get externalsecrets -n gearify-dev

# Describe for errors
kubectl describe externalsecret catalog-svc-secrets -n gearify-dev

# Check if the secret was created
kubectl get secret catalog-svc-secrets -n gearify-dev
```

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                     DEPLOYMENT ARCHITECTURE                       |
+------------------------------------------------------------------+

GitHub (gearify-gitops)          ArgoCD              Kubernetes
+---------------------+    +----------------+    +----------------+
|                     |    |                |    |                |
| apps/overlays/dev/  |--->| Watches repo   |--->| gearify-dev    |
|   catalog-svc/      |    | Detects changes|    |   namespace    |
|   order-svc/        |    | Applies to K8s |    |                |
|   ...               |    |                |    | Runs pods      |
|                     |    |                |    |                |
+---------------------+    +----------------+    +----------------+

AWS
+---------------------+
|                     |
| ECR (Images)        |
| Secrets Manager     |
| S3, DynamoDB, RDS   |
|                     |
+---------------------+
```

---

## Next Steps

1. **Set up CI/CD** to automatically update image tags
2. **Configure monitoring** with Prometheus/Grafana
3. **Set up alerting** for production
4. **Create staging environment** overlay
5. **Prepare production** with manual sync policy
