# Gearify GitOps Repository

This repository contains Kubernetes manifests, Helm charts, and ArgoCD configurations for the Gearify microservices platform.

## Repository Structure

```
gearify-gitops/
├── charts/                           # Helm charts
│   └── gearify-service/             # Base chart for microservices
├── bootstrap/                        # ArgoCD & cluster bootstrap
│   └── argocd/                      # ArgoCD installation
├── infrastructure/                   # Cluster-wide resources
│   └── base/                        # Namespaces, quotas, secrets
├── apps/                            # Application deployments
│   ├── base/{service}/              # Base service configs
│   └── overlays/{env}/{service}/    # Environment-specific overrides
├── observability/                   # Monitoring stack (future)
└── argocd/                          # ArgoCD Applications
    ├── projects/                    # ArgoCD projects
    └── applicationsets/             # ApplicationSets for services
```

## Quick Start

### Prerequisites

- EKS cluster deployed via `gearify-infrastructure`
- kubectl configured for the cluster
- Helm 3.x installed

### 1. Bootstrap ArgoCD

```bash
# Apply ArgoCD installation
kubectl apply -k bootstrap/argocd/

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2. Apply Infrastructure

```bash
kubectl apply -k infrastructure/base/
```

### 3. Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<EXTERNAL_SECRETS_ROLE_ARN>
```

### 4. Apply ArgoCD Projects and ApplicationSets

```bash
kubectl apply -f argocd/projects/
kubectl apply -f argocd/applicationsets/
```

## Helm Chart: gearify-service

The base Helm chart for all microservices includes:

- **Deployment** - Configurable replicas, resources, probes
- **Service** - ClusterIP service for internal communication
- **ServiceAccount** - With IRSA annotations for AWS access
- **HorizontalPodAutoscaler** - CPU/Memory-based autoscaling
- **PodDisruptionBudget** - For high availability
- **ConfigMap** - Environment variables
- **ExternalSecret** - AWS Secrets Manager integration
- **ServiceMonitor** - Prometheus scraping
- **Ingress** - ALB ingress (optional)

### Usage

Each service in `apps/overlays/{env}/{service}/` customizes the base chart:

```yaml
# values.yaml
gearify-service:
  nameOverride: catalog-svc

  image:
    repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/gearify-catalog-svc
    tag: "dev-abc123"

  configMap:
    enabled: true
    data:
      ASPNETCORE_ENVIRONMENT: "Development"
```

## Environment Configuration

### Dev Environment
- Namespace: `gearify-dev`
- Auto-sync: Enabled
- Resources: Low (50m-250m CPU, 128Mi-256Mi memory)
- Replicas: 1 per service

### Staging Environment (Future)
- Namespace: `gearify-staging`
- Auto-sync: Enabled
- Resources: Medium
- Replicas: 2 per service

### Production Environment (Future)
- Namespace: `gearify-prod`
- Auto-sync: Disabled (manual approval)
- Resources: High
- Replicas: 2-10 (HPA enabled)

## ArgoCD Applications

### ApplicationSets

The `services-appset.yaml` uses a matrix generator to create applications for all services across environments:

```yaml
generators:
  - matrix:
      generators:
        - list:
            elements:
              - env: dev
                namespace: gearify-dev
        - list:
            elements:
              - service: catalog-svc
              - service: order-svc
              # ... all services
```

### Sync Policies

| Environment | Auto Sync | Auto Prune | Self Heal |
|-------------|-----------|------------|-----------|
| Dev | Yes | Yes | Yes |
| Staging | Yes | Yes | Yes |
| Prod | No | No | No |

## Updating Service Images

When CI/CD pushes a new image, update the values file:

```bash
# Update image tag in dev overlay
cd apps/overlays/dev/catalog-svc/
sed -i 's/tag: ".*"/tag: "dev-newsha"/' values.yaml
git commit -am "Update catalog-svc to dev-newsha"
git push
```

ArgoCD will automatically sync (for dev/staging) or show the diff (for prod).

## Secrets Management

Secrets are stored in AWS Secrets Manager and synced via External Secrets Operator:

```yaml
externalSecrets:
  enabled: true
  data:
    - secretKey: DB_CONNECTION_STRING
      remoteRef:
        key: gearify/dev/database/main
        property: connection_string
```

### Secret Paths

| Secret | Path |
|--------|------|
| Database | `gearify/{env}/database/main` |
| Redis | `gearify/{env}/redis` |
| JWT | `gearify/{env}/jwt` |
| Stripe | `gearify/{env}/stripe` |
| SMTP | `gearify/{env}/smtp` |

## Troubleshooting

### Application Not Syncing

```bash
# Check application status
argocd app get catalog-svc-dev

# View sync details
argocd app sync catalog-svc-dev --dry-run

# Force sync
argocd app sync catalog-svc-dev --force
```

### External Secrets Not Working

```bash
# Check ExternalSecret status
kubectl get externalsecrets -n gearify-dev

# Describe for errors
kubectl describe externalsecret catalog-svc -n gearify-dev

# Check operator logs
kubectl logs -n external-secrets deployment/external-secrets
```

### Pod Issues

```bash
# Check pod status
kubectl get pods -n gearify-dev

# View pod logs
kubectl logs -f deployment/catalog-svc -n gearify-dev

# Describe pod for events
kubectl describe pod -l app.kubernetes.io/name=catalog-svc -n gearify-dev
```

## Contributing

1. Create a feature branch
2. Make changes to manifests
3. Test in dev environment
4. Create PR for review
5. Merge to main (auto-deploys to staging)
6. Promote to prod via PR
