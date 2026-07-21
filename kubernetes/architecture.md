# Kubernetes Cluster Architecture

## Overview

The homelab runs a k3s (lightweight Kubernetes) cluster with 3 nodes: 1 control plane node and 2 worker nodes. The cluster is deployed on Proxmox VMs and uses Longhorn for distributed block storage, Traefik for ingress routing, and MetalLB for LoadBalancer service exposure.

## Cluster Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    k3s Cluster (v1.34.5)                    │
│                                                             │
│  ┌─────────────────────┐    ┌─────────────────────┐       │
│  │   Control Plane     │    │   Worker Node #1    │       │
│  │   k8s-control-1     │    │   k8s-worker-1      │       │
│  │                     │    │                     │       │
│  │  - etcd (datastore) │    │  - Application pods │       │
│  │  - API server       │    │  - Longhorn agent   │       │
│  │  - Scheduler        │    │  - Node exporter    │       │
│  │  - Controller mgr   │    │                     │       │
│  │  - Traefik (LB)     │    │                     │       │
│  └─────────────────────┘    └─────────────────────┘       │
│                                                             │
│  ┌─────────────────────┐                                     │
│  │   Worker Node #2    │                                     │
│  │   k8s-worker-2      │                                     │
│  │                     │                                     │
│  │  - Application pods │                                     │
│  │  - Longhorn agent   │                                     │
│  │  - Node exporter    │                                     │
│  └─────────────────────┘                                     │
└─────────────────────────────────────────────────────────────┘
```

## Node Specifications

### Control Plane (k8s-control-1)
| Attribute | Value |
|-----------|-------|
| **VM ID** | 116 |
| **RAM** | 16 GB |
| **Disk** | 128 GB (thin-provisioned on MainThin pool) |
| **OS** | Ubuntu 24.04 LTS |
| **Kernel** | 6.8.0-136-generic |
| **Role** | control-plane, master |
| **Internal IP** | `10.10.x.10` (k8s cluster network) |

### Worker Node #1 (k8s-worker-1)
| Attribute | Value |
|-----------|-------|
| **VM ID** | 119 |
| **RAM** | 32 GB |
| **Disk** | 128 GB (thin-provisioned on MainThin pool) |
| **OS** | Ubuntu 24.04 LTS |
| **Kernel** | 6.8.0-134-generic |
| **Role** | worker |
| **Internal IP** | `10.10.x.20` (k8s cluster network) |

### Worker Node #2 (k8s-worker-2)
| Attribute | Value |
|-----------|-------|
| **VM ID** | 120 |
| **RAM** | 32 GB |
| **Disk** | 128 GB (thin-provisioned on MainThin pool) |
| **OS** | Ubuntu 24.04 LTS |
| **Kernel** | 6.8.0-136-generic |
| **Role** | worker |
| **Internal IP** | `10.10.x.21` (k8s cluster network) |

## Networking Model

### Pod Network
- **CNI Plugin**: k3s default (Flannel or Canal, depending on installation)
- **Pod CIDR**: `10.42.0.0/16` (overlapping with host network - requires NAT)
- **Service CIDR**: `10.43.0.0/16`

### Node-to-Node Communication
- **Kubelet API**: Port 10250/TCP on each node
- **etcd**: Port 2379-2380/TCP (control plane only)
- **API Server**: Port 6443/TCP (control plane, exposed via LoadBalancer)

### External Access Flow
```
Internet → OPNsense (NAT) → MetalLB IP (10.10.x.100) → Traefik Ingress → Service → Pod
```

## Component Overview

### Core Components
| Component | Purpose | Location |
|-----------|---------|----------|
| **etcd** | Distributed key-value store for cluster state | Control plane only |
| **kube-apiserver** | Kubernetes API endpoint | Control plane only |
| **kube-scheduler** | Pod scheduling decisions | Control plane only |
| **kube-controller-manager** | Cluster state reconciliation | Control plane only |
| **coredns** | DNS resolution for services and pods | All nodes (DaemonSet) |

### Storage
- **Longhorn**: Distributed block storage system
  - Runs as DaemonSet on all nodes
  - Provides persistent volumes via dynamic provisioning
  - Replication factor: 2x by default (data durability)
  - Storage classes: `longhorn` (default), `longhorn-static`

### Ingress & Load Balancing
- **Traefik**: Cloud-native ingress controller
  - Handles TLS termination (if configured)
  - Routes external traffic to internal services based on host/path rules
  - Exposed via MetalLB as LoadBalancer service
- **MetalLB**: Layer 2 load balancer for bare metal clusters
  - Assigns public IP (`10.10.x.100`) to Traefik service
  - Uses ARP to advertise the IP on the local network

### Monitoring Stack
- **Prometheus**: Metrics collection and storage
- **Alertmanager**: Alert routing and notification
- **Node Exporter**: System-level metrics (CPU, memory, disk, network)
- **kube-state-metrics**: Kubernetes object state metrics

## Resource Allocation

### Current Workload Distribution

| Node | Pods Running | Longhorn Agents | Monitoring | Notes |
|------|--------------|-----------------|------------|-------|
| Control Plane | ~20 pods (system + some apps) | 1 agent | Node Exporter | Runs system-critical workloads |
| Worker #1 | ~15 pods (applications) | 1 agent | Node Exporter | Primary application node |
| Worker #2 | ~15 pods (applications) | 1 agent | Node Exporter | Secondary application node |

### Resource Limits (Approximate)
- **Control Plane**: 8 GB RAM reserved for system components
- **Worker Nodes**: 24 GB RAM available for workloads each
- **Storage**: Longhorn volumes allocated from MainThin pool (30TB total)

## Scaling Considerations

### Horizontal Scaling
- Add more worker nodes to distribute workload
- Longhorn automatically replicates data across new nodes
- Traefik pods scale via HorizontalPodAutoscaler (if configured)

### Vertical Scaling
- Increase VM RAM/disk for control plane if etcd or API server resource pressure
- Worker nodes can be resized independently based on application needs

### High Availability
- **Current State**: Single control plane (no HA for API server/etcd)
- **Recommendation**: Add 2 more control plane nodes for true HA (3-node etcd cluster)
- **Trade-off**: Increased resource usage vs. fault tolerance

## Security Considerations

- **Network Policies**: Not extensively configured (all pods can communicate freely)
- **RBAC**: Kubernetes RBAC enabled by default (role-based access control)
- **Pod Security**: No PodSecurityPolicy or PodSecurityStandards enforced
- **Secrets Management**: Kubernetes native secrets (not encrypted at rest)
- **Recommendation**: Implement network policies, enable encryption at rest for secrets

## Maintenance Procedures

### Cluster Upgrades
1. Upgrade worker nodes first (one at a time)
2. Drain node: `kubectl drain <node> --ignore-daemonsets`
3. Upgrade k3s binary on node
4. Uncordon: `kubectl uncordon <node>`
5. Repeat for control plane last (requires cluster downtime)

### Node Replacement
1. Create new VM with same specifications
2. Join node to cluster via `k3s join` command with token from control plane
3. Wait for Longhorn replication and scheduling
4. Drain old node: `kubectl drain <old-node>`
5. Delete old node: `kubectl delete node <old-node>`

### Backup etcd (Recommended Weekly)
```bash
# On control plane node
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/rancher/k3s/server/tls/etcd-server-ca.crt \
  --cert=/etc/rancher/k3s/server/tls/etcd-server.crt \
  --key=/etc/rancher/k3s/server/tls/etcd-server.key
```

---

*Cluster architecture documentation. Specific internal IPs and network ranges have been generalized for public viewing.*
