# Hardware Specifications

## Proxmox Host Server

### CPU
- **Model**: Intel Xeon E5-2667 v3 @ 3.20GHz (Dual Socket)
- **Cores**: 16 total (8 cores per socket)
- **Architecture**: x86_64 with hardware virtualization support (VT-x/VT-d)
- **Cache**: 20 MB L3 cache

### Memory
- **Total RAM**: 128 GB DDR4
- **Usage Pattern**: 
  - ~46 GB actively used by VMs and services
  - ~73 GB available for workload scaling
  - 8 GB swap configured

### Storage Layout

| Disk | Size | Purpose | Configuration |
|------|------|---------|---------------|
| `/dev/sda` | 111 GB | Proxmox OS + local storage | LVM with root, swap, and data partitions |
| `/dev/sdb` | 2.7 TB | Backup storage | Direct mount at `/mnt/pve/backups` |
| `/dev/sdc` | 30 TB | VM/container storage pool | LVM thin provisioning (MainThin) |

#### Storage Pool Details (MainThin - 30TB)
- **Type**: LVM Thin Provisioning
- **Volume Group**: MainThin
- **Thin Pool**: MainThin
- **Usage**: Hosts all VM disk images and container storage
- **Allocated VMs**: ~40+ virtual disks across multiple VMs

### Network Interface Cards (NICs)
- **Quantity**: 4x Intel Ethernet NICs (dual-port per socket)
- **Bonding Mode**: LACP (802.3ad) for link aggregation
- **Aggregate Bandwidth**: 4 Gbps theoretical maximum
- **Bridge**: All bonded interfaces bridge to `vmbr0`

### Form Factor & Chassis
- **Type**: Rack-mount or Tower server (enterprise-class)
- **Power Supply**: Likely redundant PSUs (typical for dual-socket Xeon systems)
- **Management**: IPMI/iLO for out-of-band management

## Network Switching

- **Switch Type**: Managed LACP-capable switch
- **Port Configuration**: 4x ports dedicated to Proxmox host (LAG/bond0)
- **VLAN Support**: Full VLAN tagging on uplink port(s)
- **Uplink to OPNsense**: Single or bonded link carrying all VLANs

## Networking Hardware Summary

| Component | Specification |
|-----------|---------------|
| Host NICs | 4x Intel Ethernet (dual-port) |
| Bonding | LACP (802.3ad), 4-link aggregation |
| Switch | Managed switch with VLAN support |
| Firewall | OPNsense VM on dedicated network interface |

## Power & Cooling Considerations

- **Power Consumption**: Estimated 300-500W under load (dual Xeon + multiple drives)
- **Cooling**: Standard server chassis cooling (front-to-back airflow)
- **UPS Recommended**: Yes, for graceful shutdown capability

---

*Hardware documentation sanitized for public viewing. Specific model numbers and identifiers have been generalized.*
