# Mock Trading Mobile App (mock-trading.jdools.com)

## Overview

Mock Trading is a paper trading application that allows users to practice stock trading with virtual money. The app uses real-time stock quotes from Finnhub API and stores user data in Supabase (PostgreSQL). The web version is hosted on the homelab Kubernetes cluster via NGINX, while the mobile version uses Expo/React Native for cross-platform deployment.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Mock Trading Service                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Client Layer                                       │   │
│  │  - Mobile App (React Native / Expo)                 │   │
│  │  - Web App (Expo web build served by NGINX)         │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Backend Services (Supabase Cloud)                  │   │
│  │  - PostgreSQL Database                              │   │
│  │  - Authentication (email/password)                  │   │
│  │  - Edge Functions (Deno runtime)                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  External Services                                  │   │
│  │  - Finnhub API (real-time stock quotes)             │   │
│  │  - TradingView (charting widget)                    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Mobile Framework** | React Native (Expo SDK 56) | Cross-platform mobile app |
| **Web Build** | Expo web build + NGINX | Web version hosting |
| **Backend** | Supabase Cloud (PostgreSQL) | Database, auth, edge functions |
| **Stock Data** | Finnhub API | Real-time stock quotes |
| **Charting** | TradingView Widget | Interactive price charts |
| **Styling** | NativeWind (Tailwind CSS) | Utility-first styling |
| **State Management** | React Hooks + Supabase client | Client-side state |

## Key Features

### User Authentication
- Email/password sign-up and login
- Session persistence via AsyncStorage
- Auto-refresh tokens for seamless experience

### Trading Functionality
- **Real-time Quotes**: Fetch current stock prices from Finnhub API
- **Buy/Sell Orders**: Execute trades with virtual currency ($50,000 starting balance)
- **Portfolio Tracking**: View current holdings and transaction history
- **Whitelist System**: Admin-approved stock symbols for trading

### Stock Whitelist Management
- Pre-seeded with S&P 500 and Nasdaq-100 symbols (~500+ stocks)
- Users can request new symbols (admin approval required)
- Admins can approve/decline requests via dedicated screen

### Price Charts
- TradingView advanced chart widget embedded in mobile/web
- Support for symbol search and interval selection
- Dark theme matching app design

## Database Schema (Supabase PostgreSQL)

### Core Tables
| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `profiles` | User profiles | id, email, first_name, last_name, role, cash |
| `holdings` | Stock positions | user_id, symbol, amount |
| `transactions` | Trade history | user_id, symbol, amount, price, trade_type |
| `whitelist_stocks` | Approved symbols | symbol (unique) |
| `stock_requests` | Pending symbol requests | user_id, symbol, name, exchange |

### Row Level Security (RLS)
- Users can only view/modify their own data
- Admins have full access to all tables
- Whitelist management restricted to admins

## Edge Functions (Supabase Deno Runtime)

### finnhub-quote Function
**Purpose**: Fetch real-time stock quotes from Finnhub API

**Flow:**
1. Client calls `supabase.functions.invoke('finnhub-quote', { body: { symbol } })`
2. Edge function validates request and adds Finnhub API key
3. Fetches quote from `https://finnhub.io/api/v1/quote?symbol=<SYMBOL>`
4. Returns quote data (current price, change, percentage, etc.)

**Response Format:**
```json
{
  "symbol": "AAPL",
  "c": 178.50,      // Current price
  "d": 2.30,        // Change
  "dp": 1.31,       // Percentage change
  "h": 179.20,      // High
  "l": 176.80,      // Low
  "o": 177.50,      // Open
  "pc": 176.20      // Previous close
}
```

## Deployment Process (Web Version)

### Build Pipeline (`build-and-deploy.sh`)
1. **Environment Setup**: Load Supabase URL and anon key from `.env` or ConfigMap
2. **Docker Build**: Multi-stage build using `Dockerfile.build`:
   - Stage 1: Node.js 20 Alpine builds Expo web app
   - Stage 2: NGINX serves built assets
3. **Push to Docker Hub**: Tag with Git SHA and `latest`
4. **Apply K8s Manifests**: Update Deployment image, trigger rollout restart
5. **Wait for Rollout**: Monitor until all pods ready

### Dockerfile.build (Multi-stage)
```dockerfile
# Stage 1: Build web assets
FROM node:20-alpine AS builder
ARG EXPO_PUBLIC_SUPABASE_URL
ARG EXPO_PUBLIC_SUPABASE_ANON_KEY
RUN npx expo export --platform web --output-dir web-build

# Stage 2: Serve with NGINX
FROM nginx:stable-alpine
COPY k8s/nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/web-build/ /usr/share/nginx/html/
```

### Kubernetes Manifests
- **Deployment**: Single replica (can scale to 2+ for HA)
- **Service**: ClusterIP on port 80
- **Ingress**: Traefik routes `mock-trading.jdools.com` → service:80

## Mobile App Structure

### Directory Layout
```
mock-trading-mobile/
├── src/
│   ├── components/          # Reusable UI components
│   │   ├── ScreenHeader.js  # Navigation header with user info
│   │   └── TradingViewChart.native.js / .web.js
│   ├── lib/                 # Utilities and client initialization
│   │   ├── format.js        # Currency formatting
│   │   └── supabase.js      # Supabase client setup
│   └── screens/             # App screens
│       ├── AuthScreen.js    # Login/signup
│       ├── HomeScreen.js    # Main trading interface
│       ├── WhitelistScreen.js # Stock whitelist management
│       ├── InstructionsScreen.js # Onboarding/help
│       └── AdminScreen.js   # Admin panel (role-based)
├── k8s/                     # Kubernetes deployment files
│   ├── nginx/default.conf   # NGINX config for SPA routing
│   └── web-deployment.yaml  # Deployment, Service, Ingress
├── supabase/                # Supabase configuration and migrations
│   ├── schema.sql           # Database schema with RLS policies
│   ├── functions/finnhub-quote/ # Edge function source
│   └── seed/whitelist_seed.sql # Initial stock whitelist data
└── build-and-deploy.sh      # CI/CD script
```

### Key Screens
1. **AuthScreen**: Email/password authentication with Supabase
2. **HomeScreen**: Trading interface with portfolio, quote lookup, and transaction form
3. **WhitelistScreen**: Browse approved stocks and request new symbols
4. **InstructionsScreen**: User guide and app overview
5. **AdminScreen** (admin role only): Manage users, update cash balances, approve stock requests

## Security Considerations

- **API Keys**: Finnhub API key stored in Supabase edge function environment variables (not exposed to client)
- **Authentication**: Supabase handles session management with secure token storage
- **Row Level Security**: Database policies prevent unauthorized data access
- **CORS**: Supabase edge functions configured with appropriate CORS headers

## Monitoring & Debugging

### Logs
```bash
# View NGINX logs for web version
kubectl logs -f <mock-trading-pod-name>

# Check Supabase edge function logs (via Supabase dashboard)
# https://app.supabase.com/project/<project-id>/functions
```

### Health Checks
- **Readiness Probe**: HTTP GET `/health` returns 200 OK
- **Liveness Probe**: HTTP GET `/health` returns 200 OK

## Future Enhancements

1. **Real-time Updates**: WebSocket integration for live portfolio updates
2. **Advanced Charts**: Add technical indicators and drawing tools
3. **Paper Trading Leaderboard**: Compare portfolios with other users
4. **Mobile Push Notifications**: Alerts for price targets or trade confirmations
5. **Historical Data**: Backtest trading strategies with historical quotes

---

*Service documentation. Specific API endpoints, database credentials, and internal configurations have been generalized for public viewing.*
