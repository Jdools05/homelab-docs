# Power Playlist Service (power-playlist.jdools.com)

## Overview

A Spotify integration service that lets users create and manage music playlists. Consists of a Node.js API backend and a Next.js web frontend, both hosted on Kubernetes behind Traefik ingress with path-based routing.

## Architecture

```
User → power-playlist.jdools.com
    ├── /api/* → power-playlist-api:3000 (Node.js REST API)
    └── /*     → power-playlist-web:80  (Next.js static export via NGINX)
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Frontend** | Next.js (React framework) | Web application with SSR/SSG |
| **Backend API** | Node.js (Express/Fastify) | REST API for Spotify integration |
| **Hosting** | Kubernetes (k3s) | Container orchestration |
| **Web Server** | NGINX (for Next.js static export) or custom | Static file serving / reverse proxy |
| **Authentication** | Spotify OAuth 2.0 | User authentication via Spotify accounts |
| **Database** | PostgreSQL (self-hosted) | Store user playlists and preferences |

## Spotify Integration

Users authenticate via Spotify OAuth 2.0 — click "Login with Spotify," grant playlist read/write permissions, and the backend exchanges the authorization code for access/refresh tokens stored securely in the database. Typical scopes: `playlist-read-private`, `playlist-modify-public`, `playlist-modify-private`, `user-library-read`, `user-library-modify`.

## Kubernetes Resources

### power-playlist-api (Node.js Backend)
- **Image**: `jdools05/power-playlist-api:latest`
- **Port**: 3000/TCP
- **Resources**: 100m CPU / 256Mi RAM requests, 500m CPU / 512Mi RAM limits
- **Ingress path**: `/api/*` (priority 200 — higher than web for path matching)

### power-playlist-web (Next.js Frontend)
- **Image**: `jdools05/power-playlist-web:latest`
- **Port**: 80/TCP
- **Resources**: 50m CPU / 128Mi RAM requests, 200m CPU / 256Mi RAM limits
- **Ingress path**: `/` (priority 100)

## Database Schema

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `users` | Spotify user profiles | spotify_id, username, access_token (encrypted), refresh_token (encrypted) |
| `playlists` | User playlists | id, user_id, spotify_playlist_id, name, description, public |
| `playlist_tracks` | Tracks in playlists | playlist_id, position, track_uri, added_at |
