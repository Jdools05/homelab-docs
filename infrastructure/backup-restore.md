# Backup and Restore Strategy

## Overview

Backups cover three layers: Proxmox VM images, Kubernetes cluster state (etcd), and application data (Longhorn PVC snapshots). All backups are stored locally on a dedicated 2.7 TB drive — no offsite replication is configured.

## What's Backed Up

### 1. Proxmox VMs (Primary)
- **Tool**: `vzdump` (Proxmox native utility)
- **Storage**: `/mnt/pve/backups` on dedicated 2.7 TB drive
- **Compression**: zstd
- **Schedule**: Daily incremental backups

### 2. Kubernetes Cluster State (etcd)
- **Tool**: k3s built-in etcd backup
- **Frequency**: Before major changes or on a regular schedule
- **Storage**: Local backup storage on the control plane node

### 3. Application Data (Longhorn PVC Snapshots)
- **Tool**: Longhorn UI or `lhctl` CLI
- **Frequency**: Daily snapshots for critical volumes
- **Storage**: Local Longhorn replicas (no S3/offsite configured)

## Recovery Procedures

| Scenario | First Steps |
|----------|-------------|
| Single VM failure | Restore from latest vzdump backup |
| Storage pool issue | Replace disk(s), recreate LVM, restore VMs from backup drive |
| Complete host failure | Install Proxmox on new hardware, mount backup drive, restore OPNsense → K8s control plane → workers (dependency order) |
| Kubernetes corruption | Restore etcd snapshot, rejoin worker nodes if needed |

## Monitoring Backup Health

- **List recent backups**: `pvebackup list local-backup --sort -start`
- **Check backup storage usage**: `df -h /mnt/pve/backups`

---

*Backup strategy documentation. Actual schedules and retention policies reflect current operational settings.*
