# Helm Templates Guide

This document explains each template in the `gearify-service` Helm chart.

---

## Template Files Overview

```
charts/gearify-service/templates/
├── deployment.yaml      # Runs your application containers
├── service.yaml         # Network endpoint for the app
├── serviceaccount.yaml  # Identity for AWS access (IRSA)
├── configmap.yaml       # Environment variables
├── externalsecret.yaml  # Secrets from AWS Secrets Manager
├── hpa.yaml             # Horizontal Pod Autoscaler
├── pdb.yaml             # Pod Disruption Budget
├── ingress.yaml         # External HTTP access
└── servicemonitor.yaml  # Prometheus metrics scraping
```

---

## 1. deployment.yaml

**What it does:** Creates pods that run your application.

**Key concepts:**
- **Pod**: The smallest unit in Kubernetes, contains one or more containers
- **Deployment**: Manages pods, handles updates, restarts failed pods
- **ReplicaSet**: Ensures the right number of pods are running

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}              # e.g., "catalog-svc"
  labels:
    app: {{ .Values.name }}
spec:
  replicas: {{ .Values.replicas }}      # How many pods to run

  selector:
    matchLabels:
      app: {{ .Values.name }}           # How to find our pods

  template:                             # Pod template
    metadata:
      labels:
        app: {{ .Values.name }}
    spec:
      serviceAccountName: {{ .Values.name }}  # For AWS access

      containers:
        - name: {{ .Values.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          ports:
            - containerPort: {{ .Values.port }}

          # Health checks
          livenessProbe:                # Is the app alive?
            httpGet:
              path: /health
              port: {{ .Values.port }}
            initialDelaySeconds: 30
            periodSeconds: 10

          readinessProbe:               # Is the app ready for traffic?
            httpGet:
              path: /health
              port: {{ .Values.port }}
            initialDelaySeconds: 5
            periodSeconds: 5

          # Resource limits
          resources:
            requests:                   # Minimum guaranteed
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}
            limits:                     # Maximum allowed
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}

          # Environment variables from ConfigMap
          envFrom:
            - configMapRef:
                name: {{ .Values.name }}-config

          # Environment variables from Secrets
          env:
            - name: DB_CONNECTION_STRING
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.name }}-secrets
                  key: DB_CONNECTION_STRING
```

**Visual representation:**

```
+----------------------------------------------------------+
|                      DEPLOYMENT                           |
|  Name: catalog-svc                                        |
|  Replicas: 2                                              |
|                                                           |
|  +------------------------+  +------------------------+   |
|  |         POD 1          |  |         POD 2          |   |
|  |  catalog-svc-abc123    |  |  catalog-svc-def456    |   |
|  |                        |  |                        |   |
|  |  +------------------+  |  |  +------------------+  |   |
|  |  |    CONTAINER     |  |  |  |    CONTAINER     |  |   |
|  |  | catalog-svc:v1.0 |  |  |  | catalog-svc:v1.0 |  |   |
|  |  |    Port: 5001    |  |  |  |    Port: 5001    |  |   |
|  |  +------------------+  |  |  +------------------+  |   |
|  +------------------------+  +------------------------+   |
+----------------------------------------------------------+
```

---

## 2. service.yaml

**What it does:** Creates a stable network endpoint to access pods.

**Why needed:** Pods have random IPs that change. Service provides a stable address.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.name }}
spec:
  type: ClusterIP                 # Internal only (default)

  selector:
    app: {{ .Values.name }}       # Find pods with this label

  ports:
    - port: {{ .Values.port }}    # Service port (what others connect to)
      targetPort: {{ .Values.port }}  # Pod port (where app listens)
      protocol: TCP
```

**Visual representation:**

```
+-------------------------------------------------------------------+
|                         KUBERNETES CLUSTER                         |
|                                                                    |
|   Other Pods                        catalog-svc Pods               |
|   +--------+                                                       |
|   | api-gw |                     +--------+  +--------+            |
|   +--------+                     | Pod 1  |  | Pod 2  |            |
|       |                          | :5001  |  | :5001  |            |
|       |                          +--------+  +--------+            |
|       |                               ^          ^                 |
|       |                               |          |                 |
|       |        +------------------+   |          |                 |
|       +------> |     SERVICE      |---+----------+                 |
|                | catalog-svc:5001 |                                |
|                | (ClusterIP)      |                                |
|                +------------------+                                |
|                                                                    |
|   Traffic to "catalog-svc:5001" is load-balanced to pods          |
+-------------------------------------------------------------------+
```

---

## 3. serviceaccount.yaml

**What it does:** Gives the pod an identity for AWS access (IRSA).

**Why needed:** Pods need AWS permissions (S3, DynamoDB, etc.) without hardcoding credentials.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.name }}
  annotations:
    # This annotation links to an AWS IAM role
    eks.amazonaws.com/role-arn: {{ .Values.serviceAccount.roleArn }}
```

**How IRSA works:**

```
+-------------------------------------------------------------------+
|                            IRSA FLOW                               |
+-------------------------------------------------------------------+

1. ServiceAccount has annotation with IAM role ARN

2. When pod starts, EKS injects AWS credentials:
   +------------------+
   |       POD        |
   |                  |
   | AWS_ROLE_ARN     |  <-- Injected by EKS
   | AWS_WEB_ID_TOKEN |  <-- Injected by EKS
   +------------------+

3. AWS SDK automatically uses these to assume the role

4. Pod can now access AWS services allowed by the role:

   catalog-svc role allows:
   - DynamoDB: Read/Write to products table
   - SNS: Publish to catalog-events topic
   - S3: Read from product-images bucket
```

---

## 4. configmap.yaml

**What it does:** Stores non-sensitive configuration as environment variables.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.name }}-config
data:
  ASPNETCORE_ENVIRONMENT: {{ .Values.env.ASPNETCORE_ENVIRONMENT }}
  LOG_LEVEL: {{ .Values.env.LOG_LEVEL | default "Information" }}
  SERVICE_NAME: {{ .Values.name }}
```

**Usage in values.yaml:**

```yaml
env:
  ASPNETCORE_ENVIRONMENT: Development
  LOG_LEVEL: Debug
  REDIS_HOST: redis.gearify-dev.svc.cluster.local
```

---

## 5. externalsecret.yaml

**What it does:** Pulls secrets from AWS Secrets Manager into Kubernetes.

**Why:** Secrets shouldn't be in Git. External Secrets Operator syncs them from AWS.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ .Values.name }}-secrets
spec:
  refreshInterval: 1h              # How often to check for updates

  secretStoreRef:
    name: aws-secrets-manager      # Reference to ClusterSecretStore
    kind: ClusterSecretStore

  target:
    name: {{ .Values.name }}-secrets  # Kubernetes Secret to create

  data:
    # Each item pulls one secret from AWS
    - secretKey: DB_CONNECTION_STRING     # Key in K8s Secret
      remoteRef:
        key: gearify/dev/database/main    # Path in AWS Secrets Manager
        property: connection_string       # JSON property to extract

    - secretKey: JWT_SECRET
      remoteRef:
        key: gearify/dev/jwt
        property: secret
```

**Visual flow:**

```
+-------------------------------------------------------------------+
|                    EXTERNAL SECRETS FLOW                           |
+-------------------------------------------------------------------+

AWS Secrets Manager                     Kubernetes
+-------------------------+             +-------------------------+
| gearify/dev/database    |             | Secret: catalog-secrets |
| {                       |   SYNC      | data:                   |
|   "connection_string":  | -------->>  |   DB_CONNECTION_STRING: |
|   "Host=rds.aws..."     |             |   "Host=rds.aws..."     |
| }                       |             |                         |
+-------------------------+             +-------------------------+
                                               |
                                               v
                                        +-------------+
                                        |     POD     |
                                        | env:        |
                                        | DB_CONN=... |
                                        +-------------+
```

---

## 6. hpa.yaml

**What it does:** Automatically scales pods based on CPU/memory usage.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Values.name }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Values.name }}

  minReplicas: {{ .Values.autoscaling.minReplicas }}   # Minimum pods
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}   # Maximum pods

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale up when CPU > 70%

    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80    # Scale up when memory > 80%
```

**Scaling behavior:**

```
CPU Usage     Pods    Action
---------     ----    ------
  20%           2     Do nothing (below 70%)
  75%           2     Scale up to 3
  80%           3     Scale up to 4
  50%           4     Scale down to 3 (below 70%)
  30%           3     Scale down to 2 (minimum)
```

---

## 7. pdb.yaml

**What it does:** Ensures high availability during updates or node maintenance.

**Why:** Prevents all pods from being killed at once.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ .Values.name }}
spec:
  minAvailable: 1                    # At least 1 pod must be running
  # OR
  # maxUnavailable: 1                # At most 1 pod can be down

  selector:
    matchLabels:
      app: {{ .Values.name }}
```

**Example scenario:**

```
Node Maintenance with 3 pods:

Without PDB:                    With PDB (minAvailable: 1):
--------------                  ---------------------------
Step 1: Kill pod 1              Step 1: Kill pod 1
Step 2: Kill pod 2              Step 2: Wait for pod 1 to restart
Step 3: Kill pod 3              Step 3: Kill pod 2
        |                       Step 4: Wait for pod 2 to restart
        v                       Step 5: Kill pod 3
   ALL DOWN!                           |
   No availability!                    v
                                Always at least 1 running!
```

---

## 8. ingress.yaml

**What it does:** Exposes service to external traffic via AWS ALB.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.name }}
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  rules:
    - host: {{ .Values.ingress.host }}    # e.g., api.gearify.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Values.name }}
                port:
                  number: {{ .Values.port }}
```

**Traffic flow:**

```
+-------------------------------------------------------------------+
|                       INGRESS TRAFFIC FLOW                         |
+-------------------------------------------------------------------+

Internet
    |
    v
+---------------+
| AWS ALB       |   (Created by Ingress)
| api.gearify.com
+---------------+
    |
    v
+---------------+
| Ingress       |   (Routes to Service)
+---------------+
    |
    v
+---------------+
| Service       |   (Load balances to Pods)
| catalog-svc   |
+---------------+
    |
    v
+-------+-------+
| Pod 1 | Pod 2 |
+-------+-------+
```

---

## 9. servicemonitor.yaml

**What it does:** Tells Prometheus to scrape metrics from your service.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Values.name }}
spec:
  selector:
    matchLabels:
      app: {{ .Values.name }}

  endpoints:
    - port: http
      path: /metrics              # Endpoint exposing Prometheus metrics
      interval: 30s               # Scrape every 30 seconds
```

**Metrics flow:**

```
+-------------------------------------------------------------------+
|                      METRICS COLLECTION                            |
+-------------------------------------------------------------------+

+-------------+     GET /metrics     +-------------+
|             | <------------------- |             |
| catalog-svc |                      | Prometheus  |
|             | ----------------->>> |             |
+-------------+     Metrics data     +-------------+
                                            |
                                            v
                                     +-------------+
                                     |   Grafana   |
                                     | (Dashboards)|
                                     +-------------+
```

---

## Complete values.yaml Reference

```yaml
# Service identity
name: catalog-svc
port: 5001

# Container image
image:
  repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/gearify-catalog-svc
  tag: dev-latest
  pullPolicy: IfNotPresent

# Scaling
replicas: 1
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 70

# Resources
resources:
  requests:
    cpu: 50m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

# Environment variables (non-sensitive)
env:
  ASPNETCORE_ENVIRONMENT: Development
  LOG_LEVEL: Information

# AWS Service Account (IRSA)
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/gearify-dev-catalog-svc-role

# Secrets from AWS Secrets Manager
externalSecrets:
  enabled: true
  data:
    - secretKey: DB_CONNECTION_STRING
      remoteRef:
        key: gearify/dev/database/main
        property: connection_string

# Health checks
probes:
  liveness:
    path: /health
    initialDelaySeconds: 30
  readiness:
    path: /health
    initialDelaySeconds: 5

# Ingress (external access)
ingress:
  enabled: false
  host: catalog.gearify.com

# Prometheus metrics
serviceMonitor:
  enabled: true
  path: /metrics
  interval: 30s
```

---

## Testing Templates Locally

```bash
# Navigate to chart
cd gearify-gitops/charts/gearify-service

# Validate syntax
helm lint .

# Render templates with test values
helm template test-release . \
  -f ../../apps/overlays/dev/catalog-svc/values.yaml

# Render specific template
helm template test-release . \
  -f ../../apps/overlays/dev/catalog-svc/values.yaml \
  -s templates/deployment.yaml

# Debug mode (shows all computed values)
helm template test-release . \
  -f ../../apps/overlays/dev/catalog-svc/values.yaml \
  --debug
```
