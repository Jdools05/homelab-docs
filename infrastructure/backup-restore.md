# Backup and Restore Strategy

## Overview

This document outlines the backup strategy for the homelab infrastructure, including Proxmox VM backups, Kubernetes cluster state, and application data. The goal is to ensure recoverability in case of hardware failure, accidental deletion, or corruption.

## Backup Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Backup Strategy                          │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐               │
│  │ Proxmox Host   │───►│ Local Backup   │               │
│  │ (Source)       │    │ Storage (2.7TB)│               │
│  └─────────────────┘    └─────────────────┘               │
│         │                       │                          │
│         ▼                       ▼                          │
│  ┌─────────────────┐    ┌─────────────────┐               │
│  │ K8s Cluster     │───►│ Etcd Snapshots │               │
│  │ State (etcd)    │    │ + PVC Backups  │               │
│  └─────────────────┘    └─────────────────┘               │
│                                                             │
│  Retention: Last 7 days daily + weekly full backups        │
└─────────────────────────────────────────────────────────────┘
```

## Backup Components

### 1. Proxmox VM Backups (Primary)

**What's Backed Up:**
- All VM disk images (qcow2/raw on MainThin pool)
- VM configuration files (`/etc/pve/qemu-server/<vmid>.conf`)
- BIOS settings and boot order
- Network interface assignments

**Backup Method:**
- **Tool**: `vzdump` (Proxmox native backup utility)
- **Storage**: Local backup storage (`/mnt/pve/backups`)
- **Compression**: zstd (fast compression with good ratio)
- **Schedule**: Daily incremental backups at 2:00 AM

**Backup Command:**
```bash
# Backup VM 116 (k8s-control-1)
vzdump 116 --storage local-backup --compress zstd --remove 0

# Backup all production VMs
vzdump 105 116 119 120 128 --storage local-backup --compress zstd --mailto admin@example.com
```

**Backup Retention:**
- **Daily**: Keep last 7 days of incremental backups
- **Weekly**: Keep one full backup per week for 4 weeks
- **Monthly**: Archive to external storage quarterly

### 2. Kubernetes Cluster State (etcd)

**What's Backed Up:**
- etcd database snapshot (cluster state, configs, secrets)
- PersistentVolumeClaim (PVC) data via Longhorn snapshots
- Helm release history and values

**Backup Method:**
- **Tool**: `etcdctl snapshot save` or k3s built-in etcd backup
- **Frequency**: Every 6 hours or before major changes
- **Storage**: Local backup storage + remote offsite copy (optional)

**Backup Command:**
```bash
# Create etcd snapshot on control plane node
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/rancher/k3s/server/tls/etcd-server-ca.crt \
  --cert=/etc/rancher/k3s/server/tls/etcd-server.crt \
  --key=/etc/rancher/k3s/server/tls/etcd-server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-$(date +%Y%m%d).db --write-out=table
```

**Restore Procedure:**
```bash
# Stop k3s service
systemctl stop k3s

# Restore from snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/rancher/k3s/server/db

# Restart k3s
systemctl start k3s

# Verify cluster health
kubectl get nodes
kubectl get pods --all-namespaces
```

### 3. Application Data (PVC Backups)

**What's Backed Up:**
- Longhorn volume snapshots for PVC-backed services
- Specific data volumes (Nextcloud, monitoring, etc.)

**Backup Method:**
- **Tool**: Longhorn UI or `lhctl` CLI
- **Frequency**: Daily snapshots + weekly full exports
- **Storage**: Local Longhorn replicas + optional S3/offsite export

**Longhorn Backup Procedure:**
1. Open Longhorn Manager UI (port 9500)
2. Select volume → Create Snapshot
3. Add snapshot name and description
4. For offsite backup: Configure S3 bucket in Settings → Backups
5. Run backup job manually or schedule via cron

**Restore Procedure:**
1. Delete existing PVC (data will be lost without restore)
2. Create new PVC with same storage class and size
3. In Longhorn UI, select volume → Restore from Snapshot
4. Choose snapshot and confirm restoration
5. Update pod to use restored PVC

## Backup Verification

### Daily Checks
- [ ] Verify backup job completed successfully (check Proxmox web GUI)
- [ ] Review backup logs for errors (`/var/log/pve/backup.log`)
- [ ] Confirm etcd snapshot created (if scheduled daily)
- [ ] Check Longhorn backup jobs status

### Weekly Verification
- [ ] Test restore of one non-production VM to isolated network
- [ ] Verify etcd snapshot can be restored in test environment
- [ ] Review backup storage capacity (ensure <80% full)
- [ ] Update backup retention policy if needed

### Monthly Audit
- [ ] Calculate total backup size and growth rate
- [ ] Verify offsite replication status (if configured)
- [ ] Test full cluster restore procedure in lab environment
- [ ] Review and update backup documentation

## Recovery Procedures

### Scenario 1: Single VM Failure
**Procedure:**
1. Identify failed VM from Proxmox web GUI or alerts
2. If disk corruption: Create new VM with same config, restore from latest snapshot
3. If complete loss: Restore entire VM from vzdump backup
4. Verify service functionality after restore

### Scenario 2: Storage Pool Failure (MainThin)
**Procedure:**
1. Replace failed physical disk(s)
2. Recreate LVM volume group and thin pool
3. Restore VMs from local backup storage (`/mnt/pve/backups`)
4. Verify all services are operational

### Scenario 3: Complete Host Failure
**Procedure:**
1. Provision new hardware or use spare host
2. Install Proxmox VE on new host
3. Mount backup storage and restore critical VMs first (OPNsense, k8s-control-1)
4. Restore remaining VMs in order of dependency
5. Reconfigure network bonding and bridge settings
6. Verify cluster connectivity and service functionality

### Scenario 4: Kubernetes Cluster Corruption
**Procedure:**
1. If etcd corrupted: Restore from latest etcd snapshot (see above)
2. If worker node failed: Replace VM, rejoin to cluster via `k3s join` command
3. If control plane lost: Restore from backup or rebuild with same token
4. Verify all pods running and services accessible

## Monitoring Backup Health

### Proxmox Web GUI
- Navigate to Datacenter → Backup
- Check "Last State" column for success/failure status
- Review "Duration" for performance trends

### CLI Monitoring
```bash
# List recent backups
pvebackup list local-backup --sort -start

# Check backup storage usage
df -h /mnt/pve/backups

# View backup logs
tail -f /var/log/pve/backup.log
```

### Alerting
- **Email**: Configure `--mailto` flag in vzdump for failure notifications
- **Webhook**: Use Proxmox VE API to send alerts to monitoring system
- **Prometheus**: Export backup job metrics via custom exporter (optional)

## Documentation Updates

This document should be updated when:
- Backup schedule changes
- New VMs or services are added
- Storage capacity thresholds are reached
- Recovery procedures are tested and refined
- Offsite replication is configured

---

*Backup strategy documentation. Actual backup schedules, retention policies, and storage locations should be verified against current operational settings.*
