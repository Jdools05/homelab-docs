# Other Services

## Service Inventory

| Service | Domain/Path | Technology | Purpose | Status |
|---------|-------------|------------|---------|--------|
| **Dynamic 404 Handler** | `*.jdools.com` (catch-all) | Python 3.12 Alpine | Custom error page for undefined routes | Active |
| **Longhorn UI** | Internal only (port 9500) | Longhorn web interface | Storage management dashboard | Active |
| **Prometheus** | Internal only (port 9090) | Prometheus server | Metrics collection and storage | Active |
| **Alertmanager** | Internal only (port 9093) | Alert routing service | Notification delivery | Active |

## Dynamic 404 Handler

A catch-all Ingress rule (`*.jdools.com`) routes to a Python 3.12 Alpine container that returns a custom 404 page with links back to the main services. Prevents exposure of server information for undefined routes.

```yaml
# Ingress (catch-all)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catchall-404
spec:
  ingressClassName: traefik
  rules:
    - host: "*.jdools.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: dynamic-404
                port:
                  number: 80
```

## Longhorn UI

Web interface for managing Longhorn volumes, snapshots, and backups. Exposed as a ClusterIP service — not accessible externally by default. Features include volume health monitoring, snapshot creation/deletion, and backup target configuration (S3).

## Prometheus Stack

Comprehensive monitoring solution with these components:
- **Prometheus Server** (port 9090) — metrics collection and query interface (8 GB PVC on Longhorn)
- **Alertmanager** (port 9093) — alert routing and notification delivery
- **Node Exporter** (DaemonSet, port 9100) — system-level metrics on all nodes
- **kube-state-metrics** — Kubernetes object state as Prometheus metrics

All services are internal-only (ClusterIP). Prometheus provides a built-in PromQL query interface for ad-hoc data exploration.

## Nextcloud VM (105)

Personal cloud storage and file sharing service running on Proxmox VM 105. Uses Ubuntu/Debian with Nextcloud latest stable, PostgreSQL/MariaDB internally, and ~1 TB of storage. Internal network only — not exposed externally by default.

## Archived VMs (Stopped)

| VM | Purpose | Status |
|----|---------|--------|
| 100 (mceternal) | Minecraft server (MC Eternal modpack, 32 GB RAM) | Stopped — archived |
| 104 (dev) | Development environment (16 GB RAM, 512 GB disk) | Stopped — archived |
| 125 (docker) | Docker host for non-K8s workloads (8 GB RAM, 64 GB disk) | Stopped — archived |

These VMs are no longer actively maintained. Their storage allocations remain on the MainThin pool but can be reclaimed if needed.

---

*Other services documentation. Specific internal configurations and access details have been generalized.*
