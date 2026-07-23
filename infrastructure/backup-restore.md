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
| Kubernetes corruption | Restore etcd snapshot, rejoin worker nodes if needed |

