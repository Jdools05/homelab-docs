# Kubernetes Cluster Architecture

## Overview

The homelab runs a k3s cluster (v1.34.5) with 3 nodes: 1 control plane and 2 workers, all running Ubuntu 24.04 on Proxmox VMs. The cluster uses Longhorn for persistent storage, Traefik for ingress routing, and MetalLB for LoadBalancer service exposure.

## Cluster Topology

```
┌─────────────────────────────────────────────────────┐
│              k3s Cluster (v1.34.5)                   │
│                                                      │
│  ┌──────────────────┐    ┌──────────────────┐       │
│  │ Control Plane    │    │ Worker Node #1   │       │
│  │ (k8s-control-1)  │    │ (k8s-worker-1)   │       │
│  │                  │    │                  │       │
│  │ - etcd, API srv  │    │ - App pods       │       │
│  │ - Traefik (LB)   │    │ - Longhorn agent │       │
│  └──────────────────┘    └──────────────────┘       │
│                                                      │
│  ┌──────────────────┐                                │
│  │ Worker Node #2   │                                │
│  │ (k8s-worker-2)   │                                │
│  │ - App pods       │                                │
│  │ - Longhorn agent │                                │
│  └──────────────────┘                                │
└─────────────────────────────────────────────────────┘
```

## Node Specifications

| Node | VM RAM | Disk | Role | Internal IP |
|------|--------|------|------|-------------|
| k8s-control-1 | 16 GB | 128 GB (thin) | control-plane | `10.10.x.10` |
| k8s-worker-1 | 32 GB | 128 GB (thin) | worker | `10.10.x.20` |
| k8s-worker-2 | 32 GB | 128 GB (thin) | worker | `10.10.x.21` |

## Networking Model

- **Pod CIDR**: `10.42.0.0/16` (k3s default Flannel CNI)
- **Service CIDR**: `10.43.0.0/16`
- **External access flow**: Internet → OPNsense NAT → MetalLB IP (`10.10.x.100`) → Traefik Ingress → Service → Pod

## Key Components

| Component | Purpose |
|-----------|---------|
| **etcd** | Cluster state (control plane only) |
| **CoreDNS** | Internal service DNS resolution |
| **Longhorn** | Distributed block storage with 2x replication |
| **Traefik v3.6** | Ingress controller (handles routing + TLS termination when configured) |
| **MetalLB** | Layer 2 load balancer for external IP assignment |
| **Prometheus Stack** | Monitoring (Prometheus, Alertmanager, Node Exporter) |

## Resource Usage

- **Control plane**: ~8 GB RAM reserved for system components (etcd, API server, Traefik)
- **Worker nodes**: ~24 GB RAM available for workloads each
- **Storage**: Longhorn volumes allocated from the 30 TB MainThin pool on Proxmox host

## High Availability

- **Current state**: Single control plane node (no etcd HA)
- **Scaling**: Worker VMs can be added — Longhorn replicates data automatically across nodes

---

*Cluster architecture documentation. Specific internal IPs and network ranges have been generalized.*
