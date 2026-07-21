# Network Architecture

## Network Topology Overview

```
Internet
    │
    ▼
[OPNsense VM] ← WAN (DHCP from ISP)
    │
    ├── VLAN 1 (Management) ──► Proxmox Host: 192.168.x.x/24
    │                                ├─ Gateway: Managed Switch (.x.1 or .x.4)
    │                                └─ Used for: SSH, Proxmox GUI, management
    │
    ├── VLAN 99 (Kubernetes) ──► k3s Cluster Network
    │                                ├─ OPNsense LAN IP: 10.10.x.1/16
    │                                ├─ Subnets: 10.10.30.0/24, etc.
    │                                └─ Used for: Pod networking, inter-VM comms
    │
    └── VLANs 10-40 (Services) ──► Various service networks
                                       ├─ VLAN 10: 10.10.10.0/24
                                       ├─ Other VLANs: Service-specific
                                       └─ Used for: Isolated service networks
```

## Proxmox Host Network Configuration

### Bonded Interfaces (bond0)
- **Bond Mode**: 802.3ad (LACP) - requires switch-side LAG configuration
- **Member Interfaces**: eno1, eno2, eno3, eno4 (all 4 NICs)
- **Hash Policy**: layer3+4 (balanced distribution across links)
- **MII Monitoring**: 100ms interval for failover detection

### Bridge Configuration (vmbr0)
- **Bridge Type**: Linux bridge with VLAN awareness enabled
- **VLAN IDs**: 1, 10, 20, 30, 40, 99 (tagged traffic passes through)
- **Management IP**: `192.168.x.x/24` on untagged VLAN 1
- **Forwarding Rules**: iptables rules for NAT/routing to internal networks

### Network Interface Summary

| Interface | Type | Role | Configuration |
|-----------|------|------|---------------|
| `bond0` | Bond (LACP) | Uplink to switch | Slave of vmbr0 |
| `vmbr0` | Bridge | Proxmox management | IP on VLAN 1, VLAN-aware |
| `vmbr1` | Bridge | Secondary network | Used by OPNsense VM |
| `tailscale0` | Tailscale | VPN/WAN access | WireGuard interface for remote access |

## VLAN Design

### VLAN 1 - Management Network
- **Purpose**: Host management, SSH, Proxmox web GUI
- **Subnet**: `192.168.x.0/24` (private RFC1918)
- **Gateway**: Managed switch or OPNsense LAN interface
- **Access**: Restricted to authorized management stations

### VLAN 99 - Kubernetes Cluster Network
- **Purpose**: Internal k3s cluster communication
- **Subnet**: `10.10.x.0/16` (large private range for pod networking)
- **Gateway**: OPNsense VM LAN interface (`10.10.x.1`)
- **Routing**: Static routes via OPNsense to other internal networks

### VLAN 10 - Service Network
- **Purpose**: Isolated service network
- **Subnet**: `10.10.10.0/24`
- **Use Case**: Dedicated network for specific services requiring isolation

### Additional VLANs (20, 30, 40)
- **Purpose**: Future service segmentation or IoT device networks
- **Status**: Configured but may be unused or reserved

## Firewall Configuration (OPNsense VM)

### Network Interfaces
- **WAN**: DHCP from ISP (external internet)
- **LAN**: `10.10.x.1/16` on vmbr1 bridge (internal k8s network)
- **OPT1/OPT2**: Available for additional networks if needed

### NAT Rules
- **Masquerade**: WAN interface masquerading for all internal subnets
- **Port Forwarding**: Specific ports forwarded to services as needed
- **Source NAT**: Outbound traffic from k8s cluster appears as OPNsense IP

### Firewall Rules (High-Level)
- **WAN → LAN**: Allow established/related, deny all by default
- **LAN → WAN**: Allow all outbound (standard home lab behavior)
- **Inter-VLAN**: Controlled routing via OPNsense firewall rules
- **K8s Internal**: Full internal connectivity for cluster operations

## DNS Configuration

### Internal DNS
- **Provider**: k3s built-in CoreDNS or external DNS server
- **Resolution**: Services resolve via Kubernetes DNS (`<service>.<namespace>.svc.cluster.local`)
- **External DNS**: Managed by domain registrar (Cloudflare, Namecheap, etc.)

### External DNS Records
| Domain | Type | Value | Purpose |
|--------|------|-------|---------|
| `*.jdools.com` | CNAME/A | Proxmox/Public IP | Wildcard for services |
| `jdools.com` | A | Public IP | Landing page |
| `mock-trading.jdools.com` | A/CNAME | Public IP or CDN | Mock trading web app |
| `power-playlist.jdools.com` | CNAME | Cloudflare Pages/VPS | Spotify integration service |

## Network Security Considerations

- **DMZ Equivalent**: OPNsense VM acts as firewall between WAN and internal networks
- **VLAN Isolation**: Services can be isolated to specific VLANs if needed
- **SSH Access**: Restricted to management VLAN or VPN (Tailscale)
- **K8s API Server**: Exposed via LoadBalancer IP with authentication/authorization

## Monitoring & Troubleshooting

### Key Commands
```bash
# Check bond status
cat /proc/net/bonding/bond0

# View VLAN tags on bridge
brctl showvids vmbr0

# Check routing table
ip route show

# Monitor network interfaces
ip -s link
```

### Network Monitoring Tools
- **iftop**: Real-time bandwidth usage per connection
- **nethogs**: Per-process network usage
- **tcpdump**: Packet capture for troubleshooting
- **wireshark**: GUI packet analysis (remote access via X11 or web interface)

---

*Network documentation sanitized for public viewing. Specific IP addresses, VLAN IDs, and internal hostnames have been generalized. Actual values should be referenced from operational documentation.*
