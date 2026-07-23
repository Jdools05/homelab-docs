# Websites Documentation

This directory contains documentation and assets for all websites hosted on the homelab infrastructure. Each website has its own subdirectory with relevant files and deployment information.

## Contents

- **[jdools.com/](./jdools.com/)** - Landing page (static HTML served from K8s ConfigMap)
  - `favicon.ico` - Site favicon
  - `robots.txt` - Search engine directives
  - `sitemap.xml` - SEO sitemap
  - `DEPLOY.md` - Deployment documentation
  - `README.md` - Design and architecture details

- **[mock-trading.jdools.com/](./mock-trading.jdools.com/)** - Mock Trading web app (Expo build served by NGINX)
  - `README.md` - Build process, Dockerfile structure, deployment pipeline documentation

## Architecture Overview

All websites are hosted on the k3s Kubernetes cluster:

```
Internet → OPNsense (NAT) → MetalLB IP → Traefik Ingress → Service → Pod
                                                         │
                                                         ├── jdools.com → landing-page pod (ConfigMap volume)
                                                         └── mock-trading.jdools.com → mock-trading pod (NGINX serving Expo build)
```

## Technology Stack Summary

| Website | Hosting Method | Web Server | Source Storage |
|---------|---------------|------------|----------------|
| jdools.com | Kubernetes ConfigMap | NGINX Alpine | ConfigMap `landing-page-html` |
| mock-trading.jdools.com | Docker image (multi-stage) | NGINX stable-alpine | Docker Hub (`jdools05/mock-trading`) |

## Ingress Routing

All websites are exposed via Traefik ingress controller with the following routing rules:

| Host | Path | Backend Service |
|------|------|-----------------|
| `jdools.com` | `/` | landing-page:80 |
| `mock-trading.jdools.com` | `/` | mock-trading:80 |
| `*.jdools.com` (wildcard) | `/` | dynamic-404:80 (catch-all) |
