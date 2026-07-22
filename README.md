# Homelab Infrastructure Documentation

Welcome to the homelab infrastructure documentation repository. This project documents a self-hosted homelab environment running on Proxmox VE with a Kubernetes (k3s) cluster for containerized services.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Homelab Network                          │
│                                                             │
│  ┌──────────────────┐        ┌──────────────────────────┐   │
│  │   Internet/ISP   │        │   Internal Networks      │   │
│  │                  │        │                          │   │
│  │   WAN (DHCP)     ├───────►│  Management Network      │   │
│  │                  │  LAG   │  Kubernetes Cluster      │   │
│  └──────────────────┘        │  Service Networks        │   │
│                              └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Proxmox VE Host                          │
│                                                             │
│  CPU: Intel Xeon E5-2667 v3 (Dual Socket, 16 cores)         │
│  RAM: 128 GB DDR4                                           │
│  Storage: 30TB LVM Thin Pool                                │
│  Network: 4x 1GbE Bonded (LACP)                             │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ OPNsense    │  │ k8s Ctrl-1  │  │ k8s Worker-1│          │
│  │ (Firewall)  │  │ (Control)   │  │             │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│  ┌─────────────┐  ┌─────────────┐                           │
│  │ k8s Worker-2│  │ Nextcloud   │                           │
│  │             │  │ (Cloud)     │                           │
│  └─────────────┘  └─────────────┘                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster (k3s v1.34.5)               │
│                                                             │
│  Node Count: 3 (1 control + 2 workers)                      │
│  Ingress: Traefik with MetalLB                              │
│  Storage: Longhorn (distributed block storage)              │
│  Monitoring: Prometheus Stack                               │
│                                                             │
│  Services Exposed via Ingress:                              │
│  ├── jdools.com              → Landing Page                 │
│  ├── mock-trading.jdools.com → React Native App (Web)       │
│  ├── power-playlist.jdools.com → Spotify Integration        │
│  └── aoa-marching-cubes.jdools.com → Java 3D Visualization  │  
└─────────────────────────────────────────────────────────────┘
```

## Quick Reference

| Component | Technology | Status |
|-----------|------------|--------|
| Hypervisor | Proxmox VE 8.0 | Production |
| Kubernetes | k3s v1.34.5 | Production |
| Ingress Controller | Traefik v3.6 | Production |
| Load Balancer | MetalLB | Production |
| Storage | Longhorn v1.7 | Production |
| Monitoring | Prometheus Stack | Production |
| Firewall | OPNsense VM | Production |

## Services

| Service | Domain | Tech Stack | Description |
|---------|--------|------------|-------------|
| **Landing Page** | jdools.com | Static HTML/NGINX | Portfolio site with aurora UI |
| **Mock Trading** | mock-trading.jdools.com | React Native, Supabase, Finnhub API | Paper trading app with real-time quotes |
| **Power Playlist** | power-playlist.jdools.com | Node.js API + Next.js Web | Spotify playlist management |
| **AOA Marching Cubes** | aoa-marching-cubes.jdools.com | Java 3D Visualization | 3D surface visualization tool |

## Documentation Structure

- **[infrastructure/](./infrastructure/)** - Hardware, networking, and Proxmox configuration
- **[kubernetes/](./kubernetes/)** - Kubernetes cluster architecture and manifests
- **[services/](./services/)** - Detailed service documentation
- **[websites/](./websites/)** - Website assets and deployment docs
- **[references/](./references/)** - DNS, SSL, security, and changelog

## TODO

- [ ] **VLAN segmentation** — Document and formalize VLAN layout (management, k8s, services) in the network docs
- [ ] **iSCSI integration** — Get iSCSI working for shared block storage across Proxmox nodes

---

*Last updated: 2026-07-21*
