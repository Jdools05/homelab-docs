# Ingress Configuration (Traefik)

## Overview

Traefik v3.6 is the ingress controller, installed via Helm in `kube-system`. It routes external traffic from MetalLB's LoadBalancer IP to internal Kubernetes services based on host and path rules. All current traffic flows through Cloudeflare Tunnel to use TLS termination.

## Traffic Flow

```
Internet → Router → Switch → OPNsense (NAT) → MetalLB IP (10.10.x.100)
    → Traefik Ingress Controller (DaemonSet on all nodes)
        ├── jdools.com           → landing-page service
        ├── www.jdools.com       → 301 → jdools.com (www-to-apex middleware)
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
| `www.jdools.com` | `/` | www-to-apex redirect middleware | 80 |
| `mock-trading.jdools.com` | `/` | mock-trading | 80 |
| `power-playlist.jdools.com` | `/api` | power-playlist-api | 3000 |
| `power-playlist.jdools.com` | `/` | power-playlist-web | 80 |
| `aoa-marching-cubes.jdools.com` | `/` | aoa-marching-cubes | 80 |
| `*.jdools.com` (wildcard) | `/` | dynamic-404 | 80 (catch-all) |

### www.jdools.com → jdools.com redirect

`www.jdools.com` is intentionally **not** served as its own content. A
`www-jdools-com-ingress` routes the host at priority 100 to a `www-to-apex`
`redirectRegex` middleware that issues a permanent (308) redirect to the apex
domain, preserving the path:

```
https://www.jdools.com/anything  →  308  https://jdools.com/anything
```

The apex `www-to-apex` middleware lives in the `default` namespace, so the
router references it as `default-www-to-apex@kubernetescrd` — the namespace
prefix is required (matching how `default-error-pages` is referenced). A bare
`www-to-apex@kubernetescrd` fails to resolve, which silently **disables the
router**, so `www` traffic falls through to the catch-all 404.

The regex is scheme-agnostic (`https?://www\.jdools\.com/?(.*)$` →
`https://jdools.com/$1`) because the Cloudflare Tunnel terminates TLS at the
edge and forwards traffic to the Connector over HTTP — Traefik only ever sees an
`http://` request, so a regex hard-coded to `https://` would never match.

The `*.jdools.com` catch-all uses a `HostRegexp` rule, so it also matches
`www.jdools.com`; the dedicated `www` router wins only because its priority
(100) exceeds the catch-all's default (0).


## Service Discovery

Traefik discovers services automatically via Kubernetes CRDs (Ingress, IngressClass). Services are exposed internally as ClusterIP on their respective ports. Traefik uses readiness probe results to determine healthy endpoints.
