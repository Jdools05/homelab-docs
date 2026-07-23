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
