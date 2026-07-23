# Security Overview

## Network Security

### Firewall Configuration (OPNsense VM)
- OPNsense VM acts as the primary firewall between WAN and internal networks
- NAT configured for all internal subnets
- Port forwarding enabled for specific services exposed externally
- Default policy: allow outbound, deny inbound with explicit port forwards

### Network Segmentation
No VLAN segmentation is currently configured — all traffic flows untagged on `vmbr0`. The planned layout includes:
- VLAN 1 (Management network)
- VLAN 99 (Kubernetes cluster network)
- VLANs 10–40 (Service-specific networks, reserved for future use)

## Access Control

### Proxmox VE Access
- Username/password authentication for web GUI and API
- Role-based access control (RBAC) with predefined roles: `PVEAdmin`, `PVEAuditor`, and custom roles
- SSH key-based authentication available for CLI access
- Two-factor authentication (2FA) available but not enabled by default

### Kubernetes RBAC
- Kubernetes native RBAC enabled (default in k3s)
- ClusterRole/ClusterRoleBinding resources manage cluster-wide permissions
- Namespace-scoped Roles and RoleBindings for per-namespace access
- Default roles: `cluster-admin`, `admin`, `edit`, `view`

### SSH Access
- Root SSH access enabled on Proxmox host and Kubernetes nodes
- Password authentication allowed alongside key-based auth

## Data Protection

### Encryption at Rest
No encryption is currently enabled by default (LVM thin pool, unencrypted Kubernetes secrets in etcd, Longhorn volumes on worker node disks).

### Backup Security
Backups stored locally on `/mnt/pve/backups` (dedicated 2.7TB drive). No encryption applied to backup files by default. No offsite replication configured.

## Application Security

### Kubernetes Pod Security
No PodSecurityPolicy or PodSecurityStandards enforced. Pods run with default security context (no restrictions on privileges, capabilities, or seccomp profiles).

### Network Policies
No network policies configured — all pods can communicate freely across namespaces.

### API Security
Kubernetes API server exposed via LoadBalancer IP (MetalLB). No rate limiting configured on Traefik ingress. CORS headers not enforced by default.

## Monitoring & Logging

- Proxmox audit logs available in web GUI → Datacenter → Audit Log
- Kubernetes audit logging not enabled by default
- OPNsense firewall logs accessible via web GUI
- Prometheus monitoring stack collects system and application metrics (internal only)

---

*Security documentation. Specific network configurations, IP addresses, and internal policies have been generalized.*
