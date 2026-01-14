# bitwarden-reader Helm Chart

![Chart Releaser](https://github.com/k8sforge/bitwarden-reader-chart/actions/workflows/chart-releaser.yml/badge.svg)

A Helm chart for deploying k8s-bitwarden-reader on Kubernetes. k8s-bitwarden-reader is a web application that reads and displays secrets synced from Bitwarden Secrets Manager to Kubernetes.

---

## Overview

This is a **reusable Helm chart repository**. The chart is versioned and published so it can be referenced by other repositories that deploy k8s-bitwarden-reader.

This chart includes:

- **Complete Resource Coverage** - ServiceAccount, RBAC (Role/RoleBinding), Deployment, Service, and Ingress
- **ALB Ingress Support** - Full AWS ALB configuration with health checks
- **Configurable Health Checks** - Platform-agnostic health check configuration for ingress
- **RBAC Support** - ServiceAccount, Role, and RoleBinding with BitwardenSecret CRD permissions
- **Resource Management** - Configurable resource defaults for k8s-bitwarden-reader components

---

## Chart Details

- **Chart Name**: `bitwarden-reader`
- **Chart Version**: See [Chart.yaml](charts/bitwarden-reader/Chart.yaml#L5)
- **App Version**: `latest` (k8s-bitwarden-reader image tag)

---

## Distribution

This chart is published in two formats:

- **OCI (ghcr.io)** – modern, registry-based installs
- **Helm repository (GitHub Pages)** – classic `helm repo add` workflow

Both distributions publish the same chart versions.

---

## Quick Start

### Install via OCI (recommended)

```bash
helm install my-bitwarden-reader \
  oci://ghcr.io/k8sforge/bitwarden-reader-chart/bitwarden-reader \
  --version 0.1.0 \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set app.secretNames[0]=example-secret \
  --set app.secretNames[1]=example-secret2
```

If the registry is private:

```bash
helm registry login ghcr.io
```

---

### Install via Helm Repository (GitHub Pages)

```bash
helm repo add bitwarden-reader https://k8sforge.github.io/bitwarden-reader-chart
helm repo update

helm install my-bitwarden-reader bitwarden-reader/bitwarden-reader --version 0.1.0 \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set app.secretNames[0]=example-secret
```

---

### Install from Source (local development)

```bash
# Install with default values
helm install my-bitwarden-reader charts/bitwarden-reader \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set app.secretNames[0]=example-secret
```

---

## Prerequisites

- Kubernetes 1.20+
- `kubectl` configured
- Helm 3.x
- Ingress controller (AWS ALB, nginx, traefik, or platform-specific)
- Bitwarden Secrets Manager Operator installed
- BitwardenSecret CRDs configured
- Secrets synced to the target namespace

---

## Configuration

The following table lists the main configurable parameters:

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `global.enabled` | Enable the chart | `true` |
| `namespace.name` | Namespace name | `bitwarden-secrets` |
| `namespace.create` | Create namespace | `false` |
| `image.repository` | Image repository | `ghcr.io/platformfuzz/k8s-bitwarden-reader` |
| `image.tag` | Image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `Always` |
| `replicaCount` | Number of replicas | `1` |
| `app.secretNames` | List of secret names to read | `[]` |
| `serviceAccount.create` | Create service account | `true` |
| `rbac.create` | Create RBAC resources | `true` |
| `service.type` | Service type | `ClusterIP` |
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.className` | Ingress class name | `alb` |
| `ingress.hosts` | Ingress hosts (empty = accept any) | `[]` |
| `resources.requests.cpu` | CPU request | `100m` |
| `resources.requests.memory` | Memory request | `128Mi` |
| `resources.limits.cpu` | CPU limit | `200m` |
| `resources.limits.memory` | Memory limit | `256Mi` |

See [values.yaml](charts/bitwarden-reader/values.yaml) for the full configuration.

---

## Accessing k8s-bitwarden-reader

### Get k8s-bitwarden-reader URL

```bash
# Get the ingress URL
kubectl -n bitwarden-secrets get ingress -l app.kubernetes.io/name=bitwarden-reader -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'
```

### Port Forward

```bash
# Port forward to access locally
kubectl port-forward -n bitwarden-secrets svc/<release-name> 8080:80
```

Then access at `http://localhost:8080`

---

## Health Checks

Health checks are configured for both the Kubernetes probes and ingress annotations.

### Kubernetes Probes

The chart includes liveness and readiness probes:

```yaml
livenessProbe:
  httpGet:
    path: /api/v1/health
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /api/v1/health
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
```

### Ingress Health Checks

Health check annotations are automatically configured in the default values. The chart includes platform-agnostic health check configuration:

```yaml
ingress:
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /api/v1/health
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
```

For other platforms, you can customize the annotations in `values.yaml`. See [EXAMPLES.md](EXAMPLES.md) for platform-specific ingress configurations.

---

## RBAC Configuration

The chart includes RBAC resources (ServiceAccount, Role, RoleBinding) with default permissions:

```yaml
rbac:
  create: true
  rules:
    - apiGroups: [""]
      resources: ["secrets"]
      verbs: ["get", "list"]
    - apiGroups: ["k8s.bitwarden.com", "bitwarden-secrets-operator.io"]
      resources: ["bitwardensecrets"]
      verbs: ["get", "list", "patch", "update"]
```

These permissions allow k8s-bitwarden-reader to:

- Read Kubernetes secrets synced from Bitwarden
- Read BitwardenSecret CRD status and metadata
- Trigger sync operations on BitwardenSecret CRDs

You can customize the RBAC rules in `values.yaml` to match your security requirements.

---

## Resource Management

Default resource requests and limits are configured:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

These can be adjusted based on your workload requirements.

---

## Using This Chart from Another Repository (Repo B Pattern)

### Example dependency

```yaml
# Chart.yaml
apiVersion: v2
name: my-deployment
type: application
version: 1.0.0

dependencies:
  - name: bitwarden-reader
    version: 0.1.0
    repository: https://k8sforge.github.io/bitwarden-reader-chart
```

Then:

```bash
helm dependency update
helm upgrade --install my-bitwarden-reader . -f values.yaml
```

> Note: Helm 3.8+ supports OCI-based dependencies, but classic repositories are shown here for maximum compatibility.

---

## Versioning and Releases

This chart follows semantic versioning.

To release a new version:

```bash
git tag v0.2.0
git push --tags
```

GitHub Actions will automatically publish the chart to:

- **GHCR (OCI)**
- **GitHub Pages (Helm repo)**

---

## Development

### Lint

```bash
helm lint charts/bitwarden-reader
```

### Dry-run (requires cluster connection)

```bash
helm install my-bitwarden-reader charts/bitwarden-reader --dry-run --debug
```

### Render templates (no cluster required)

```bash
# Render templates without connecting to cluster
helm template my-bitwarden-reader charts/bitwarden-reader

# Or with custom values
helm template my-bitwarden-reader charts/bitwarden-reader -f values.yaml
```

---

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader
kubectl logs -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader
```

### Check Service

```bash
kubectl get svc -n bitwarden-secrets
kubectl describe svc -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader
```

### Check Ingress

```bash
kubectl get ingress -n bitwarden-secrets
kubectl describe ingress -n bitwarden-secrets
```

### Check RBAC

```bash
kubectl get role,rolebinding,serviceaccount -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader
```

### Check Resource Usage

```bash
kubectl top pods -n bitwarden-secrets -l app.kubernetes.io/name=bitwarden-reader
```

---

## License

MIT License. See LICENSE for details.

---

## References

- [k8s-bitwarden-reader Source](https://github.com/platformfuzz/k8s-bitwarden-reader)
- [Helm Documentation](https://helm.sh/docs/)
