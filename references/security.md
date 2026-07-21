# Security Overview

## Overview

This document outlines the security measures implemented in the homelab infrastructure, including network segmentation, access control, firewall rules, and best practices. The goal is to provide a secure environment for development and production workloads while maintaining operational simplicity.

## Network Security

### Firewall Configuration (OPNsense VM)

**Current State:**
- OPNsense VM acts as the primary firewall between WAN and internal networks
- NAT configured for all internal subnets
- Port forwarding enabled for specific services (if exposed externally)

**Recommended Enhancements:**
1. **Default Deny Policy**: Block all inbound traffic by default, allow only necessary ports
2. **Intrusion Detection**: Enable OPNsense IDS/IPS ruleset
3. **GeoIP Blocking**: Restrict access from high-risk countries if applicable
4. **Rate Limiting**: Prevent DDoS attacks on exposed services

### VLAN Segmentation

**Current VLANs:**
- VLAN 1: Management network (Proxmox host, SSH access)
- VLAN 99: Kubernetes cluster network
- VLAN 10-40: Service-specific networks (reserved for future use)

**Security Benefits:**
- Isolates management traffic from production workloads
- Limits lateral movement in case of compromise
- Enables granular firewall rules between VLANs

**Recommendation:**
- Implement strict inter-VLAN routing policies via OPNsense
- Restrict SSH access to VLAN 1 only (management network)
- Block unnecessary ports between service VLANs

## Access Control

### Proxmox VE Access

**Authentication:**
- Username/password authentication for web GUI and API
- Role-based access control (RBAC) with predefined roles:
  - `PVEAdmin`: Full administrative access
  - `PVEAuditor`: Read-only access for auditing
  - Custom roles can be created for specific permissions

**Authorization:**
- Users assigned to datacenter, node, or pool-level roles
- SSH key-based authentication recommended for CLI access
- Two-factor authentication (2FA) available but not enabled by default

**Recommendation:**
- Enable 2FA for all administrative accounts
- Review user permissions quarterly
- Audit access logs via Proxmox web GUI → Datacenter → Audit Log

### Kubernetes RBAC

**Current State:**
- Kubernetes native RBAC enabled (default in k3s)
- ClusterRole and ClusterRoleBinding resources manage cluster-wide permissions
- Namespace-scoped Roles and RoleBindings for per-namespace access

**Default Roles:**
- `cluster-admin`: Full administrative access (use with caution)
- `admin`: Administrative access within a namespace
- `edit`: Can create, update, and delete most objects in a namespace
- `view`: Read-only access to resources

**Recommendation:**
- Avoid using `cluster-admin` for daily operations
- Create custom roles for specific use cases (e.g., developer, CI/CD)
- Implement Pod Security Standards (`restricted`) for production workloads

### SSH Access

**Current Configuration:**
- Root SSH access enabled on Proxmox host and Kubernetes nodes
- Password authentication allowed (not recommended for production)

**Security Risks:**
- Brute-force attacks against root account
- Credential theft via keylogging or phishing
- Lateral movement if credentials compromised

**Recommendation:**
1. **Disable Root SSH Login**: Create dedicated user accounts with sudo access
2. **Enable Key-Based Authentication Only**: Disable password authentication
3. **Use Fail2Ban**: Automatically block IPs after failed login attempts
4. **Restrict SSH Sources**: Allow SSH only from specific IP ranges (e.g., home network, VPN)

Example `/etc/ssh/sshd_config` hardening:
```bash
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

## Data Protection

### Encryption at Rest

**Current State:**
- **Proxmox Storage**: No encryption enabled by default (LVM thin pool)
- **Kubernetes Secrets**: Stored unencrypted in etcd (default behavior)
- **Longhorn Volumes**: No encryption at rest (data stored on worker node disks)

**Recommendations:**
1. **Enable LUKS Encryption**: Encrypt physical disks on worker nodes (adds complexity)
2. **Encrypt Kubernetes Secrets**: Use Sealed Secrets or External Secrets Operator
3. **Longhorn Volume Encryption**: Configure Longhorn to use encrypted volumes (if supported by underlying storage)

### Backup Security

**Current State:**
- Backups stored locally on `/mnt/pve/backups` (dedicated 2.7TB drive)
- No encryption applied to backup files by default
- Backups not replicated offsite (single point of failure for disaster recovery)

**Recommendations:**
1. **Encrypt Backups**: Use `gpg` or `age` to encrypt backup archives before storage
2. **Offsite Replication**: Configure Proxmox Backup Server (PBS) with remote repository
3. **Backup Integrity Verification**: Regularly test restore procedures and verify checksums

## Application Security

### Kubernetes Pod Security

**Current State:**
- No PodSecurityPolicy or PodSecurityStandards enforced
- Pods run with default security context (no restrictions on privileges, capabilities, or seccomp profiles)

**Recommendations:**
1. **Enable Pod Security Standards**: Implement `restricted` standard for production namespaces
2. **Run Containers as Non-Root**: Set `securityContext.runAsNonRoot: true` in pod specs
3. **Read-Only Root Filesystem**: Enable `readOnlyRootFilesystem: true` where possible
4. **Drop All Capabilities**: Use `capabilities.drop: ["ALL"]` unless specific capabilities required

Example pod security context:
```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

### Network Policies

**Current State:**
- No network policies configured (all pods can communicate freely)

**Recommendations:**
1. **Implement Default Deny**: Apply `NetworkPolicy` with `podSelector: {}` and `ingress: []` to restrict all inbound traffic
2. **Allow Specific Communications**: Add ingress/egress rules for required service-to-service communication
3. **Isolate Sensitive Services**: Restrict access to databases, APIs, and internal services

Example default deny network policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### API Security

**Current State:**
- Kubernetes API server exposed via LoadBalancer IP (MetalLB)
- No rate limiting configured on Traefik ingress
- CORS headers not enforced by default

**Recommendations:**
1. **Enable Rate Limiting**: Add Traefik middleware for request rate limiting
2. **Configure CORS**: Explicitly set allowed origins, methods, and headers via Traefik middleware
3. **API Authentication**: Implement JWT or OAuth2 for protected API endpoints
4. **Disable Unused Endpoints**: Remove or restrict access to deprecated/unused API versions

## Monitoring & Logging

### Security Event Logging

**Current State:**
- Proxmox audit logs available in web GUI → Datacenter → Audit Log
- Kubernetes audit logging not enabled by default (can be enabled via k3s configuration)
- OPNsense firewall logs accessible via web GUI

**Recommendations:**
1. **Enable Kubernetes Audit Logging**: Configure k3s to log API server requests to file or external SIEM
2. **Centralized Logging**: Forward all logs to a centralized system (e.g., Loki, ELK stack)
3. **Alert on Anomalies**: Set up alerts for failed login attempts, privilege escalation, etc.

### Intrusion Detection

**Current State:**
- No IDS/IPS deployed in the homelab

**Recommendations:**
1. **OPNsense IDS/IPS**: Enable built-in Snort or Suricata ruleset in OPNsense
2. **Host-Based IDS**: Deploy OSSEC or Wazuh on critical servers for file integrity monitoring
3. **Network Monitoring**: Use Zeek (Bro) or Suricata for network traffic analysis

## Incident Response

### Detection & Containment

**Procedures:**
1. **Identify Compromise**: Review audit logs, firewall logs, and Kubernetes events for anomalies
2. **Contain Threat**: Isolate affected systems (disable network interfaces, stop pods, etc.)
3. **Preserve Evidence**: Capture memory dumps, disk images, and log snapshots before remediation
4. **Eradicate Root Cause**: Patch vulnerabilities, rotate credentials, remove malicious artifacts

### Recovery & Lessons Learned

**Procedures:**
1. **Restore from Backups**: Use known-good backups to restore affected systems
2. **Verify Integrity**: Scan restored systems for malware or unauthorized changes
3. **Document Incident**: Record timeline, impact, root cause, and remediation steps
4. **Update Security Posture**: Implement controls to prevent similar incidents in the future

## Compliance Considerations

### Data Privacy (GDPR, CCPA)

If handling personal data (user emails, trading portfolios, etc.):
- **Data Minimization**: Collect only necessary user data
- **Encryption in Transit**: Enable HTTPS for all external-facing services
- **Right to Erasure**: Implement mechanisms for users to request data deletion
- **Consent Management**: Obtain explicit consent before collecting personal information

### Security Standards

While not formally certified, the homelab aligns with:
- **CIS Benchmarks**: Follow CIS recommendations for Kubernetes and Linux hardening
- **NIST Cybersecurity Framework**: Implement core functions (Identify, Protect, Detect, Respond, Recover)
- **OWASP Top 10**: Address common web application security risks

## Security Audit Checklist

### Quarterly Review Items
- [ ] Review Proxmox user permissions and remove unused accounts
- [ ] Rotate SSH keys and API tokens
- [ ] Update Kubernetes cluster to latest stable version
- [ ] Review and update firewall rules in OPNsense
- [ ] Test backup restore procedures
- [ ] Scan for known vulnerabilities (use tools like Trivy, Grype)
- [ ] Review TLS certificate expiry dates
- [ ] Audit Kubernetes RBAC roles and bindings

---

*Security documentation. Specific network configurations, IP addresses, and internal policies have been generalized for public viewing.*
