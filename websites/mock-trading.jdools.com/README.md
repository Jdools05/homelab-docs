# Mock Trading Web App (mock-trading.jdools.com)

## Overview

This directory documents the web build of the Mock Trading mobile application. The web version is built using Expo and served via NGINX from a Kubernetes Deployment. The source code for the full React Native app (including mobile-specific components) is maintained in `D:\Projects\mock-trading-mobile`.

## Build & Serving Architecture

The web version uses a multi-stage Docker pipeline:
1. Node.js 20 Alpine builds the Expo web app (`npx expo export --platform web`)
2. NGINX serves the static assets from a second stage image
3. Image pushed to Docker Hub, then deployed to Kubernetes with Traefik ingress

### NGINX Configuration

- Serves static assets from the built `web-build/` directory
- SPA routing fallback: all paths resolve to `index.html` (`try_files $uri $uri/ /index.html`)
- Health check endpoint at `/health` returning 200 OK for k8s readiness probes

## Kubernetes Deployment

### Deployment Manifest (excerpt)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mock-trading
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mock-trading
  template:
    metadata:
      labels:
        app: mock-trading
    spec:
      containers:
        - name: web
          image: jdools05/mock-trading:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Service and Ingress
- **Service**: ClusterIP on port 80
- **Ingress Host**: `mock-trading.jdools.com`
- **Traefik Annotation**: `traefik.ingress.kubernetes.io/router.entrypoints: web,websecure`

## Environment Variables

The build process requires two environment variables passed via Docker build args:

| Variable | Purpose | Source |
|----------|---------|--------|
| `EXPO_PUBLIC_SUPABASE_URL` | Supabase project URL | Supabase dashboard |
| `EXPO_PUBLIC_SUPABASE_ANON_KEY` | Supabase anonymous API key | Supabase dashboard → Settings → API |

These are embedded at build time and cannot be changed without rebuilding the Docker image.

## Monitoring & Health Checks

### Readiness Probe
- **Endpoint**: `GET /health`
- **Response**: 200 OK with body "OK"
- **Purpose**: Ensures NGINX is serving before routing traffic

---

*Web build documentation. Specific Supabase credentials and internal deployment details have been generalized for public viewing.*
