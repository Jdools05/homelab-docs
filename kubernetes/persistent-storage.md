# Persistent Storage (Longhorn)

## Overview

The homelab uses Longhorn v1.7 as the distributed block storage system for Kubernetes persistent volumes. Longhorn provides data redundancy through replication across multiple nodes, ensuring durability even if a single node fails.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              Longhorn Distributed Storage                   │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐               │
│  │ Control Plane   │    │ Worker Node #1  │               │
│  │ (k8s-control-1) │    │ (k8s-worker-1)  │               │
│  │                 │    │                 │               │
│  │ - Manager       │    │ - Engine Image  │               │
│  │ - CSI Plugin    │    │ - Instance Mgr  │               │
│  │ - UI            │    │ - Longhorn Mgr  │               │
│  └─────────────────┘    └─────────────────┘               │
│                                                             │
│  ┌─────────────────┐                                         │
│  │ Worker Node #2  │                                         │
│  │ (k8s-worker-2)  │                                         │
│  │                 │                                         │
│  │ - Engine Image  │                                         │
│  │ - Instance Mgr  │                                         │
│  │ - Longhorn Mgr  │                                         │
│  └─────────────────┘                                         │
└─────────────────────────────────────────────────────────────┘

Data Flow:
Pod → PVC → Longhorn Volume (replicated 2x across nodes) → Physical Disk on Worker Nodes
```

## Storage Classes

### longhorn (Default)
| Attribute | Value |
|-----------|-------|
| **Provisioner** | `driver.longhorn.io` |
| **Reclaim Policy** | Delete (volumes deleted with PVC) |
| **Volume Binding Mode** | Immediate (allocate on creation) |
| **Allow Volume Expansion** | Yes |
| **Default Storage Class** | Yes |

### longhorn-static
| Attribute | Value |
|-----------|-------|
| **Provisioner** | `driver.longhorn.io` |
| **Reclaim Policy** | Delete |
| **Volume Binding Mode** | Immediate |
| **Allow Volume Expansion** | Yes |
| **Default Storage Class** | No (explicitly named volumes only) |

## Persistent Volume Claims (PVCs)

### Current PVC Inventory

| Namespace | PVC Name | Size | Access Mode | Storage Class | Status | Bound To |
|-----------|----------|------|-------------|---------------|--------|----------|
| `monitoring` | prometheus-server | 8 GB | RWO | longhorn | Bound | prometheus-server-0 |
| `monitoring` | storage-prometheus-alertmanager-0 | 2 GB | RWO | longhorn | Bound | prometheus-alertmanager-0 |

### PVC Details

#### Prometheus Server (8 GB)
- **Purpose**: Stores Prometheus metrics data
- **Access Mode**: ReadWriteOnce (single node access)
- **Replication Factor**: 2x (default Longhorn setting)
- **Actual Disk Usage**: ~8 GB physical (4 GB per replica)

#### Alertmanager Storage (2 GB)
- **Purpose**: Stores alert routing configuration and history
- **Access Mode**: ReadWriteOnce
- **Replication Factor**: 2x
- **Actual Disk Usage**: ~4 GB physical (2 GB per replica)

## Longhorn Volume Management

### Creating a New Volume via PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi  # Request 10 GB (actual usage depends on replication)
```

### Volume Replication
- **Default Replication Factor**: 2 (each volume has 2 copies across different nodes)
- **Purpose**: Data durability - can survive one node failure
- **Storage Overhead**: ~100% (2x the requested size in physical disk usage)

**Example**: A 10 GB PVC with replication factor 2 uses ~20 GB physical storage.

### Volume Expansion
Longhorn supports online volume expansion:
```bash
# Expand PVC from 10Gi to 20Gi
kubectl patch pvc <pvc-name> -n <namespace> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Verify expansion
kubectl get pvc <pvc-name> -n <namespace>
```

## Longhorn UI

### Access
- **Service**: `longhorn-frontend` (ClusterIP)
- **Port**: 80/TCP
- **URL**: Internal only (not exposed externally by default)
- **Authentication**: None (internal network only)

### Features
- View all volumes and their status
- Create/delete snapshots
- Manage backups (if S3 configured)
- Monitor volume health and replication status
- View engine logs for troubleshooting

## Backup Configuration (Optional)

Longhorn can backup volumes to S3-compatible storage:

1. **Configure S3 Bucket**:
   - Settings → Backups → Add Backup Target
   - Enter S3 endpoint, access key, secret key, bucket name

2. **Create Backup Policy**:
   - Select volume in Longhorn UI
   - Click "Backup" → "Schedule Backup"
   - Configure schedule (e.g., daily at 2 AM)

3. **Restore from Backup**:
   - Create new volume from backup snapshot
   - Attach to pod as usual

## Monitoring Longhorn Health

### CLI Commands
```bash
# List all volumes
kubectl get pv -o wide | grep longhorn

# Check volume status
kubectl describe pv <volume-name>

# View Longhorn manager logs
kubectl logs -n longhorn-system deployment/longhorn-manager

# Check engine images (storage engines)
kubectl get pods -n longhorn-system -l longhorn.io/component=engine-image
```

### Health Indicators
- **Healthy**: All replicas synchronized, replication factor met
- **Degraded**: One replica missing (awaiting recovery or replacement)
- **Critical**: Multiple replicas missing, data at risk

## Performance Considerations

### I/O Patterns
- **Read Performance**: Good - can read from any replica
- **Write Performance**: Slightly slower due to replication overhead (~10-20% penalty)
- **Recommendation**: Use for non-latency-sensitive workloads (databases, file storage, backups)

### SSD vs HDD
- Current setup uses standard HDDs on worker nodes
- For improved performance: Consider adding NVMe SSDs and configuring Longhorn to prefer SSD replicas

## Security Considerations

- **Encryption at Rest**: Not enabled by default (would require additional configuration)
- **Access Control**: Longhorn volumes accessible to any pod with matching PVC
- **Recommendation**: Implement PodSecurityPolicy to restrict volume access

## Maintenance Procedures

### Replacing a Failed Node
1. Create new VM with same specifications
2. Join to k3s cluster: `k3s join <control-plane-ip> --token <token>`
3. Longhorn automatically replicates data to new node
4. Wait for replication to complete (check Longhorn UI)
5. Drain old failed node and remove from cluster

### Expanding Storage Capacity
1. Add new physical disk to worker node
2. Create new LVM volume group on new disk
3. Configure Longhorn to use additional disk:
   ```bash
   # On the worker node
   longhorn-cli driver add-instance-manager --disk /dev/sdX
   ```
4. Verify new disk appears in Longhorn UI

---

*Persistent storage documentation. Specific volume sizes, replication factors, and internal configurations have been generalized for public viewing.*
