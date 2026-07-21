# Kubernetes Namespaces

## Overview

Kubernetes namespaces provide a way to divide cluster resources between multiple users or projects. The homelab uses several namespaces to organize services by function and environment.

## Namespace Inventory

### Active Namespaces

| Namespace | Purpose | Resource Count | Status |
|-----------|---------|----------------|--------|
| `default` | Primary application workloads | ~30+ resources | Active |
| `kube-system` | Kubernetes system components | ~15 resources | Active |
| `kube-public` | Publicly accessible resources | Minimal | Active |
| `kube-node-lease` | Node lease objects (heartbeat) | Minimal | Active |
| `longhorn-system` | Longhorn distributed storage | ~20 resources | Active |
| `metallb-system` | MetalLB load balancer | ~5 resources | Active |
| `monitoring` | Prometheus monitoring stack | ~10 resources | Active |

## Namespace Details

### default
**Purpose**: Primary namespace for application workloads (homelab single-tenant setup)

**Resources Hosted:**
- Landing page (NGINX static site)
- Mock Trading app (React Native web build)
- Power Playlist API + Web (Node.js + Next.js)
- AOA Marching Cubes (Java 3D visualization)
- Dynamic 404 handler (Python/Alpine)

**Rationale**: Single-tenant homelab doesn't require strict namespace isolation. All user services share the `default` namespace for simplicity.

### kube-system
**Purpose**: Kubernetes system-critical components

**Resources Hosted:**
- CoreDNS (cluster DNS)
- k3s local-path-provisioner
- Metrics server
- Traefik ingress controller
- Traefik LoadBalancer pods (via MetalLB)

**Access Control**: Restricted access - only cluster administrators should modify system resources.

### longhorn-system
**Purpose**: Longhorn distributed block storage system

**Resources Hosted:**
- Longhorn manager ( DaemonSet on all nodes)
- CSI plugins (container storage interface)
- Engine images (storage engine)
- Longhorn UI (web interface for management)
- Admission webhook (validation)

**Access Control**: Cluster administrators can manage storage classes and volumes.

### metallb-system
**Purpose**: Layer 2 load balancer for bare metal clusters

**Resources Hosted:**
- MetalLB controller (pods on control plane)
- MetalLB speaker (DaemonSet on all nodes)
- Webhook service (TLS termination for CRD validation)

**Access Control**: Cluster administrators configure IP address pools and L2 advertisement settings.

### monitoring
**Purpose**: Prometheus monitoring stack for cluster observability

**Resources Hosted:**
- Prometheus server (metrics collection + storage)
- Alertmanager (alert routing + notification)
- Node Exporter (DaemonSet for system metrics)
- kube-state-metrics (Kubernetes object metrics)
- Pushgateway (for batch job metrics)

**Access Control**: Monitoring data is internal - not exposed externally by default.

## Namespace Isolation Considerations

### Current State
- **No Network Policies**: All pods can communicate across namespaces
- **No Resource Quotas**: No limits on CPU/memory per namespace
- **No Limit Ranges**: No default resource constraints for new pods

### Recommended Improvements
1. **Network Policies**: Restrict inter-namespace communication
2. **Resource Quotas**: Prevent noisy neighbor problems
3. **Limit Ranges**: Set default CPU/memory requests/limits
4. **RBAC**: Restrict namespace access to specific users/groups

## Namespace Creation Guidelines

### When to Create a New Namespace
- Multi-tenant environment (different teams/projects)
- Separate environments (dev/staging/prod)
- Isolated workloads requiring different resource limits
- Third-party applications that should be isolated from core services

### Current Recommendation
For this single-user homelab, the current namespace layout is sufficient. Consider adding namespaces only if:
- Service count grows significantly (>50 services)
- Different security requirements emerge
- Resource isolation becomes necessary

## Namespace Management Commands

```bash
# List all namespaces
kubectl get namespaces

# Describe a specific namespace
kubectl describe namespace <name>

# Create a new namespace
kubectl create namespace <new-namespace>

# Delete a namespace (and all resources in it)
kubectl delete namespace <name>

# Check resource usage per namespace
kubectl top namespaces
```

## Security Context

### Pod Security Standards
No pod security standards are currently enforced. All pods run with default security context (no restrictions on privileges, capabilities, or seccomp profiles).

**Recommendation**: Implement `restricted` pod security standard for production workloads to minimize attack surface.

### Service Account Permissions
- **default service account**: Used by all pods unless explicitly overridden
- **Cluster-admin access**: Not granted by default (requires explicit ClusterRoleBinding)

---

*Namespace documentation. Specific resource counts and internal configurations have been generalized for public viewing.*
