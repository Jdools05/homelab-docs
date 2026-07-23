# Kubernetes Namespaces

## Overview

The homelab uses a small set of namespaces to organize cluster resources. Since this is a single-user setup, most application workloads live in the `default` namespace for simplicity.

## Namespace Inventory

| Namespace | Purpose | Status |
|-----------|---------|--------|
| `default` | Application workloads (landing page, mock trading, power playlist, etc.) | Active |
| `kube-system` | k3s system components (CoreDNS, Traefik, metrics server) | Active |
| `longhorn-system` | Longhorn distributed storage | Active |
| `metallb-system` | MetalLB load balancer | Active |
| `monitoring` | Prometheus monitoring stack | Active |

## Namespace Details

**`default`** — All user services run here: landing page (NGINX), mock trading (React Native web build), Power Playlist API + Web, AOA Marching Cubes (Java 3D visualization), and the dynamic 404 handler. Single-tenant homelab doesn't need strict namespace isolation.

**`kube-system`** — Kubernetes system-critical components: CoreDNS, k3s local-path-provisioner, metrics server, Traefik ingress controller, and MetalLB LoadBalancer pods.

**`longhorn-system`** — Longhorn manager (DaemonSet), CSI plugins, engine images, and the Longhorn UI for storage management.

**`metallb-system`** — MetalLB controller (on control plane) and speaker (DaemonSet on all nodes). Handles Layer 2 IP advertisement.

**`monitoring`** — Prometheus server, Alertmanager, Node Exporter (DaemonSet), kube-state-metrics, and Pushgateway for batch job metrics.

## Current State

- **No Network Policies**: All pods can communicate across namespaces freely
- **No Resource Quotas**: No CPU/memory limits per namespace

---

*Namespace documentation. Specific resource counts have been generalized.*
