# Services Documentation

This directory contains documentation for each service running on the homelab infrastructure. Each service document covers architecture, technology stack, deployment process, and operational details.

## Contents

- **[jdools-landing.md](./jdools-landing.md)** - Landing page (jdools.com) - Static HTML/NGINX
- **[mock-trading-mobile.md](./mock-trading-mobile.md)** - Mock Trading app - React Native + Supabase + Finnhub API
- **[power-playlist.md](./power-playlist.md)** - Power Playlist service - Spotify integration with Node.js API and Next.js frontend
- **[other-services.md](./other-services.md)** - Additional services (dynamic 404 handler, monitoring stack, etc.)

## Service Architecture Overview

All services run on the k3s Kubernetes cluster and are exposed via Traefik ingress controller:

```
Internet → OPNsense (NAT) → MetalLB IP → Traefik Ingress
    ├── jdools.com              → landing-page pod (NGINX, ConfigMap volume)
    ├── mock-trading.jdools.com → mock-trading pod (NGINX serving Expo build)
    ├── power-playlist.jdools.com
    │   ├── /api/*              → power-playlist-api:3000 (Node.js REST API)
    │   └── /*                  → power-playlist-web:80  (Next.js static export)
    ├── aoa-marching-cubes.jdools.com → aoa-marching-cubes pod (Java 3D viz)
    └── *.jdools.com (catch-all)  → dynamic-404 handler (Python/Alpine)
```
