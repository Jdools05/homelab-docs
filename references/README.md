# References Documentation

This directory contains reference documentation for the homelab infrastructure, including DNS configuration, SSL/TLS certificate management, security overview, and change history. These documents serve as operational guides and compliance references.

## Contents

- **[dns-records.md](./dns-records.md)** - DNS configuration for all domains and subdomains (external registrar + internal Kubernetes CoreDNS)
- **[ssl-certificates.md](./ssl-certificates.md)** - TLS/SSL certificate management, renewal procedures, and security best practices
- **[security.md](./security.md)** - Security measures including network segmentation, access control, data protection, and incident response
- **[changelog.md](./changelog.md)** - History of significant infrastructure changes and maintenance activities

## Purpose

These reference documents provide:
1. **Operational Guidance**: Step-by-step procedures for common tasks (DNS updates, certificate renewal, security audits)
2. **Compliance Reference**: Alignment with security standards (CIS Benchmarks, NIST CSF, OWASP Top 10)
3. **Knowledge Sharing**: Centralized location for infrastructure knowledge accessible to all team members
4. **Portfolio Showcase**: Demonstrates infrastructure expertise and attention to security best practices

## Security Notice

All documentation in this repository has been sanitized for public viewing. Specific network configurations, IP addresses, internal policies, and sensitive operational details have been generalized or omitted. The `_raw-data/` directory contains unsanitized operational data for reference only.
