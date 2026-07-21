# DNS Records Configuration

## Overview

This document outlines the DNS configuration for all domains and subdomains used by the homelab services. The DNS provider is managed externally (typically Cloudflare, Namecheap, or similar registrar), while internal resolution is handled by Kubernetes CoreDNS.

## External DNS (Public)

### Primary Domain: `jdools.com`

| Type | Host | Value | TTL | Purpose |
|------|------|-------|-----|---------|
| **A** | `@` | `<public-ip>` | 300 | Main domain → landing page |
| **CNAME** | `*` | `@` | 300 | Wildcard subdomain redirect (optional) |

### Service Subdomains

| Type | Host | Value | TTL | Purpose |
|------|------|-------|-----|---------|
| **A** or **CNAME** | `mock-trading` | `<public-ip>` or CDN URL | 300 | Mock Trading web app |
| **CNAME** | `power-playlist` | `<external-hosting-url>` | 300 | Power Playlist (hosted elsewhere) |
| **A** or **CNAME** | `aoa-marching-cubes` | `<public-ip>` | 300 | AOA Marching Cubes visualization |

### Notes on DNS Strategy

- **Wildcard vs. Explicit**: Wildcard records (`*.jdools.com`) can simplify management but may conflict with specific subdomain records. Use explicit records for production services.
- **CNAME Flattening**: Some providers (Cloudflare) support CNAME flattening, allowing `@` to point to a CNAME target.
- **TTL Values**: Low TTL (300s = 5 minutes) recommended for flexibility during migrations.

## Internal DNS (Kubernetes CoreDNS)

### Service Discovery
Kubernetes services are accessible within the cluster using the following naming convention:

```
<service-name>.<namespace>.svc.cluster.local
```

**Examples:**
- `landing-page.default.svc.cluster.local`
- `mock-trading.default.svc.cluster.local`
- `power-playlist-api.default.svc.cluster.local`

### Pod-to-Pod Communication
Pods can communicate using short service names if in the same namespace:
```bash
# From any pod in 'default' namespace:
curl http://landing-page:80/health
curl http://mock-trading:80/health
```

## DNS Management Tools

### External (Registrar)
- **Cloudflare Dashboard**: https://dash.cloudflare.com
- **Namecheap**: https://namecheap.com
- **GoDaddy**: https://godaddy.com
- **AWS Route 53**: https://aws.amazon.com/route53/

### Internal (Kubernetes)
```bash
# View CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS resolution from within cluster
kubectl run -i --tty dns-test --image=busybox:latest --rm --restart=Never -- nslookup <service-name>
```

## DNS Propagation & Verification

### Checking External Records
```bash
# Check A record
dig jdools.com +short

# Check CNAME for subdomain
dig mock-trading.jdools.com +short

# Verify TTL
dig jdools.com +trace
```

### Checking Internal Resolution
```bash
# From within k8s cluster
nslookup landing-page.default.svc.cluster.local

# Using kubectl exec
kubectl exec -it <pod-name> -- nslookup mock-trading
```

## DNS Security Considerations

- **DNSSEC**: Enable if supported by registrar for additional security
- **CAA Records**: Restrict which Certificate Authorities can issue certs for your domain
- **TXT Records**: Used for domain verification (Google Search Console, etc.) and SPF/DKIM for email

## Maintenance Procedures

### Adding a New Service Domain
1. Add DNS record at registrar (A or CNAME)
2. Create Kubernetes Ingress resource with new host
3. Verify external resolution: `dig <new-domain> +short`
4. Test internal routing via Traefik ingress

### Updating Records
1. Modify DNS record at registrar
2. Wait for TTL to expire (or manually flush cache)
3. Update Kubernetes resources if service IP changes
4. Verify end-to-end connectivity

---

*DNS records documentation. Specific public IPs and provider details have been generalized for public viewing.*
