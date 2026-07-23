# Network Architecture

## Topology

```
Internet
    │
    ▼
Home Router
    │
    ▼
TP-Link Switch
    │
    ▼ 
[OPNsense VM] ← WAN via vmbr0 (Treats home LAN as WAN)
    │
    ├── LAN: 10.10.x.1/16 → All VM containers
    │     Used for: pod networking, inter-VM communication
    │
    └── vmbr0 bridge on Proxmox host
          ├─ Management IP (untagged)
          └─ Uplink to TP-Link T1600G-28TS switch
```

The OPNsense VM sits between the ISP and the internal network, handling NAT, routing, and firewall rules. The Proxmox host connects via a 4-link LACP bond (`bond0`) that bridges into `vmbr0`. All traffic currently flows untagged on `vmbr0` to the switch uplink.

## Proxmox Host Network

- **Bond**: `bond0` — 802.3ad LACP across 4 NICs (eno1–eno4)
- **Bridge**: `vmbr0` — management network, currently untagged only
- **Secondary bridge**: `vmbr1` — used by OPNsense VM for LAN interface
- **Tailscale**: `tailscale0` — WireGuard interface for remote access

## Firewall Configuration (OPNsense VM)

- **WAN**: DHCP from home router
- **LAN**: `10.10.x.1/16` on `vmbr1` bridge
- **NAT**: Masquerade for all internal subnets via WAN
- **Default policy**: Allow outbound, deny inbound (with explicit port forwards as needed)

## DNS

- **External**: Cloudflare domain records (A/CNAME records pointing to public IP)
- **Internal**: Kubernetes CoreDNS handles in-cluster service discovery (`<service>.<namespace>.svc.cluster.local`)
