# Proxmox VE Configuration

## Overview

The homelab runs on Proxmox Virtual Environment (PVE) 8.0, providing hardware virtualization and container management for all services. The host is configured with LVM thin provisioning for efficient storage allocation across multiple VMs.

## Version Information

- **Proxmox VE**: 8.0 (kernel 7.0.14-5-pve)
- **QEMU/KVM**: Latest stable from Proxmox repository
- **LXC**: Container runtime for lightweight workloads

## Virtual Machine Inventory

### Production VMs

| VM ID | Name | OS | RAM | Disk | Status | Purpose |
|-------|------|-----|-----|------|--------|---------|
| 100 | mceternal | Debian/Ubuntu | 32 GB | 128 GB (thin) | Stopped | Minecraft server or similar workload |
| 104 | dev | Ubuntu | 16 GB | 512 GB (thin) | Stopped | Development environment |
| 105 | nextcloud-vm | Nextcloud | 4 GB | 1 TB + 828 MB (thin) | Running | Cloud storage and file sharing |
| 116 | k8s-control-1 | Ubuntu 24.04 | 16 GB | 128 GB (thin) | Running | Kubernetes control plane node |
| 119 | k8s-worker-1 | Ubuntu 24.04 | 32 GB | 128 GB (thin) | Running | Kubernetes worker node #1 |
| 120 | k8s-worker-2 | Ubuntu 24.04 | 32 GB | 128 GB (thin) | Running | Kubernetes worker node #2 |
| 125 | docker | Ubuntu/Debian | 8 GB | 64 GB (thin) | Stopped | Docker host for non-K8s workloads |
| 128 | opnsense | OPNsense | 8 GB | 20 GB (thin) + 2TB data | Running | Firewall, NAT, and routing |

### VM Storage Breakdown (MainThin Pool)

The 30TB LVM thin pool contains the following virtual disk allocations:

| VM ID | Disk Size | Notes |
|-------|-----------|-------|
| 100 | 128 GB + 4 MB overhead | Primary OS disk + boot partition |
| 104 | 512 GB | Large storage for development workloads |
| 105 | 1 TB + 828 MB + 4 MB | Nextcloud data volume + system |
| 106 | 512 GB | Additional VM storage |
| 107-109 | 4-8 GB each | Smaller utility VMs |
| 110-113 | 4-256 GB | Mixed workload VMs |
| 114-117 | 128-512 GB | Development and testing VMs |
| 118-120 | 128-256 GB | K8s worker nodes with additional storage |
| 121-130 | 8-128 GB | Various utility and service VMs |

**Total Allocated**: ~15+ TB across all VMs (thin-provisioned, actual usage varies)

## Storage Configuration

### Primary Storage Pool (MainThin)
- **Type**: LVM Thin Provisioning
- **Volume Group**: MainThin
- **Thin Pool Name**: MainThin
- **Total Capacity**: 30 TB physical disk (`/dev/sdc`)
- **Overprovisioning Ratio**: ~1.5:1 to 2:1 (allocate more than physical capacity)

### Local Storage (pve)
- **Type**: LVM Thick Provisioning
- **Volume Group**: pve
- **Usage**: Proxmox host OS, local templates, ISOs
- **Capacity**: ~110 GB (`/dev/sda`)

### Backup Storage
- **Location**: `/mnt/pve/backups` (mounted from `/dev/sdb`)
- **Disk**: 2.7 TB dedicated backup drive
- **Contents**: VM backups, configuration snapshots, disaster recovery data

## Container (LXC) Support

Proxmox supports Linux Containers (LXC) for lightweight workloads:
- **No active containers** currently running (all services use full VMs)
- **Container runtime**: LXC with systemd integration
- **Use Case**: Lightweight services that don't require full kernel isolation

## Network Configuration Summary

### Host Networking
- **Bond0**: 4x NIC LACP bond → vmbr0 bridge
- **vmbr0**: Management network (VLAN 1) + trunk for other VLANs
- **vmbr1**: Secondary bridge for OPNsense VM internal networking

### VM Network Assignment
| VM | Interface | Bridge | VLAN | Purpose |
|----|-----------|--------|------|---------|
| k8s-control-1 | e1000/virtio | vmbr0 or vmbr1 | 99 (k8s) | Cluster control plane |
| k8s-worker-1 | e1000/virtio | vmbr0 or vmbr1 | 99 (k8s) | Worker node #1 |
| k8s-worker-2 | e1000/virtio | vmbr0 or vmbr1 | 99 (k8s) | Worker node #2 |
| opnsense | virtio x2 | vmbr0 + vmbr1 | WAN/LAN | Firewall routing |

## Proxmox Cluster Features

### High Availability (HA)
- **Status**: Not configured (single-node homelab)
- **Recommendation**: Consider HA for production workloads if second node added

### Live Migration
- **Capability**: Supported via `qm migrate` command
- **Requirement**: Shared storage (already have MainThin pool)
- **Use Case**: Move VMs between hosts without downtime

### Backup Strategy
- **Tool**: Proxmox Backup Server (PBS) or local backup to `/mnt/pve/backups`
- **Schedule**: Daily incremental backups recommended
- **Retention**: Keep last 7 days of daily backups + weekly full backups

## Management Access

### Web GUI
- **URL**: `https://<proxmox-ip>:8006` (internal management network)
- **Authentication**: PVE user accounts with role-based access control
- **Remote Access**: Via Tailscale VPN or SSH tunnel for secure remote management

### CLI Access
- **SSH**: Root access to Proxmox host from authorized IPs
- **VM Console**: `qm terminal <vmid>` or VNC via web GUI
- **Storage Management**: `pvesm` commands for storage pool operations

## Monitoring & Logging

### System Monitoring
- **Proxmox Built-in**: Resource usage graphs in web GUI
- **External Monitoring**: Prometheus node_exporter on host and VMs
- **Alerting**: Email or webhook notifications for critical events

### Log Locations
- **Host Logs**: `/var/log/pve/`, `/var/log/syslog`
- **VM Logs**: `/var/log/qemu/<vmid>.log`
- **Backup Logs**: `/var/log/pbs/backup.log`

## Maintenance Procedures

### Adding a New VM
1. Create VM via web GUI or `qm create <vmid>`
2. Attach ISO and boot to install OS
3. Configure network interface (virtio recommended for performance)
4. Set up storage disk on MainThin pool
5. Install Proxmox guest agents (`qemu-guest-agent`)

### Resizing a VM Disk
```bash
# Expand virtual disk by 10GB
qm resize <vmid> virtio0 +10G

# Then extend filesystem inside the VM
# For ext4:
resize2fs /dev/vda1
# For XFS:
xfs_growfs /mnt
```

### Backup and Restore
```bash
# Create backup of VM 116 (k8s-control-1)
vzdump 116 --storage local-backup --compress zstd

# List available backups
pvebackup list local-backup

# Restore from backup
qm restore <vmid> <backup-file>
```

---

*Proxmox configuration documentation sanitized for public viewing. Specific VM IDs, storage allocations, and internal network configurations have been generalized.*
