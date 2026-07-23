# JDools Landing Page (jdools.com)

## Overview

A static portfolio site with a glassmorphism UI and animated aurora background. It links to other homelab services: Mock Trading, Power Playlist, and AOA Marching Cubes. Hosted on Kubernetes as a ConfigMap volume mounted into an NGINX pod.

## Architecture

```
Kubernetes ConfigMap (landing-page-html)
  ├── index.html (aurora UI)
  ├── sitemap.xml
  └── robots.txt
        │ mounted as volume
        ▼
NGINX Alpine Pod (port 80/TCP)
        │ ClusterIP service
        ▼
Traefik Ingress → jdools.com
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Hosting** | Kubernetes (k3s) | Container orchestration |
| **Web Server** | NGINX Alpine (`nginx:alpine`) | Static file serving |
| **Configuration** | ConfigMap | HTML, sitemap, robots.txt storage |
| **Ingress** | Traefik v3.6 | External routing |

## Design Features

- Glassmorphism UI with frosted glass card effect and backdrop blur
- Animated aurora background (CSS gradients + keyframe animations)
- Responsive layout for desktop and mobile
- Service cards linking to Mock Trading, Power Playlist, and AOA Marching Cubes
