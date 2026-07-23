# Proxmox VE Configuration

## Overview

The homelab runs on Proxmox Virtual Environment (PVE) 8.0, providing hardware virtualization for all services. Storage uses LVM thin provisioning across a single-node setup with no high-availability cluster configured.

## Storage Configuration

| Storage Pool | Type | Capacity | Purpose |
|--------------|------|----------|---------|
| **MainThin** | LVM Thin Provisioning | 30 TB (`/dev/sdc`) | All VM disk images and container storage |
| **pve (local-lvm)** | LVM Thick Provisioning | ~110 GB (`/dev/sda`) | Proxmox host OS, templates, ISOs |
| **backups** | Directory mount | 2.7 TB (`/dev/sdb`) | VM backups and disaster recovery data |

## Network Configuration

- **bond0**: 4x NIC LACP bond → `vmbr0` bridge (untagged management network)
- **vmbr1**: Secondary bridge for OPNsense VM LAN interface

## Management Access

- **Web GUI**: `https://<proxmox-ip>:8006` (internal network)
- **CLI**: SSH root access from authorized IPs
- **Remote**: Via Tailscale VPN or SSH tunnel
- **iDRAC**: Dedicated out-of-band management port on the server

## Backup Strategy

VMs are backed up using Proxmox's native `vzdump` utility to `/mnt/pve/backups`. See [backup-restore.md](./backup-restore.md) for full procedures. No Proxmox Backup Server (PBS) instance is currently deployed — backups live on the dedicated 2.7 TB drive.
