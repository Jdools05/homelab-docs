# Services Documentation

This directory contains detailed documentation for each service running on the homelab infrastructure. Each service document covers architecture, technology stack, deployment process, and operational details.

## Contents

- **[jdools-landing.md](./jdools-landing.md)** - Landing page (jdools.com) - Static HTML/NGINX
- **[mock-trading-mobile.md](./mock-trading-mobile.md)** - Mock Trading app - React Native + Supabase + Finnhub API
- **[power-playlist.md](./power-playlist.md)** - Power Playlist service - Spotify integration with Node.js API and Next.js frontend
- **[other-services.md](./other-services.md)** - Additional services (dynamic 404 handler, etc.)

## Service Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Services Layer                           │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐               │
│  │ Landing Page    │    │ Mock Trading    │               │
│  │ (Static HTML)   │    │ (React Native)  │               │
│  │                 │    │                 │               │
│  │ NGINX           │    │ NGINX + Supabase│               │
│  └─────────────────┘    └─────────────────┘               │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐               │
│  │ Power Playlist  │    │ AOA Marching    │               │
│  │ (Node.js API)   │    │ Cubes           │               │
│  │ + Next.js Web   │    │ (Java 3D Viz)   │               │
│  └─────────────────┘    └─────────────────┘               │
└─────────────────────────────────────────────────────────────┘

All services exposed via Traefik ingress controller on jdools.com domains.
