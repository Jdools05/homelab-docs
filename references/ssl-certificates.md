# SSL/TLS Certificate Management

## Current Status

**HTTPS is not currently enabled.** All traffic flows over plain HTTP:
- Traefik ingress controller configured for `web` (port 80) and `websecure` (port 443) entrypoints
- No TLS certificates deployed yet
- Services exposed via MetalLB IP without encryption

## Planned Approach

Let's Encrypt via cert-manager is the intended approach for automated certificate issuance and renewal, integrated with Traefik's ACME HTTP-01 challenge. This requires port 80 to be accessible from the internet (which it currently is via OPNsense NAT).

---

*SSL/TLS certificate documentation. Specific email addresses, domain names, and internal configurations have been generalized.*
