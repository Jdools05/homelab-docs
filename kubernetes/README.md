# Kubernetes Cluster Documentation

This directory contains documentation for the k3s Kubernetes cluster running on the homelab infrastructure. The cluster provides containerized service hosting with distributed storage, monitoring, and external access via Traefik ingress controller.

## Contents

- **[architecture.md](./architecture.md)** - Cluster topology, node roles, networking model, and design decisions
- **[ingress.md](./ingress.md)** - Traefik ingress controller setup, TLS termination, and routing rules
- **[namespaces.md](./namespaces.md)** - Namespace layout, purpose, and resource allocation
- **[persistent-storage.md](./persistent-storage.md)** - Longhorn storage classes, PVCs, and volume management
- **[manifests/](./manifests/)** - Current Kubernetes resource YAML files (exported from cluster)

## Cluster Overview

| Component | Version/Status |
|-----------|----------------|
| **Kubernetes** | k3s v1.34.5 |
| **Nodes** | 3 (1 control + 2 workers) |
| **OS** | Ubuntu 24.04 LTS |
| **Container Runtime** | containerd 2.1.5-k3s1 |
| **Ingress Controller** | Traefik v3.6 |
| **Load Balancer** | MetalLB (Layer 2 mode) |
| **Storage** | Longhorn v1.7 (distributed block storage) |
| **Monitoring** | Prometheus Stack (Prometheus, Alertmanager, Node Exporter) |

## Service Exposure

All external services are exposed via Traefik ingress controller with the following domains:

| Domain | Service | Type |
|--------|---------|------|
| `jdools.com` | Landing Page (NGINX) | Static HTML/JS/CSS |
| `mock-trading.jdools.com` | Mock Trading App (React Native web build) | SPA served by NGINX |
| `power-playlist.jdools.com` | Power Playlist API + Web | Node.js API + Next.js frontend |
| `aoa-marching-cubes.jdools.com` | AOA Marching Cubes (Java 3D visualization) | Static web app |

## Security Notice

All documentation in this repository has been sanitized for public viewing. Specific internal IPs, cluster-internal service addresses, and sensitive configuration details have been generalized or omitted. The `_raw-data/kubernetes/` directory contains unsanitized operational data for reference only.
