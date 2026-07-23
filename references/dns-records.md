# DNS Records Configuration

## Overview

DNS is managed at two levels: external records at the domain registrar (pointing to public IPs or CDN URLs), and internal resolution via Kubernetes CoreDNS for in-cluster service discovery.

## External DNS (Public)

### Primary Domain: `jdools.com`

| Type | Host | Value | TTL | Purpose |
|------|------|-------|-----|---------|
| **A** | `@` | `<public-ip>` | 300 | Main domain → landing page |
| **CNAME** | `*` | `@` | 300 | Wildcard subdomain redirect (optional) |

### Service Subdomains

| Type | Host | Value | TTL | Purpose |
|------|------|-------|-----|---------|
| **A/CNAME** | `mock-trading` | `<public-ip>` or CDN URL | 300 | Mock Trading web app |
| **CNAME** | `power-playlist` | `<external-hosting-url>` | 300 | Spotify integration service |
| **A/CNAME** | `aoa-marching-cubes` | `<public-ip>` | 300 | AOA Marching Cubes visualization |

## Internal DNS (Kubernetes CoreDNS)

Services are accessible within the cluster using:
```
<service-name>.<namespace>.svc.cluster.local
```

**Examples:**
- `landing-page.default.svc.cluster.local`
- `mock-trading.default.svc.cluster.local`
- `power-playlist-api.default.svc.cluster.local`

Pods in the same namespace can use short names:
```bash
curl http://landing-page:80/health
```

---

*DNS records documentation. Specific public IPs and provider details have been generalized.*
