# Power Playlist Service (power-playlist.jdools.com)

## Overview

Power Playlist is a Spotify integration service that allows users to create and manage music playlists. The service consists of a Node.js API backend and a Next.js web frontend, both hosted on the homelab Kubernetes cluster behind Traefik ingress controller.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                Power Playlist Service                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Client Layer                                       │   │
│  │  - Next.js Web App (power-playlist-web)             │   │
│  │    Served by: NGINX pod in Kubernetes               │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  API Layer                                          │   │
│  │  - Node.js API (power-playlist-api)                 │   │
│  │    Served by: Custom container in Kubernetes        │   │
│  │  Port: 3000/TCP                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  External Services                                  │   │
│  │  - Spotify API (OAuth2 authentication)              │   │
│  │  - Database (PostgreSQL/Supabase or similar)        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

Traffic Flow:
User → power-playlist.jdools.com
  ├── /api/* → power-playlist-api:3000 (API requests)
  └── /*     → power-playlist-web:80 (Web frontend)
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Frontend** | Next.js (React framework) | Web application with SSR/SSG |
| **Backend API** | Node.js (Express/Fastify) | REST API for Spotify integration |
| **Hosting** | Kubernetes (k3s) | Container orchestration |
| **Web Server** | NGINX (for Next.js static export) or custom | Static file serving / reverse proxy |
| **Authentication** | Spotify OAuth 2.0 | User authentication via Spotify accounts |
| **Database** | PostgreSQL (Supabase or self-hosted) | Store user playlists and preferences |
| **Ingress** | Traefik v3.6 | External routing with path-based rules |

## Spotify Integration

### OAuth 2.0 Flow
1. User clicks "Login with Spotify" on web app
2. Redirected to Spotify authorization page
3. User grants permissions (playlist read/write access)
4. Spotify redirects back with authorization code
5. Backend exchanges code for access/refresh tokens
6. Tokens stored securely (encrypted in database or secure cookie)

### Configuration (from Kubernetes ConfigMap)
```yaml
SPOTIFY_REDIRECT_URI: https://power-playlist.jdools.com/api/auth/spotify/callback
VITE_API_BASE_URL: https://power-playlist.jdools.com/api
VITE_FRONTEND_BASE_URL: https://power-playlist.jdools.com
VITE_IS_PRODUCTION: "true"
```

### Spotify API Scopes (Typical)
- `playlist-read-private` - Read user's playlists
- `playlist-modify-public` - Create/edit public playlists
- `playlist-modify-private` - Create/edit private playlists
- `user-library-read` - Access user's saved songs
- `user-library-modify` - Save/remove songs from library

## API Endpoints (Typical Structure)

| Method | Path | Description | Authentication |
|--------|------|-------------|----------------|
| `GET` | `/api/auth/spotify/callback` | OAuth callback handler | None |
| `POST` | `/api/playlists` | Create new playlist | Spotify token |
| `GET` | `/api/playlists` | List user's playlists | Spotify token |
| `GET` | `/api/playlists/:id` | Get playlist details | Spotify token |
| `PUT` | `/api/playlists/:id` | Update playlist metadata | Spotify token |
| `DELETE` | `/api/playlists/:id` | Delete playlist | Spotify token |
| `POST` | `/api/playlists/:id/tracks` | Add tracks to playlist | Spotify token |
| `GET` | `/api/me` | Get current user profile | Spotify token |

## Deployment Architecture

### Kubernetes Resources

#### power-playlist-api (Node.js Backend)
```yaml
# Deployment
replicas: 1
image: jdools05/power-playlist-api:latest
port: 3000/TCP
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Service (ClusterIP)
port: 3000
targetPort: 3000

# Ingress (path: /api)
host: power-playlist.jdools.com
priority: 200 (higher than web for path matching)
```

#### power-playlist-web (Next.js Frontend)
```yaml
# Deployment
replicas: 1
image: jdools05/power-playlist-web:latest
port: 80/TCP
resources:
  requests:
    cpu: 50m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

# Service (ClusterIP)
port: 80
targetPort: 80

# Ingress (path: /)
host: power-playlist.jdools.com
priority: 100
```

### Traefik Routing Rules
- `/api/*` → `power-playlist-api:3000` (API requests)
- `/*` → `power-playlist-web:80` (Web frontend)

## Database Schema (Typical for Playlist Service)

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `users` | Spotify user profiles | spotify_id, username, access_token (encrypted), refresh_token (encrypted) |
| `playlists` | User playlists | id, user_id, spotify_playlist_id, name, description, public |
| `playlist_tracks` | Tracks in playlists | playlist_id, position, track_uri, added_at |

## Development & Deployment

### Local Development Setup
1. Clone repository
2. Install dependencies: `npm install` (both api and web directories)
3. Configure environment variables (Spotify client ID/secret, database URL)
4. Run API: `cd api && npm run dev`
5. Run Web: `cd web && npm run dev`

### Production Build Process
1. **API**: Docker build → push to Docker Hub (`jdools05/power-playlist-api`)
2. **Web**: Next.js export (static HTML) → Docker build with NGINX → push to Docker Hub (`jdools05/power-playlist-web`)
3. **Deploy**: Apply Kubernetes manifests or use CI/CD pipeline

### Environment Variables Required
- `SPOTIFY_CLIENT_ID` - Spotify app client ID
- `SPOTIFY_CLIENT_SECRET` - Spotify app client secret
- `DATABASE_URL` - PostgreSQL connection string
- `JWT_SECRET` (if using JWT for internal auth)
- `NODE_ENV=production`

## Monitoring & Logging

### API Logs
```bash
kubectl logs -f deployment/power-playlist-api -n default
```

### Web Access Logs
```bash
kubectl logs -f deployment/power-playlist-web -n default
```

### Health Checks
- **API**: HTTP GET `/health` or `/api/health` returns 200 OK
- **Web**: HTTP GET `/` returns 200 OK (Next.js static export)

## Security Considerations

- **Spotify Tokens**: Stored encrypted in database, never logged
- **CORS**: Configured to allow only `https://power-playlist.jdools.com`
- **Rate Limiting**: Not currently configured (recommend adding via Traefik middleware)
- **HTTPS**: Required by Spotify OAuth (redirect URI must be HTTPS)
- **Secrets Management**: Spotify credentials stored in Kubernetes Secrets

## Future Enhancements

1. **Mobile App**: React Native version for iOS/Android
2. **Collaborative Playlists**: Allow multiple users to edit same playlist
3. **Music Recommendation**: AI-powered suggestions based on listening history
4. **Analytics Dashboard**: Track playlist creation and song popularity
5. **Social Features**: Share playlists, follow other users

---

*Service documentation. Specific API endpoints, database credentials, and internal configurations have been generalized for public viewing.*
