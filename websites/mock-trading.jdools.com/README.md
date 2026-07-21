# Mock Trading Web App (mock-trading.jdools.com)

## Overview

This directory documents the web build of the Mock Trading mobile application. The web version is built using Expo and served via NGINX from a Kubernetes Deployment. The source code for the full React Native app (including mobile-specific components) is maintained in `D:\Projects\mock-trading-mobile`.

## Web Build Process

### Build Pipeline
1. **Source Code**: React Native app with Expo SDK 56
2. **Build Command**: `npx expo export --platform web --output-dir web-build`
3. **Output**: Static HTML/CSS/JS in `web-build/` directory
4. **Docker Image**: Multi-stage build using Node.js 20 (builder) + NGINX (runtime)

### Dockerfile.build Structure
```dockerfile
# Stage 1: Build web assets with Expo
FROM node:20-alpine AS builder
ARG EXPO_PUBLIC_SUPABASE_URL
ARG EXPO_PUBLIC_SUPABASE_ANON_KEY
ENV EXPO_PUBLIC_SUPABASE_URL=${EXPO_PUBLIC_SUPABASE_URL}
ENV EXPO_PUBLIC_SUPABASE_ANON_KEY=${EXPO_PUBLIC_SUPABASE_ANON_KEY}
RUN npm ci --silent || npm install --silent
COPY . .
RUN npx expo export --platform web --output-dir web-build

# Stage 2: Serve with NGINX
FROM nginx:stable-alpine
RUN rm -rf /usr/share/nginx/html/*
COPY k8s/nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/web-build/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### NGINX Configuration (`k8s/nginx/default.conf`)
```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Serve static assets; fallback to index.html for SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Health check endpoint for k8s probes
    location = /health {
        return 200 'OK';
        add_header Content-Type text/plain;
    }
}
```

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

## Build & Deploy Script (`build-and-deploy.sh`)

Located in `D:\Projects\mock-trading-mobile\build-and-deploy.sh`, this script automates:
1. Loading environment variables from `.env` or ConfigMap
2. Building Docker image with Supabase credentials as build args
3. Pushing to Docker Hub (`jdools05/mock-trading:<git-sha>`)
4. Applying Kubernetes manifests (with image tag substituted)
5. Rolling out the new deployment
6. Waiting for rollout completion

### Usage
```bash
# Set required environment variables
export DOCKERHUB_USERNAME=jdools05
export EXPO_PUBLIC_SUPABASE_URL=https://<project>.supabase.co
export EXPO_PUBLIC_SUPABASE_ANON_KEY=<key>

# Run build and deploy
chmod +x build-and-deploy.sh
./build-and-deploy.sh
```

## Monitoring & Health Checks

### Readiness Probe
- **Endpoint**: `GET /health`
- **Response**: 200 OK with body "OK"
- **Purpose**: Ensures NGINX is serving before routing traffic

### Liveness Probe
- **Endpoint**: `GET /health`
- **Response**: 200 OK with body "OK"
- **Interval**: Every 20 seconds

## Troubleshooting

### Build Failures
- Check Supabase credentials are valid and not expired
- Verify network connectivity to Expo CDN during build
- Review Docker build logs for dependency installation errors

### Runtime Issues
- Check NGINX logs: `kubectl logs -f <pod-name>`
- Verify Supabase URL/KEY are correctly embedded in image
- Test `/health` endpoint directly: `curl http://<service-ip>/health`

---

*Web build documentation. Specific Supabase credentials and internal deployment details have been generalized for public viewing.*
