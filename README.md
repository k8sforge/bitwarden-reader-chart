# bitwarden-reader Helm Chart Repository

![Auto Tag Release](https://github.com/k8sforge/bitwarden-reader-chart/actions/workflows/chart-releaser.yml/badge.svg)

This is a Helm chart repository for the [k8s-bitwarden-reader](https://github.com/platformfuzz/k8s-bitwarden-reader) Helm chart. k8s-bitwarden-reader is a web application that reads and displays secrets synced from Bitwarden Secrets Manager to Kubernetes.

## Quick Start

### Add the Repository

```bash
helm repo add bitwarden-reader https://k8sforge.github.io/bitwarden-reader-chart
helm repo update
```

### Install the Chart

```bash
helm install my-bitwarden-reader bitwarden-reader/bitwarden-reader --version <version> \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set app.secretNames[0]=example-secret
```

### List Available Versions

```bash
helm search repo bitwarden-reader/bitwarden-reader --versions
```

## Chart Information

- **Chart Name**: `bitwarden-reader`
- **Repository**: `https://k8sforge.github.io/bitwarden-reader-chart`
- **Latest Version**: See [index.yaml](index.yaml) for available versions

## Documentation

For complete documentation, configuration options, and examples, visit the [main repository](https://github.com/k8sforge/bitwarden-reader-chart).

## Alternative: OCI Installation

This chart is also available via OCI registry:

```bash
helm install my-bitwarden-reader \
  oci://ghcr.io/k8sforge/bitwarden-reader-chart/bitwarden-reader \
  --version <version> \
  --namespace bitwarden-secrets \
  --create-namespace \
  --set app.secretNames[0]=example-secret
```

## Support

- **Issues**: [GitHub Issues](https://github.com/k8sforge/bitwarden-reader-chart/issues)
- **Source Code**: [GitHub Repository](https://github.com/k8sforge/bitwarden-reader-chart)
- **Application Source**: [k8s-bitwarden-reader](https://github.com/platformfuzz/k8s-bitwarden-reader)
