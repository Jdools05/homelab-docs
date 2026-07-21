# Other Services

## Overview

In addition to the primary services (Landing Page, Mock Trading, Power Playlist), the homelab runs several supporting services that provide infrastructure utilities and fallback handling.

## Service Inventory

| Service | Domain/Path | Technology | Purpose | Status |
|---------|-------------|------------|---------|--------|
| **Dynamic 404 Handler** | `*.jdools.com` (catch-all) | Python 3.12 Alpine | Custom error page for undefined routes | Active |
| **Longhorn UI** | Internal only (port 9500) | Longhorn web interface | Storage management dashboard | Active |
| **Prometheus** | Internal only (port 9090) | Prometheus server | Metrics collection and storage | Active |
| **Alertmanager** | Internal only (port 9093) | Alert routing service | Notification delivery | Active |

## Dynamic 404 Handler

### Purpose
Provides a custom 404 error page for any requests to undefined routes on `*.jdools.com` domains. This ensures consistent user experience and prevents exposure of server information.

### Technology Stack
- **Runtime**: Python 3.12 Alpine (lightweight container)
- **Web Server**: Custom Python HTTP server or Flask/FastAPI (minimal framework)
- **Response**: HTML page with link back to main services

### Deployment
```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dynamic-404
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dynamic-404
  template:
    metadata:
      labels:
        app: dynamic-404
    spec:
      containers:
        - name: python-404
          image: python:3.12-alpine
          ports:
            - containerPort: 8080
```

### Ingress Rule (Catch-all)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catchall-404
spec:
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

### Response Format
Returns HTML page with:
- Custom 404 error message
- Link to `jdools.com` (main landing page)
- List of available services with direct links

## Longhorn UI

### Purpose
Web interface for managing Longhorn distributed storage volumes, snapshots, and backups.

### Access
- **Service**: `longhorn-frontend` (ClusterIP)
- **Port**: 80/TCP
- **URL**: Internal only (not exposed externally by default)
- **Authentication**: None (internal network security)

### Features
- View all volumes and their replication status
- Create/delete snapshots for backup
- Manage backup targets (S3 configuration)
- Monitor volume health and performance metrics
- Access engine logs for troubleshooting

### Security Considerations
- Not exposed to internet (ClusterIP only)
- No authentication required (relies on network isolation)
- Recommendation: Add authentication if exposing externally

## Prometheus Stack

### Purpose
Comprehensive monitoring solution for cluster observability.

### Components

#### Prometheus Server
- **Port**: 9090/TCP (internal ClusterIP)
- **Purpose**: Metrics collection, storage, and query interface
- **Data Retention**: Configured via storage class (8 GB PVC)
- **Scrape Targets**: All Kubernetes nodes, pods, and custom exporters

#### Alertmanager
- **Port**: 9093/TCP (internal ClusterIP)
- **Purpose**: Alert routing and notification delivery
- **Notifications**: Email, webhook, or other integrations

#### Node Exporter
- **Type**: DaemonSet (runs on all nodes)
- **Port**: 9100/TCP (per-node metrics endpoint)
- **Metrics Collected**: CPU, memory, disk I/O, network traffic

#### kube-state-metrics
- **Purpose**: Exposes Kubernetes object state as Prometheus metrics
- **Metrics**: Pod status, deployment replicas, resource usage, etc.

### Query Interface
Prometheus provides a built-in query interface (PromQL) for ad-hoc data exploration:
```promql
# Example queries
container_memory_usage_bytes{namespace="default"}
node_cpu_seconds_total{mode="idle"}
rate(http_requests_total[5m])
```

## Nextcloud VM (105)

### Purpose
Personal cloud storage and file sharing service.

### Technology Stack
- **OS**: Ubuntu/Debian (running on Proxmox VM)
- **Application**: Nextcloud latest stable
- **Database**: PostgreSQL or MariaDB (internal to VM)
- **Storage**: 1 TB primary volume + 828 MB system volume

### Access
- **URL**: Internal network only (not exposed externally by default)
- **Authentication**: Username/password with optional 2FA
- **Features**: File sync, calendar, contacts, office suite integration

### Security Considerations
- Not exposed to internet (internal VM only)
- Recommend: Add reverse proxy with HTTPS if exposing externally
- Recommend: Enable Two-Factor Authentication for all users

## MC Eternal Modpack Server (100) - Stopped

### Purpose
Minecraft server running the "MC Eternal" modpack (vanilla+ mod collection).

### Status
Currently **stopped** (VM 100 power state: off)

### Technology Stack
- **OS**: Ubuntu/Debian (running on Proxmox VM)
- **Application**: Minecraft Java Edition with MC Eternal modpack
- **RAM Allocation**: 32 GB (dedicated to JVM heap)
- **Disk**: 128 GB (world data + mods)

### Use Case
- Personal gaming server
- Multiplayer hosting for friends/family
- Currently archived (not actively maintained)

## Docker Host VM (125) - Stopped

### Purpose
Dedicated Docker host for non-Kubernetes workloads.

### Status
Currently **stopped** (VM 125 power state: off)

### Technology Stack
- **OS**: Ubuntu/Debian
- **Application**: Docker Engine + Docker Compose
- **RAM Allocation**: 8 GB
- **Disk**: 64 GB

### Use Case
- Running containers that don't need Kubernetes orchestration
- Testing new container images before deploying to K8s
- Services with complex networking requirements

---

*Other services documentation. Specific internal configurations and access details have been generalized for public viewing.*
