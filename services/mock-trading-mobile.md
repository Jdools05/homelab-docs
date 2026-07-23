# Mock Trading Mobile App (mock-trading.jdools.com)

## Overview

A paper trading application that lets users practice stock trading with virtual money. Uses real-time quotes from the Finnhub API and stores user data in Supabase Cloud (PostgreSQL). Available as a React Native mobile app (Expo SDK 56) and a web build hosted on Kubernetes.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Client Layer                                       │
│  ├── Mobile App (React Native / Expo)               │
│  └── Web App (Expo web build served by NGINX)       │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  Backend: Supabase Cloud                            │
│  ├── PostgreSQL Database                            │
│  ├── Authentication (email/password)                │
│  └── Edge Functions (Deno runtime, Finnhub API key) │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  External: Finnhub API (stock quotes) + TradingView │
└─────────────────────────────────────────────────────┘
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Mobile Framework** | React Native (Expo SDK 56) | Cross-platform mobile app |
| **Web Build** | Expo web build + NGINX | Web version hosting on K8s |
| **Backend** | Supabase Cloud (PostgreSQL) | Database, auth, edge functions |
| **Stock Data** | Finnhub API | Real-time stock quotes |
| **Charting** | TradingView Widget | Interactive price charts |
| **Styling** | NativeWind (Tailwind CSS) | Utility-first styling |

## Key Features

- **Authentication**: Email/password via Supabase with session persistence
- **Trading**: Real-time quotes from Finnhub, buy/sell orders with $50k starting balance, portfolio tracking
- **Stock Whitelist**: Pre-seeded S&P 500 + Nasdaq-100 (~500+ stocks), admin approval for new symbols
- **Charts**: TradingView advanced chart widget embedded in mobile/web

## Database Schema (Supabase PostgreSQL)

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `profiles` | User profiles | id, email, first_name, last_name, role, cash |
| `holdings` | Stock positions | user_id, symbol, amount |
| `transactions` | Trade history | user_id, symbol, amount, price, trade_type |
| `whitelist_stocks` | Approved symbols | symbol (unique) |
| `stock_requests` | Pending symbol requests | user_id, symbol, name, exchange |
