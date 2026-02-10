# Gearify GitOps Repository

This repository contains all Kubernetes configuration for deploying Gearify microservices using GitOps principles.

---

## Table of Contents

1. [What is GitOps?](#what-is-gitops)
2. [What is Helm?](#what-is-helm)
3. [Repository Structure](#repository-structure)
4. [How It All Works Together](#how-it-all-works-together)
5. [Helm Charts Explained](#helm-charts-explained)
6. [ArgoCD Configuration](#argocd-configuration)
7. [Adding a New Service](#adding-a-new-service)
8. [Common Commands](#common-commands)
9. [Troubleshooting](#troubleshooting)

---

## What is GitOps?

GitOps is a way of managing infrastructure and applications where:

```
+------------------------------------------------------------------+
|                        TRADITIONAL WAY                            |
|                                                                   |
|   Developer --> kubectl apply --> Kubernetes                      |
|                                                                   |
|   Problem: No history, no review, hard to track changes           |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|                         GITOPS WAY                                |
|                                                                   |
|   Developer --> Git Push --> ArgoCD --> Kubernetes                |
|                                                                   |
|   Benefits:                                                       |
|   - Git is the single source of truth                             |
|   - All changes are tracked in Git history                        |
|   - Changes go through pull requests (review)                     |
|   - Easy rollback (git revert)                                    |
|   - Automatic sync from Git to cluster                            |
+------------------------------------------------------------------+
```

### Key Principles

| Principle | Description |
|-----------|-------------|
| **Declarative** | You describe WHAT you want, not HOW to get it |
| **Versioned** | All config is in Git with full history |
| **Automated** | Changes in Git automatically apply to cluster |
| **Self-healing** | If someone manually changes cluster, it reverts |

### Simple Example

```yaml
# You write this in Git:
replicas: 3

# ArgoCD sees the change and runs:
# kubectl scale deployment myapp --replicas=3

# If someone manually runs:
# kubectl scale deployment myapp --replicas=1

# ArgoCD will automatically revert it back to 3
```

---

## What is Helm?

Helm is a **package manager for Kubernetes** - like npm for Node.js or pip for Python.

### The Problem Helm Solves

Without Helm, deploying a service requires multiple YAML files:

```
my-service/
├── deployment.yaml      # 50 lines
├── service.yaml         # 20 lines
├── configmap.yaml       # 30 lines
├── serviceaccount.yaml  # 15 lines
├── hpa.yaml             # 25 lines
└── ingress.yaml         # 20 lines

# Total: 160 lines per service
# 13 services = 2,080 lines of YAML!
# And 90% is duplicated...
```

### How Helm Solves It

Helm uses **templates** with **variables**:

```
charts/gearify-service/           # Reusable template (write once)
├── templates/
│   ├── deployment.yaml          # Template with {{ .Values.xxx }}
│   ├── service.yaml
│   └── ...
└── values.yaml                   # Default values

apps/overlays/dev/catalog-svc/
└── values.yaml                   # Just the differences
```

### Template Example

**Template (templates/deployment.yaml):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}
spec:
  replicas: {{ .Values.replicas }}
  template:
    spec:
      containers:
        - name: {{ .Values.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.port }}
```

**Values (values.yaml):**
```yaml
name: catalog-svc
replicas: 2
port: 5001
image:
  repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/gearify-catalog-svc
  tag: dev-latest
```

**Result after Helm processes it:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-svc
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: catalog-svc
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/gearify-catalog-svc:dev-latest
          ports:
            - containerPort: 5001
```

---

## Repository Structure

```
gearify-gitops/
│
├── charts/                          # HELM CHARTS (Templates)
│   └── gearify-service/             # Base chart for all microservices
│       ├── Chart.yaml               # Chart metadata
│       ├── values.yaml              # Default values
│       └── templates/               # Kubernetes YAML templates
│           ├── deployment.yaml      # Pod deployment
│           ├── service.yaml         # Network service
│           ├── serviceaccount.yaml  # AWS permissions (IRSA)
│           ├── configmap.yaml       # Environment variables
│           ├── externalsecret.yaml  # Secrets from AWS
│           ├── hpa.yaml             # Auto-scaling
│           ├── pdb.yaml             # High availability
│           ├── ingress.yaml         # External access
│           └── servicemonitor.yaml  # Prometheus metrics
│
├── apps/                            # SERVICE CONFIGURATIONS
│   ├── base/                        # Base configs (shared)
│   │   └── catalog-svc/
│   │       └── values.yaml
│   └── overlays/                    # Environment-specific
│       └── dev/                     # Development environment
│           ├── catalog-svc/
│           │   └── values.yaml      # Dev settings for catalog
│           ├── order-svc/
│           │   └── values.yaml      # Dev settings for orders
│           └── ... (13 services)
│
├── argocd/                          # ARGOCD CONFIGURATION
│   ├── projects/                    # ArgoCD projects (grouping)
│   │   ├── gearify-apps.yaml
│   │   └── gearify-infrastructure.yaml
│   ├── applicationsets/             # Auto-deploy all services
│   │   ├── services-appset.yaml
│   │   └── web-appset.yaml
│   └── apps/                        # Individual app definitions
│       └── dev/
│           ├── infrastructure.yaml
│           └── observability.yaml
│
├── bootstrap/                       # CLUSTER SETUP
│   ├── argocd/                      # ArgoCD installation
│   │   ├── namespace.yaml
│   │   ├── kustomization.yaml
│   │   └── argocd-cm-patch.yaml
│   └── cluster-addons/              # Required cluster components
│       ├── aws-load-balancer-controller.yaml
│       ├── external-secrets.yaml
│       └── kustomization.yaml
│
├── infrastructure/                  # CLUSTER-WIDE RESOURCES
│   └── base/
│       ├── namespaces.yaml          # Kubernetes namespaces
│       ├── resource-quotas.yaml     # Resource limits
│       ├── network-policies.yaml    # Network security
│       └── cluster-secret-store.yaml # AWS Secrets connection
│
└── observability/                   # MONITORING STACK
    └── base/
        ├── namespace.yaml
        ├── prometheus/              # Metrics collection
        ├── grafana/                 # Dashboards
        ├── jaeger/                  # Distributed tracing
        └── otel-collector/          # OpenTelemetry
```

### Folder Purposes

| Folder | Purpose | When to Modify |
|--------|---------|----------------|
| `charts/` | Reusable templates | Adding new K8s resource types |
| `apps/overlays/` | Service configs per env | Changing service settings |
| `argocd/` | ArgoCD app definitions | Adding new services |
| `bootstrap/` | Initial cluster setup | One-time cluster setup |
| `infrastructure/` | Cluster-wide resources | Changing namespaces, quotas |
| `observability/` | Monitoring stack | Modifying monitoring |

---

## How It All Works Together

### Deployment Flow

```
+---------------------------------------------------------------------+
|                         DEPLOYMENT FLOW                              |
+---------------------------------------------------------------------+

1. DEVELOPER PUSHES CODE
   └── gearify-catalog-svc (main branch)
              |
              v
2. GITHUB ACTIONS BUILDS IMAGE
   └── docker build --> push to ECR
   └── Image: gearify-catalog-svc:abc123
              |
              v
3. GITHUB ACTIONS UPDATES GITOPS
   └── Updates apps/overlays/dev/catalog-svc/values.yaml
   └── image.tag: abc123
              |
              v
4. ARGOCD DETECTS CHANGE
   └── Watches this gearify-gitops repository
   └── Sees new image tag in values.yaml
              |
              v
5. ARGOCD SYNCS TO KUBERNETES
   └── helm template + kubectl apply
   └── New pods start with new image
              |
              v
6. KUBERNETES ROLLING UPDATE
   └── Old pods gracefully shutdown
   └── New pods start and pass health checks
   └── Zero downtime deployment complete!
```

### What ArgoCD Does

```
+---------------------------------------------------------------------+
|                           ARGOCD                                     |
|                                                                      |
|   +------------------+         +------------------+                  |
|   |   Git Repo       |         |   Kubernetes     |                  |
|   |   (Desired)      |  --?--  |   (Actual)       |                  |
|   |                  |  same?  |                  |                  |
|   | replicas: 3      |         | replicas: 2      |                  |
|   +------------------+         +------------------+                  |
|            |                          |                              |
|            |        NOT SAME!         |                              |
|            |                          |                              |
|            +----------+---------------+                              |
|                       v                                              |
|               +---------------+                                      |
|               |  SYNC ACTION  |                                      |
|               |               |                                      |
|               | kubectl scale |                                      |
|               | --replicas=3  |                                      |
|               +---------------+                                      |
|                                                                      |
|   Result: Kubernetes now matches Git (replicas: 3)                   |
+---------------------------------------------------------------------+
```

---

## Helm Charts Explained

### Chart.yaml - Chart Metadata

```yaml
# charts/gearify-service/Chart.yaml

apiVersion: v2              # Helm chart version (always v2 for Helm 3)
name: gearify-service       # Chart name
description: Base chart for Gearify microservices
type: application           # application or library
version: 1.0.0              # Chart version (for the chart itself)
appVersion: "1.0.0"         # Application version (your app)
```

### values.yaml - Default Configuration

```yaml
# charts/gearify-service/values.yaml

# These are DEFAULT values
# Each service overrides what it needs

name: ""                    # Required: service name
port: 8080                  # Default port

replicas: 1                 # Default replicas

image:
  repository: ""            # Required: ECR image URL
  tag: "latest"             # Default tag
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 50m                # Minimum CPU
    memory: 128Mi           # Minimum memory
  limits:
    cpu: 250m               # Maximum CPU
    memory: 256Mi           # Maximum memory
```

### Templates - Kubernetes Resources

Each file in `templates/` generates a Kubernetes resource:

| Template | Creates | Purpose |
|----------|---------|---------|
| `deployment.yaml` | Deployment | Runs your container pods |
| `service.yaml` | Service | Internal network endpoint |
| `serviceaccount.yaml` | ServiceAccount | Identity for AWS access |
| `configmap.yaml` | ConfigMap | Environment variables |
| `externalsecret.yaml` | ExternalSecret | Pulls secrets from AWS |
| `hpa.yaml` | HorizontalPodAutoscaler | Auto-scaling |
| `pdb.yaml` | PodDisruptionBudget | High availability |
| `ingress.yaml` | Ingress | External HTTP access |
| `servicemonitor.yaml` | ServiceMonitor | Prometheus scraping |

### Template Syntax Cheat Sheet

```yaml
# Access values from values.yaml
{{ .Values.name }}
{{ .Values.image.repository }}

# Built-in variables
{{ .Release.Name }}        # Helm release name
{{ .Release.Namespace }}   # Kubernetes namespace
{{ .Chart.Name }}          # Chart name

# Conditionals
{{- if .Values.ingress.enabled }}
  # Only include this if ingress is enabled
{{- end }}

# Loops
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}

# Default values
{{ .Values.port | default 8080 }}

# Quote strings
{{ .Values.name | quote }}
```

---

## ArgoCD Configuration

### Projects - Grouping Applications

Projects organize applications and control access:

```yaml
# argocd/projects/gearify-apps.yaml

apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: gearify-apps
  namespace: argocd
spec:
  description: Gearify microservices

  # Where apps can be deployed
  destinations:
    - namespace: gearify-dev
      server: https://kubernetes.default.svc

  # What Git repos are allowed
  sourceRepos:
    - https://github.com/your-org/gearify-gitops.git
```

### ApplicationSet - Deploy All Services

One ApplicationSet creates apps for ALL services:

```yaml
# argocd/applicationsets/services-appset.yaml

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: gearify-services
spec:
  generators:
    - matrix:
        generators:
          # Environments
          - list:
              elements:
                - env: dev
                  namespace: gearify-dev

          # Services (each becomes an app)
          - list:
              elements:
                - service: catalog-svc
                - service: order-svc
                - service: payment-svc
                # ... all 13 services

  template:
    metadata:
      name: "{{service}}-{{env}}"     # e.g., catalog-svc-dev
    spec:
      source:
        path: charts/gearify-service
        helm:
          valueFiles:
            - ../../apps/overlays/{{env}}/{{service}}/values.yaml
      destination:
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true       # Delete removed resources
          selfHeal: true    # Revert manual changes
```

This single file creates 13 ArgoCD applications!

---

## Adding a New Service

### Step 1: Create Values File

```bash
mkdir -p apps/overlays/dev/my-new-svc
```

Create `apps/overlays/dev/my-new-svc/values.yaml`:

```yaml
name: my-new-svc
port: 5020

image:
  repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/gearify-my-new-svc
  tag: dev-latest

replicas: 1

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

env:
  ASPNETCORE_ENVIRONMENT: Development

serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/gearify-dev-my-new-svc-role
```

### Step 2: Create Chart.yaml

Create `apps/overlays/dev/my-new-svc/Chart.yaml`:

```yaml
apiVersion: v2
name: my-new-svc
version: 1.0.0
dependencies:
  - name: gearify-service
    version: "1.0.0"
    repository: "file://../../../charts/gearify-service"
```

### Step 3: Add to ApplicationSet

Edit `argocd/applicationsets/services-appset.yaml`:

```yaml
- service: my-new-svc   # Add this line
```

### Step 4: Commit and Push

```bash
git add .
git commit -m "Add my-new-svc to dev environment"
git push
```

ArgoCD automatically detects and deploys the new service!

---

## Common Commands

### Helm Commands

```bash
# Navigate to chart directory
cd gearify-gitops/charts/gearify-service

# Validate chart syntax
helm lint .

# Preview generated YAML (without deploying)
helm template my-release . -f ../../apps/overlays/dev/catalog-svc/values.yaml

# Preview with debug output
helm template my-release . \
  -f ../../apps/overlays/dev/catalog-svc/values.yaml \
  --debug

# Install/upgrade manually (not GitOps way)
helm upgrade --install catalog-svc . \
  -f ../../apps/overlays/dev/catalog-svc/values.yaml \
  -n gearify-dev

# List installed releases
helm list -A

# Uninstall a release
helm uninstall catalog-svc -n gearify-dev
```

### ArgoCD Commands

```bash
# Port-forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login via CLI
argocd login localhost:8080

# List all applications
argocd app list

# Get application details
argocd app get catalog-svc-dev

# View what will change (diff)
argocd app diff catalog-svc-dev

# Sync an application
argocd app sync catalog-svc-dev

# Force sync (ignore warnings)
argocd app sync catalog-svc-dev --force

# Rollback to previous version
argocd app rollback catalog-svc-dev 1
```

### Kubectl Commands

```bash
# View all pods in namespace
kubectl get pods -n gearify-dev

# View all services
kubectl get svc -n gearify-dev

# View pod logs
kubectl logs -f deployment/catalog-svc -n gearify-dev

# Describe pod (troubleshooting)
kubectl describe pod -l app=catalog-svc -n gearify-dev

# Port-forward for local testing
kubectl port-forward svc/catalog-svc 5001:5001 -n gearify-dev

# Execute command in pod
kubectl exec -it deployment/catalog-svc -n gearify-dev -- /bin/sh
```

---

## Troubleshooting

### ArgoCD Shows "OutOfSync"

```bash
# Check what's different
argocd app diff catalog-svc-dev

# Force sync
argocd app sync catalog-svc-dev --force
```

### Pod Won't Start

```bash
# Check pod status
kubectl get pods -n gearify-dev

# Check events
kubectl describe pod <pod-name> -n gearify-dev

# Common issues:
# - ImagePullBackOff: Wrong image name or no ECR access
# - CrashLoopBackOff: App crashes, check logs
# - Pending: No resources available
```

### External Secrets Not Working

```bash
# Check ExternalSecret status
kubectl get externalsecrets -n gearify-dev

# Describe for errors
kubectl describe externalsecret catalog-svc -n gearify-dev

# Check if ClusterSecretStore exists
kubectl get clustersecretstore
```

### Helm Template Error

```bash
# Debug with verbose output
helm template test charts/gearify-service \
  -f apps/overlays/dev/catalog-svc/values.yaml \
  --debug 2>&1 | head -50
```

---

## Quick Reference

| Term | What It Is |
|------|------------|
| **GitOps** | Using Git as single source of truth for infrastructure |
| **Helm** | Package manager for Kubernetes (templates + values) |
| **Chart** | A Helm package (folder with templates) |
| **Values** | Variables that customize a Helm chart |
| **ArgoCD** | Tool that syncs Git to Kubernetes automatically |
| **Application** | ArgoCD resource representing a deployed service |
| **ApplicationSet** | ArgoCD resource that creates multiple applications |
| **Sync** | ArgoCD applying Git changes to cluster |
| **Kustomize** | Another way to customize YAML (used in bootstrap/) |

---

## Environment Configuration

| Environment | Namespace | Auto-Sync | Replicas | Resources |
|-------------|-----------|-----------|----------|-----------|
| Dev | gearify-dev | Yes | 1 | Low |
| Staging | gearify-staging | Yes | 2 | Medium |
| Production | gearify-prod | No (manual) | 2-10 (HPA) | High |

---

## Quick Start

### Prerequisites

- EKS cluster running
- kubectl configured
- Helm installed

### 1. Bootstrap ArgoCD

```bash
kubectl apply -k bootstrap/argocd/
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

### 2. Apply Infrastructure

```bash
kubectl apply -k infrastructure/base/
```

### 3. Apply ArgoCD Applications

```bash
kubectl apply -f argocd/projects/
kubectl apply -f argocd/applicationsets/
```

### 4. Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
# Username: admin
# Password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## Next Steps

1. Deploy EKS cluster using `gearify-infrastructure`
2. Push this repo to GitHub
3. Bootstrap ArgoCD
4. Watch services deploy automatically!

For infrastructure setup, see the [Infrastructure Plan](../documentation/infra/INFRASTRUCTURE_PLAN.md).
