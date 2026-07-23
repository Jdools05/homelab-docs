# Persistent Storage (Longhorn)

## Overview

The cluster uses Longhorn v1.7 as its distributed block storage system. It runs as a DaemonSet across all 3 nodes, providing persistent volumes with automatic 2x replication for data durability even if one node fails. Volumes are allocated from the underlying Proxmox MainThin pool (30 TB total).

## Architecture

```
Pod → PVC → Longhorn Volume (replicated 2x across nodes)
                              → Physical disk on worker nodes (via MainThin pool)
```

Longhorn runs as a DaemonSet with manager, CSI plugins, and engine images on every node. The Longhorn UI is available internally for volume management and snapshot creation.

## Storage Classes

| Class | Provisioner | Reclaim Policy | Default? |
|-------|-------------|----------------|----------|
| `longhorn` | `driver.longhorn.io` | Delete | Yes |
| `longhorn-static` | `driver.longhorn.io` | Delete | No (explicit PVCs only) |

## Current PVCs

| Namespace | PVC Name | Size | Access Mode | Status |
|-----------|----------|------|-------------|--------|
| `monitoring` | prometheus-server | 8 GB | RWO | Bound |
| `monitoring` | storage-prometheus-alertmanager-0 | 2 GB | RWO | Bound |

These are the only PVCs currently in use — monitoring data on Longhorn with 2x replication.

## Key Features

- **Replication**: 2x by default (each volume has 2 copies across different nodes)
- **Volume expansion**: Supported online via `kubectl patch pvc`
- **Snapshots**: Create/delete snapshots from the Longhorn UI or CLI

---

*Persistent storage documentation. Specific volume sizes and internal configurations have been generalized.*
