# Ingress Configuration (Traefik)

## Overview

The homelab uses Traefik v3.6 as the ingress controller for Kubernetes. Traefik handles external traffic routing, TLS termination, and service discovery from Kubernetes resources. External access flows through OPNsense NAT → MetalLB LoadBalancer IP → Traefik → Internal Services.

## Architecture

```
Internet
    │
    ▼
[OPNsense Firewall] ← WAN (DHCP)
    │
    ├── Port Forward: 80/TCP, 443/TCP → MetalLB IP
    │
    ▼
[MetalLB LoadBalancer] ← IP: 10.10.x.100
    │
    ▼
[Traefik Ingress Controller] (DaemonSet on all nodes)
    │
    ├── Rule: host(jdools.com) → landing-page service
    ├── Rule: host(mock-trading.jdools.com) → mock-trading service
    ├── Rule: host(power-playlist.jdools.com, path=/api) → power-playlist-api
    ├── Rule: host(power-playlist.jdools.com, path=/) → power-playlist-web
    ├── Rule: host(aoa-marching-cubes.jdools.com) → aoa-marching-cubes service
    └── Default: 404 handler (dynamic-404)
```

## Traefik Deployment

### Installation Method
- **Helm Chart**: Installed via Helm from Rancher chart repository
- **Namespace**: `kube-system`
- **Service Type**: LoadBalancer (exposed via MetalLB)
- **Replicas**: DaemonSet (one pod per node for high availability)

### Configuration

#### Entrypoints
| Name | Port | Protocol | Description |
|------|------|----------|-------------|
| `web` | 80 | HTTP | Incoming HTTP traffic |
| `websecure` | 443 | HTTPS | Incoming HTTPS traffic (if TLS configured) |

#### Providers
- **Kubernetes CRD**: Watches Ingress, IngressClass, and other Traefik custom resources
- **Kubernetes Gateway** (optional): Can also use Kubernetes Gateway API

## Ingress Resources

### Service Routing Rules

| Host | Path | Backend Service | Port | Priority | Description |
|------|------|-----------------|------|----------|-------------|
| `jdools.com` | `/` | landing-page | 80 | 100 | Landing page (static HTML) |
| `mock-trading.jdools.com` | `/` | mock-trading | 80 | 100 | React Native web app |
| `power-playlist.jdools.com` | `/api` | power-playlist-api | 3000 | 200 | Node.js API backend |
| `power-playlist.jdools.com` | `/` | power-playlist-web | 80 | 100 | Next.js frontend |
| `aoa-marching-cubes.jdools.com` | `/` | aoa-marching-cubes | 80 | 100 | Java 3D visualization |
| `*.jdools.com` (wildcard) | `/` | dynamic-404 | 80 | - | Catch-all for undefined routes |

### Ingress Annotations

#### Standard Annotations
```yaml
# Traefik entrypoints
traefik.ingress.kubernetes.io/router.entrypoints: web,websecure

# Priority (higher number = higher priority)
traefik.ingress.kubernetes.io/router.priority: "100"

# Kubernetes ingress class
kubernetes.io/ingress.class: traefik
```

#### Middleware Annotations
```yaml
# Error pages middleware
traefik.ingress.kubernetes.io/router.middlewares: default-error-pages@kubernetescrd
```

## TLS Configuration

### Current Status
- **TLS Termination**: Not configured (HTTP only)
- **Certificate Provider**: N/A (no Let's Encrypt or custom certs deployed)
- **Redirect HTTP→HTTPS**: Not enabled

### Recommended Enhancement
To enable HTTPS:
1. Install cert-manager for automated certificate provisioning
2. Configure ClusterIssuer with Let's Encrypt ACME challenge
3. Add TLS section to Ingress resources:
   ```yaml
   tls:
     - hosts:
         - jdools.com
       secretName: jdools-tls
   ```

## Service Discovery

Traefik automatically discovers services via Kubernetes:
- **Service Type**: ClusterIP (internal only)
- **Endpoints**: Automatically updated as pods scale
- **Health Checks**: Traefik uses readiness probe results to determine healthy endpoints

### Example: Power Playlist API Discovery
```yaml
# Service definition
apiVersion: v1
kind: Service
metadata:
  name: power-playlist-api
spec:
  selector:
    app: power-playlist-api
  ports:
    - port: 3000
      targetPort: 3000

# Ingress routing to /api path
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: power-playlist-api-ingress
spec:
  rules:
    - host: power-playlist.jdools.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: power-playlist-api
                port:
                  number: 3000
```

## Load Balancing

### Algorithm
- **Default**: Round-robin across healthy endpoints
- **Consistent Hashing**: Available via Traefik middleware (if configured)

### Health Checks
- **Readiness Probe**: Kubernetes readiness probe determines endpoint health
- **Traefik Middleware**: Can add custom health check logic if needed

## Monitoring & Debugging

### Viewing Active Routes
```bash
# List all ingress resources
kubectl get ingress --all-namespaces

# Describe specific ingress for detailed routing info
kubectl describe ingress <ingress-name> -n <namespace>
```

### Traefik Dashboard (Development)
Traefik includes a built-in dashboard for debugging:
1. Enable in Traefik configuration:
   ```yaml
   api:
     dashboard: true
   ```
2. Access via `http://<traefik-ip>/dashboard/` (internal only, not exposed externally)

### Log Analysis
```bash
# View Traefik logs
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=100

# Filter for specific host
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik | grep "jdools.com"
```

## Security Considerations

- **No Authentication**: Currently, any request to valid hosts is served (no auth middleware)
- **Rate Limiting**: Not configured (vulnerable to DDoS if exposed publicly)
- **CORS**: Handled by individual services (not via Traefik middleware)
- **Recommendation**: Add rate limiting, authentication, and CORS headers via Traefik middlewares

## Maintenance Procedures

### Updating Traefik Version
1. Edit Helm release: `helm upgrade traefik rancher/traefik --namespace kube-system`
2. Or update chart repository and reinstall if major version change required

### Adding New Service
1. Create Kubernetes Service resource
2. Create Ingress resource with host/path rules
3. Verify routing via `kubectl describe ingress <name>`
4. Test external access through OPNsense port forward

---

*Ingress configuration documentation. Specific internal IPs and service addresses have been generalized for public viewing.*
