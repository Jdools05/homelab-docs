# JDools Landing Page (jdools.com)

## Overview

The JDools landing page is a static portfolio website served from a Kubernetes ConfigMap. It provides a glassmorphism-style interface with an animated aurora background, showcasing links to other homelab services including Mock Trading, Power Playlist, and AOA Marching Cubes.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Landing Page Service                     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Kubernetes ConfigMap (landing-page-html)           │   │
│  │  - index.html (main page with aurora UI)            │   │
│  │  - sitemap.xml (SEO sitemap)                        │   │
│  │  - robots.txt (search engine directives)            │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  NGINX Pod (nginx:alpine)                           │   │
│  │  - Serves static files from ConfigMap               │   │
│  │  - Handles HTTP requests                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Traefik Ingress → jdools.com                       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Hosting** | Kubernetes (k3s) | Container orchestration |
| **Web Server** | NGINX Alpine (`nginx:alpine`) | Static file serving |
| **Configuration** | ConfigMap | HTML, sitemap, robots.txt storage |
| **Ingress** | Traefik v3.6 | External routing and TLS termination (planned) |
| **UI Framework** | Vanilla HTML/CSS/JS | Glassmorphism design with aurora animation |

## Design Features

### Visual Design
- **Glassmorphism UI**: Frosted glass card effect with backdrop blur
- **Aurora Background**: Animated gradient background with purple/blue hues
- **Responsive Layout**: Works on desktop and mobile devices
- **Service Cards**: Hoverable cards linking to other homelab services

### Services Listed
1. **Power Playlist** - Spotify playlist management (`power-playlist.jdools.com`)
2. **Mock Trading** - Paper trading app with real-time quotes (`mock-trading.jdools.com`)
3. **AOA Marching Cubes** - 3D surface visualization tool (`aoa-marching-cubes.jdools.com`)

## Deployment Process

### ConfigMap Structure
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: landing-page-html
  namespace: default
data:
  index.html: |
    <!-- Full HTML with aurora UI -->
  sitemap.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
      <!-- Sitemap entries for all services -->
    </urlset>
  robots.txt: |
    User-agent: *
    Allow: /
    Disallow: /api/
```

### NGINX Configuration
The NGINX pod mounts the ConfigMap as a volume and serves files directly from it. Standard NGINX configuration handles MIME types and static file serving.

### Update Process
1. Edit `index.html` in local development environment
2. Apply updated ConfigMap: `kubectl apply -f configmap.yaml`
3. NGINX pod automatically picks up new files (ConfigMap volume mount)
4. No redeployment required for content changes

## SEO Configuration

### Sitemap (sitemap.xml)
Lists all service URLs with:
- Last modification date
- Change frequency
- Priority ranking

### Robots.txt
Directives for search engine crawlers:
- Allow all paths except `/api/`
- Point to sitemap location

## Maintenance

### Adding New Services
1. Update `index.html` in ConfigMap with new service card
2. Add URL to `sitemap.xml`
3. Apply updated ConfigMap to cluster

### Updating Design
1. Modify CSS styles in `index.html`
2. Test locally or in development environment
3. Apply updated ConfigMap

## Monitoring & Logging

### Access Logs
NGINX access logs available via:
```bash
kubectl logs -f <landing-page-pod-name>
```

### Health Checks
- **Readiness Probe**: HTTP GET `/` returns 200 OK
- **Liveness Probe**: HTTP GET `/` returns 200 OK
- **Interval**: Every 10 seconds

## Security Considerations

- **No Authentication**: Public-facing static site, no login required
- **CORS**: Not configured (static HTML/JS served directly)
- **HTTPS**: Planned but not yet implemented (would require TLS certificate in Traefik)
- **Content Security Policy**: Could be added via NGINX headers for XSS protection

## Future Enhancements

1. **TLS/HTTPS**: Enable HTTPS with Let's Encrypt certificates
2. **Analytics**: Add privacy-friendly analytics (Plausible, Fathom)
3. **Blog Section**: Add blog posts about homelab projects
4. **Contact Form**: Simple contact form with email notification
5. **Performance Optimization**: Minify HTML/CSS/JS for faster loading

---

*Service documentation. Specific internal configurations and deployment details have been generalized for public viewing.*
