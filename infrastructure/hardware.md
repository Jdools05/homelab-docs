# Hardware Overview

## Proxmox Host Server

| Component | Specification |
|-----------|---------------|
| **CPU** | Intel Xeon E5-2667 v3 (Dual Socket, 16 cores total) |
| **RAM** | 128 GB DDR4 |
| **OS Disk** | ~111 GB (`/dev/sda`) — Proxmox OS + local storage |
| **Backup Disk** | 2.7 TB (`/dev/sdb`) — dedicated backup drive |
| **VM Storage** | 30 TB LVM Thin Pool (`/dev/sdc`) — hosts all VM disk images |
| **Network** | 4x Intel Ethernet NICs bonded via LACP → `vmbr0` bridge |

The host runs as a rack-mount server with redundant PSUs and dedicated iDRAC for remote management.

## Network Switching

- **Switch**: TP-Link T1600G-28TS smart switch
- 4 ports dedicated to Proxmox (LAG/bond0)
- Uplink connects to OPNsense firewall VM

## Networking Hardware Summary

| Component | Specification |
|-----------|---------------|
| Host NICs | 4x Intel Ethernet (dual-port) |
| Bonding | LACP (802.3ad), 4-link aggregation |
| Switch | TP-Link T1600G-28TS managed switch |
| Firewall | OPNsense VM |
