# SSL/TLS Certificate Management

## Current Status

**HTTPS is not currently enabled** on the homelab services. All traffic flows over plain HTTP:
- Traefik ingress controller configured for `web` (port 80) and `websecure` (port 443) entrypoints
- No TLS certificates deployed yet
- Services exposed via MetalLB IP without encryption

## Planned Implementation

### Option 1: Let's Encrypt via cert-manager (Recommended)

**Advantages:**
- Automated certificate issuance and renewal
- Free certificates from trusted CA
- Seamless integration with Traefik

**Requirements:**
- Public domain names with valid DNS records
- Port 80 accessible from internet (for ACME HTTP-01 challenge)
- cert-manager installed in Kubernetes cluster

**Implementation Steps:**

1. **Install cert-manager**
   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
   ```

2. **Create ClusterIssuer for Let's Encrypt**
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       email: admin@example.com  # Replace with actual email
       privateKeySecretRef:
         name: letsencrypt-priv-key
       solvers:
         - http01:
             ingress:
               class: traefik
   ```

3. **Update Ingress Resources to Use TLS**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: jdools-com-ingress
   spec:
     tls:
       - hosts:
           - jdools.com
         secretName: jdools-tls  # cert-manager will create this
     rules:
       - host: jdools.com
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: landing-page
                   port:
                     number: 80
   ```

4. **Verify Certificate Issuance**
   ```bash
   kubectl get certificates
   kubectl describe certificate jdools-tls
   ```

### Option 2: Self-Signed Certificates (Development Only)

**Use Case:** Internal testing, non-production environments

**Drawbacks:**
- Not trusted by browsers (security warnings)
- Manual renewal required
- Not suitable for public-facing services

**Implementation:**
```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=*.jdools.com" \
  -addext "subjectAltName=DNS:*.jdools.com,DNS:jdools.com"

# Create Kubernetes Secret
kubectl create secret tls jdools-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n default
```

## Certificate Storage

### Kubernetes Secrets
Certificates are stored as Kubernetes TLS secrets:
```bash
# View certificate details (without private key)
kubectl get secret <secret-name> -o yaml | grep "tls.crt"

# Check expiry date
openssl x509 -in tls.crt -noout -dates
```

### File Locations (if using file-based storage)
- **Certificates**: `/etc/ssl/certs/<domain>.crt`
- **Private Keys**: `/etc/ssl/private/<domain>.key`
- **CA Bundle**: `/etc/ssl/certs/ca-certificates.crt`

## Certificate Renewal

### Automated (cert-manager)
- cert-manager automatically renews certificates 30 days before expiry
- No manual intervention required
- Monitor renewal status via: `kubectl get certificates -o wide`

### Manual
1. Generate new certificate (see self-signed or Let's Encrypt process)
2. Update Kubernetes secret: `kubectl create secret tls <name> --cert=new.crt --key=new.key --dry-run=client -o yaml | kubectl apply -f -`
3. Restart pods to load new certificates (if using file mounts)

## Security Best Practices

### Certificate Selection
- **Public Services**: Use Let's Encrypt or commercial CA certificates
- **Internal Services**: Self-signed or internal PKI (e.g., Vault, CFSSL)
- **Wildcard Certificates**: Useful for multiple subdomains but increase risk if compromised

### Key Management
- **Private Keys**: Never commit to version control
- **Storage**: Use Kubernetes secrets or external secret managers (HashiCorp Vault, AWS Secrets Manager)
- **Rotation**: Rotate keys annually or after security incidents

### TLS Configuration
- **Minimum Version**: TLS 1.2 (disable TLS 1.0 and 1.1)
- **Cipher Suites**: Use strong ciphers only (AES-GCM, ChaCha20)
- **HSTS**: Enable HTTP Strict Transport Security for additional protection

## Monitoring & Alerting

### Certificate Expiry Monitoring
```bash
# Check all certificates in cluster
kubectl get certificates -o wide

# Monitor specific certificate
kubectl describe certificate <name> | grep "Not After"
```

### Integration with Prometheus
- Export cert-manager metrics to Prometheus
- Set up alerts for certificates expiring within 30 days
- Example alert rule: `certmanager_certificate_expiring_within_days{days<=30} > 0`

## Troubleshooting

### Certificate Not Issued
1. Check ClusterIssuer status: `kubectl describe clusterissuer letsencrypt-prod`
2. Verify DNS records point to public IP
3. Ensure port 80 is accessible from internet (for ACME challenge)
4. Check cert-manager logs: `kubectl logs -n cert-manager deployment/cert-manager`

### Certificate Renewal Failed
1. Review cert-manager events: `kubectl get events -n cert-manager --sort-by='.lastTimestamp'`
2. Check ACME account status with Let's Encrypt
3. Verify DNS propagation for new domains

---

*SSL/TLS certificate documentation. Specific email addresses, domain names, and internal configurations have been generalized for public viewing.*
