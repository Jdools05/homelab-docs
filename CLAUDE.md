# CLAUDE.md - Homelab Documentation Repository

## Purpose
This repository documents the entire homelab infrastructure including hardware, networking, Kubernetes cluster, services, and websites. It serves as both operational documentation and a portfolio piece for future employers.

## Security Guidelines
- **NEVER** include real IP addresses in public-facing docs (use placeholders like `192.168.x.x`)
- **NEVER** include API keys, passwords, tokens, or SSH private keys
- **NEVER** include internal hostnames with `.ts.net` domains (use generic names)
- **NEVER** include personal information (real names, emails, contact details)
- Architecture descriptions and technical decisions are fine to share
- Code samples (HTML, YAML manifests, app code) are non-sensitive

## Repository Structure
```
homelab-docs/
├── README.md                    # Overview and quick reference
├── CLAUDE.md                    # This file
├── .gitignore
│
├── _raw-data/                   # Raw collected data (not for public sharing)
│   ├── proxmox/                 # Proxmox host data
│   └── kubernetes/              # K8s cluster data
│
├── infrastructure/              # Infrastructure documentation
│   ├── README.md
│   ├── hardware.md              # Hardware specs (sanitized)
│   ├── network.md               # Network architecture (sanitized)
│   ├── proxmox-configuration.md # VMs, containers, storage pools
│   └── backup-restore.md        # Backup strategy
│
├── kubernetes/                  # Kubernetes documentation
│   ├── README.md
│   ├── architecture.md          # Cluster topology and design
│   ├── ingress.md               # Traefik setup and routing
│   ├── namespaces.md            # Namespace layout
│   ├── persistent-storage.md    # PVCs, storage classes
│   └── manifests/               # Current K8s resource YAMLs
│
├── services/                    # Service documentation
│   ├── README.md
│   ├── jdools-landing.md        # Landing page architecture
│   ├── mock-trading-mobile.md   # Mobile trading app
│   ├── power-playlist.md        # Spotify integration service
│   └── other-services.md        # Other services (404, etc.)
│
├── websites/                    # Website assets and docs
│   ├── README.md
│   ├── jdools.com/              # Landing page assets
│   │   ├── index.html
│   │   ├── sitemap.xml
│   │   ├── robots.txt
│   │   └── DEPLOY.md
│   └── mock-trading.jdools.com/ # Mock trading web build reference
│       └── README.md
│
└── references/                  # Reference documentation
    ├── dns-records.md           # DNS configuration (sanitized)
    ├── ssl-certificates.md      # TLS cert management
    ├── security.md              # Security overview (sanitized)
    └── changelog.md             # Infrastructure change history
```

## Maintenance
When updating documentation:
1. Check for sensitive data before committing
2. Update the changelog in `references/changelog.md`
3. Keep raw data in `_raw-data/` - don't commit to public repo
4. Use placeholders for IPs, hostnames, and secrets
