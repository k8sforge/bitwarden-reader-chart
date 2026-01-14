# Usage Examples

This document provides practical examples of how to use the bitwarden-reader Helm chart.

## Table of Contents

1. [Basic Installation](#basic-installation)
2. [Installation with Custom Values](#installation-with-custom-values)
3. [Installation from OCI Registry](#installation-from-oci-registry)
4. [Multiple Secrets Configuration](#multiple-secrets-configuration)
5. [AWS ALB Example](#aws-alb-example)
6. [RBAC Configuration](#rbac-configuration)
7. [Resource Management](#resource-management)
8. [Using as a Dependency](#using-as-a-dependency)
9. [Accessing bitwarden-reader](#accessing-bitwarden-reader)
10. [Upgrade and Rollback](#upgrade-and-rollback)

---

## Basic Installation

### Install from Local Chart

```bash
# Clone the repository
git clone https://github.com/k8sforge/bitwarden-reader-chart.git
cd bitwarden-reader-chart

# Install with default values (requires secretNames to be set)
helm install my-bitwarden-reader charts/bitwarden-reader \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set app.secretNames[0]=example-secret
```

### Install with Override Values

```bash
# Install and override specific values
helm install my-bitwarden-reader charts/bitwarden-reader \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set ingress.className=alb \
  --set replicaCount=2 \
  --set ingress.enabled=true \
  --set app.secretNames[0]=example-secret \
  --set app.secretNames[1]=example-secret2
```

---

## Installation with Custom Values

### Create a Custom Values File

Create `my-bitwarden-reader-values.yaml`:

```yaml
# my-bitwarden-reader-values.yaml
global:
  enabled: true

namespace:
  name: "bitwarden-secrets"
  create: false  # Namespace usually exists from Bitwarden operator

image:
  repository: ghcr.io/platformfuzz/k8s-bitwarden-reader
  tag: latest
  pullPolicy: Always

replicaCount: 1

serviceAccount:
  create: true

rbac:
  create: true
  rules:
    - apiGroups: [""]
      resources: ["secrets"]
      verbs: ["get", "list"]
    - apiGroups: ["k8s.bitwarden.com", "bitwarden-secrets-operator.io"]
      resources: ["bitwardensecrets"]
      verbs: ["get", "list", "patch", "update"]

app:
  secretNames:
    - example-secret
    - example-secret2

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: "alb"
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /api/v1/health
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
  hosts:
    - host: ""
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### Install with Custom Values

```bash
# Install with custom values
helm install my-bitwarden-reader charts/bitwarden-reader \
  -f my-bitwarden-reader-values.yaml \
  --namespace bitwarden-secrets \
  --create-namespace
```

---

## Installation from OCI Registry

If the chart is published to an OCI registry (like GitHub Container Registry):

```bash
# For OCI registries, use direct reference (no helm repo add needed)
helm install my-bitwarden-reader \
  oci://ghcr.io/k8sforge/bitwarden-reader-chart/bitwarden-reader \
  --version 0.1.0 \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set app.secretNames[0]=example-secret \
  --set app.secretNames[1]=example-secret2

# Or with custom values
helm install my-bitwarden-reader \
  oci://ghcr.io/k8sforge/bitwarden-reader-chart/bitwarden-reader \
  --version 0.1.0 \
  -f my-bitwarden-reader-values.yaml \
  --namespace bitwarden-secrets \
  --create-namespace
```

---

## Installation from Helm Repository

```bash
# Add the repository
helm repo add bitwarden-reader https://k8sforge.github.io/bitwarden-reader-chart
helm repo update

# Install from repository
helm install my-bitwarden-reader bitwarden-reader/bitwarden-reader \
  --version 0.1.0 \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set app.secretNames[0]=example-secret
```

---

## Multiple Secrets Configuration

### Example: Reading Multiple Secrets

```yaml
app:
  secretNames:
    - example-secret
    - example-secret2
    - api-credentials
    - database-password
```

Install with:

```bash
helm install my-bitwarden-reader charts/bitwarden-reader \
  --namespace bitwarden-secrets \
  --set app.secretNames[0]=example-secret \
  --set app.secretNames[1]=example-secret2 \
  --set app.secretNames[2]=api-credentials
```

---

## Health Check Configuration

### Customize Health Check Probes

```yaml
livenessProbe:
  httpGet:
    path: /api/v1/health
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /api/v1/health
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

---

## AWS ALB Example

### Full ALB Configuration

```yaml
ingress:
  enabled: true
  className: "alb"
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /api/v1/health
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
  hosts:
    - host: ""
      paths:
        - path: /
          pathType: Prefix
```

### ALB with HTTPS

```yaml
ingress:
  enabled: true
  className: "alb"
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/cert-id
    alb.ingress.kubernetes.io/healthcheck-path: /api/v1/health
  hosts:
    - host: reader.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - reader.example.com
      secretName: reader-tls
```

---

## RBAC Configuration

### Custom RBAC Rules

```yaml
rbac:
  create: true
  rules:
    - apiGroups: [""]
      resources: ["secrets"]
      verbs: ["get", "list", "watch"]
    - apiGroups: ["k8s.bitwarden.com", "bitwarden-secrets-operator.io"]
      resources: ["bitwardensecrets"]
      verbs: ["get", "list", "watch", "patch", "update"]
```

### Disable RBAC

```yaml
rbac:
  create: false
```

---

## Resource Management

### Custom Resource Limits

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### High Resource Configuration

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

---

## Using as a Dependency

### Chart.yaml with Dependency

```yaml
apiVersion: v2
name: my-deployment
type: application
version: 1.0.0

dependencies:
  - name: bitwarden-reader
    version: 0.1.0
    repository: https://k8sforge.github.io/bitwarden-reader-chart
    condition: bitwarden-reader.enabled
```

### values.yaml for Parent Chart

```yaml
bitwarden-reader:
  enabled: true
  namespace:
    name: "bitwarden-secrets"
    create: false
  app:
    secretNames:
      - example-secret
      - example-secret2
  ingress:
    enabled: true
    className: "alb"
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
```

### Install with Dependency

```bash
# Update dependencies
helm dependency update

# Install parent chart (which includes bitwarden-reader)
helm install my-deployment . \
  --namespace my-namespace \
  --create-namespace
```

---

## Accessing bitwarden-reader

### Get Ingress URL

```bash
# Get the ingress hostname
kubectl -n bitwarden-secrets get ingress -l app.kubernetes.io/name=bitwarden-reader \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'
```

### Port Forward

```bash
# Port forward to access locally
kubectl port-forward -n bitwarden-secrets svc/my-bitwarden-reader 8080:80
```

Then access at `http://localhost:8080`

### Access via Ingress

```bash
# Get the ingress URL
INGRESS_URL=$(kubectl -n bitwarden-secrets get ingress -l app.kubernetes.io/name=bitwarden-reader \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')

# Access the application
curl http://$INGRESS_URL/api/v1/health
```

---

## Upgrade and Rollback

### Upgrade

```bash
# Upgrade with new values
helm upgrade my-bitwarden-reader charts/bitwarden-reader -f my-bitwarden-reader-values.yaml

# Upgrade with new chart version from OCI registry
helm upgrade my-bitwarden-reader \
  oci://ghcr.io/k8sforge/bitwarden-reader-chart/bitwarden-reader \
  --version 0.2.0 \
  --namespace bitwarden-secrets
```

### Rollback

```bash
# Check release history
helm history my-bitwarden-reader -n bitwarden-secrets

# Rollback to previous version
helm rollback my-bitwarden-reader -n bitwarden-secrets

# Rollback to specific revision
helm rollback my-bitwarden-reader 3 -n bitwarden-secrets
```

---

## Troubleshooting

### Check Pod Status

```bash
# Check pods
kubectl get pods -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader

# Check logs
kubectl logs -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader --tail=100

# Describe pod for events
kubectl describe pod -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader
```

### Check Service

```bash
kubectl get svc -n bitwarden-secrets
kubectl describe svc -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader
```

### Check Ingress

```bash
# Get ingress details
kubectl get ingress -n bitwarden-secrets
kubectl describe ingress -n bitwarden-secrets

# Check ingress events
kubectl get events -n bitwarden-secrets --field-selector involvedObject.kind=Ingress
```

### Check RBAC

```bash
# Check RBAC resources
kubectl get role,rolebinding,serviceaccount -n bitwarden-secrets \
  -l app.kubernetes.io/name=bitwarden-reader

# Verify service account is used
kubectl get pod -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader \
  -o jsonpath='{.items[0].spec.serviceAccountName}'
```

### Check Resource Usage

```bash
# Check resource usage
kubectl top pods -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader

# Check resource limits
kubectl describe pod -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader | grep -A 5 "Limits"
```

### Validate Chart

```bash
# Lint
helm lint charts/bitwarden-reader

# Dry-run
helm install my-bitwarden-reader charts/bitwarden-reader --dry-run --debug -n bitwarden-secrets

# Template rendering
helm template my-bitwarden-reader charts/bitwarden-reader -f values.yaml
```

### Test Health Endpoint

```bash
# Port forward first
kubectl port-forward -n bitwarden-secrets svc/my-bitwarden-reader 8080:80

# Test health endpoint
curl http://localhost:8080/api/v1/health
```

---

## Next Steps

1. **Access bitwarden-reader UI** using the ingress URL or port-forward
2. **Configure RBAC** to match your security requirements
3. **Adjust resources** based on your workload
4. **Customize ingress** for your platform (ALB)
5. **Monitor the application** using Kubernetes metrics
6. **Review logs** for troubleshooting
