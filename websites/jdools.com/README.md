# JDools Landing Page (jdools.com)

## Overview

This directory contains assets and documentation for the JDools landing page hosted at `https://jdools.com`. The landing page is a static portfolio site with a glassmorphism UI design featuring an animated aurora background.

## Current Assets

| File | Description |
|------|-------------|
| `favicon.ico` | Site favicon (binary) |
| `robots.txt` | Search engine directives |
| `sitemap.xml` | SEO sitemap listing all services |
| `DEPLOY.md` | Deployment documentation for K8s ConfigMap updates |

## HTML Source

The main `index.html` is **not stored in this repository** because it's served from a Kubernetes ConfigMap (`landing-page-html`). The HTML contains the full glassmorphism UI with:

- Animated aurora background (CSS gradients + keyframe animations)
- Frosted glass card effect (backdrop-filter blur)
- Service links to Mock Trading, Power Playlist, and AOA Marching Cubes
- Responsive design for desktop and mobile

## Design Features

### Visual Style
- **Glassmorphism**: Frosted glass card with `backdrop-filter: blur(20px)`
- **Aurora Background**: Animated radial gradients in purple/blue hues
- **Service Cards**: Hoverable links with transform animation on hover
- **Color Scheme**: Dark theme (`#0f172a` background) with indigo accents (`#6366f1`)

### Services Linked
1. **Power Playlist** - Spotify playlist management (`power-playlist.jdools.com`)
2. **Mock Trading** - Paper trading app with real-time quotes (`mock-trading.jdools.com`)
3. **AOA Marching Cubes** - 3D surface visualization tool (`aoa-marching-cubes.jdools.com`)

## SEO Configuration

### robots.txt
```txt
User-agent: *
Allow: /

Sitemap: https://jdools.com/sitemap.xml

Disallow: /api/
```

### sitemap.xml
Lists all service URLs with last modification dates and priority rankings. Updated when new services are added or existing ones are modified.

## Deployment Architecture

```
┌─────────────────────────────────────────────────────┐
│  Kubernetes ConfigMap (landing-page-html)           │
│  ├── index.html (main page)                        │
│  ├── sitemap.xml                                   │
│  └── robots.txt                                    │
└──────────────────────┬──────────────────────────────┘
                       │ mounted as volume
                       ▼
┌─────────────────────────────────────────────────────┐
│  NGINX Pod (nginx:alpine)                           │
│  - Serves static files from ConfigMap               │
│  - Port 80/TCP                                      │
└──────────────────────┬──────────────────────────────┘
                       │ ClusterIP service
                       ▼
┌─────────────────────────────────────────────────────┐
│  Traefik Ingress → jdools.com                       │
└─────────────────────────────────────────────────────┘
```

---

*Website assets directory. The main HTML source is stored in Kubernetes ConfigMap and not committed to this repository.*
