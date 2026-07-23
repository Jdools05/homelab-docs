# Changelog

Significant infrastructure changes, upgrades, and maintenance activities on the homelab environment.

## 2026-07-21 - Initial Documentation Repository Created
- **Component**: All
- **Description**: Created comprehensive documentation repository at `D:\Projects\homelab-docs` covering infrastructure, Kubernetes cluster, services, and websites. Includes sanitized versions suitable for public viewing (portfolio/employer review).
- **Impact**: None (documentation only)
- **Reason**: Centralize operational knowledge, enable knowledge sharing, create portfolio piece for future employers

## 2026-XX-XX - Kubernetes Cluster Initial Deployment
- **Component**: Kubernetes (k3s v1.34.5)
- **Description**: Deployed initial k3s cluster with 3 nodes (1 control plane + 2 workers). Installed Traefik ingress controller, MetalLB for LoadBalancer services, Longhorn for distributed storage, and Prometheus monitoring stack.
- **Impact**: Enabled containerized service hosting on homelab infrastructure
- **Reason**: Replace ad-hoc VM deployments with orchestrated Kubernetes cluster

## 2026-XX-XX - OPNsense Firewall VM Provisioned
- **Component**: Network / Proxmox
- **Description**: Created OPNsense VM (VMID 128) to serve as primary firewall, NAT gateway. Configured WAN interface with DHCP from ISP and LAN interface on internal k8s network.
- **Impact**: Centralized network security and routing for all homelab traffic
- **Reason**: Replace basic router functionality with full-featured firewall (VPN, IDS/IPS, advanced routing)

## 2026-XX-XX - Proxmox Host Network Bonding Configured
- **Component**: Network / Proxmox
- **Description**: Configured LACP bond0 using 4x Intel Ethernet NICs for link aggregation. Bridge vmbr0 created without VLAN awareness (no VLANs configured yet).
- **Impact**: Increased bandwidth (theoretical 4 Gbps aggregate) and redundancy for Proxmox host management traffic
- **Reason**: Improve network reliability and throughput for hypervisor management

## 2026-XX-XX - Longhorn Distributed Storage Deployed
- **Component**: Kubernetes / Storage
- **Description**: Installed Longhorn v1.7 as distributed block storage system. Configured with 2x replication factor across all worker nodes for data durability.
- **Impact**: Enabled persistent volume provisioning for Kubernetes workloads with automatic data redundancy
- **Reason**: Replace local-path-provisioner (no redundancy) with production-grade distributed storage

## 2026-XX-XX - Mock Trading App Deployed to K8s
- **Component**: Services / Kubernetes
- **Description**: Built and deployed React Native web app (Expo build) via multi-stage Docker image. Configured Traefik ingress for `mock-trading.jdools.com` domain routing to NGINX pod serving static assets.
- **Impact**: Made mock trading application accessible via public URL with HTTPS (planned)
- **Reason**: Provide hosted version of mobile app for web users

## 2026-XX-XX - Landing Page ConfigMap Deployment
- **Component**: Services / Kubernetes
- **Description**: Created ConfigMap (`landing-page-html`) containing index.html, sitemap.xml, and robots.txt. NGINX pod mounts ConfigMap as volume to serve static landing page at `jdools.com`.
- **Impact**: Simplified content updates (edit HTML → apply ConfigMap → automatic rollout)
- **Reason**: Replace traditional file-based deployment with GitOps-friendly ConfigMap approach

## 2026-XX-XX - Power Playlist Service Architecture Established
- **Component**: Services / Kubernetes
- **Description**: Deployed Node.js API backend (port 3000) and Next.js web frontend for Spotify integration service. Configured Traefik ingress with path-based routing (`/api` → API, `/*` → Web).
- **Impact**: Enabled external access to Power Playlist service via dedicated domain
- **Reason**: Host self-managed alternative to commercial playlist management services

## 2026-XX-XX - AOA Marching Cubes Visualization Deployed
- **Component**: Services / Kubernetes
- **Description**: Deployed Java-based 3D surface visualization tool (AOA Marching Cubes) as static web application. Configured Traefik ingress for `aoa-marching-cubes.jdools.com` domain routing.
- **Impact**: Made 3D visualization tool accessible via public URL
- **Reason**: Showcase technical skills and provide interactive demo of algorithm

## Maintenance Notes

### Regular Tasks (Weekly/Monthly)
- Review Proxmox backup success/failure logs
- Check Kubernetes cluster health (`kubectl get nodes`, `kubectl get pods --all-namespaces`)
- Monitor disk usage on Proxmox host and Longhorn volumes
- Update documentation if infrastructure changes occur

### Backup Schedule
- **Daily**: Proxmox VM backups (vzdump) to `/mnt/pve/backups`
- **Weekly**: etcd snapshot backup from control plane node
- **Monthly**: Test restore procedure for critical services

---

*Changelog maintained as part of homelab documentation.*
