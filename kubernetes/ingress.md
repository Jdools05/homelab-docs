# Ingress Configuration (Traefik)

## Overview

Traefik v3.6 is the ingress controller, installed via Helm in `kube-system`. It routes external traffic from MetalLB's LoadBalancer IP to internal Kubernetes services based on host and path rules. All current traffic flows over HTTP — TLS termination has not been enabled yet.

## Traffic Flow

```
Internet → OPNsense (NAT) → MetalLB IP (10.10.x.100)
    → Traefik Ingress Controller (DaemonSet on all nodes)
        ├── jdools.com           → landing-page service
        ├── mock-trading.jdools.com  → mock-trading service
        ├── power-playlist.jdools.com
        │   ├── /api/*           → power-playlist-api:3000
        │   └── /*               → power-playlist-web:80
        ├── aoa-marching-cubes.jdools.com → aoa-marching-cubes service
        └── *.jdools.com (catch-all)     → dynamic-404 handler
```

## Routing Rules

| Host | Path | Backend Service | Port |
|------|------|-----------------|------|
| `jdools.com` | `/` | landing-page | 80 |
| `mock-trading.jdools.com` | `/` | mock-trading | 80 |
| `power-playlist.jdools.com` | `/api` | power-playlist-api | 3000 |
| `power-playlist.jdools.com` | `/` | power-playlist-web | 80 |
| `aoa-marching-cubes.jdools.com` | `/` | aoa-marching-cubes | 80 |
| `*.jdools.com` (wildcard) | `/` | dynamic-404 | 80 (catch-all) |

## TLS Status

Not configured. All services are exposed over plain HTTP via the Traefik `web` entrypoint (port 80).

## Service Discovery

Traefik discovers services automatically via Kubernetes CRDs (Ingress, IngressClass). Services are exposed internally as ClusterIP on their respective ports. Traefik uses readiness probe results to determine healthy endpoints.

---

*Ingress configuration documentation. Specific internal IPs have been generalized.*
