# StockSense AI — Complete Technical Documentation

> Predictive Portfolio Intelligence Platform for Indian Equity Markets (NSE/BSE)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      StockSense AI Platform                      │
├───────────────┬───────────────────┬────────────────────────────┤
│   Data Layer  │  Intelligence     │  Delivery Layer             │
│               │  Engine           │                             │
│  Kite API ──► │  TA Indicators ──►│  Pre-Market Email (08:00)   │
│  NSE Feed  ──►│  NLP Sentiment ──►│  SMS via Twilio             │
│  News API  ──►│  Signal Engine ──►│  Push via Firebase          │
│  Fundamentals │  Claude AI     ──►│  React Dashboard            │
│  Bulk Deals   │  Risk Filter      │  AI Advisor Chat            │
└───────────────┴───────────────────┴────────────────────────────┘
```

## Tech Stack

| Layer        | Technology                          |
|--------------|-------------------------------------|
| Backend      | Node.js 20 + Express 5              |
| Database     | PostgreSQL 16 + pg pool             |
| Scheduler    | node-cron (IST timezone)            |
| ML/Indicators| Custom pure-JS engine (25 indicators)|
| AI/NLP       | Anthropic Claude (claude-sonnet-4)  |
| Market Data  | Zerodha Kite API → NSE fallback     |
| News         | NewsAPI + NSE Announcements         |
| Email        | SendGrid via Nodemailer             |
| SMS          | Twilio                              |
| Push         | Firebase Cloud Messaging            |
| Frontend     | React 19 + Tailwind CSS             |
| Hosting      | Railway / Render / AWS ECS          |

---

## File Structure

```
stocksense/
├── backend/
│   ├── src/
│   │   ├── index.js                    # Express server entry
│   │   ├── api/
│   │   │   ├── portfolio.js            # Portfolio CRUD routes
│   │   │   ├── alerts.js               # Alert history + config
│   │   │   ├── predictions.js          # Signal + prediction routes
│   │   │   └── advisor.js              # Claude AI advisor proxy
│   │   ├── db/
│   │   │   ├── pool.js                 # PostgreSQL connection pool
│   │   │   └── schema.sql              # Full DB schema (11 tables)
│   │   ├── ml/
│   │   │   ├── indicatorEngine.js      # All 25 technical indicators
│   │   │   ├── technicalAnalysis.js    # TA scoring + persistence
│   │   │   └── signalEngine.js         # Multi-factor signal generation
│   │   ├── services/
│   │   │   ├── marketData.js           # Kite API + NSE data fetcher
│   │   │   ├── sentimentService.js     # NLP sentiment via Claude
│   │   │   ├── fundamentalsService.js  # Balance sheet, P/E, ROE
│   │   │   ├── alertService.js         # Email + SMS + Push dispatch
│   │   │   ├── trustService.js         # Accuracy tracking + risk profiles
│   │   │   └── predictionService.js    # ML prediction persistence
│   │   ├── scheduler/
│   │   │   └── cron.js                 # All cron jobs (IST timezone)
│   │   └── utils/
│   │       ├── logger.js               # Winston structured logging
│   │       └── errorHandler.js         # Global error middleware
│   ├── package.json
│   └── .env.example
└── frontend/
    └── src/
        ├── components/
        │   ├── Portfolio.jsx
        │   ├── Recommendations.jsx
        │   ├── TrustDashboard.jsx
        │   ├── IndicatorSuite.jsx
        │   └── AIAdvisor.jsx
        └── App.jsx
```

---

## Setup & Installation

### 1. Clone and install

```bash
git clone https://github.com/yourorg/stocksense-ai
cd stocksense/backend
npm install
```

### 2. Environment variables

```bash
cp .env.example .env
```

```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/stocksense

# Zerodha Kite
KITE_API_KEY=your_kite_api_key
KITE_ACCESS_TOKEN=your_daily_access_token   # Refresh daily via OAuth

# Anthropic Claude
ANTHROPIC_API_KEY=sk-ant-...

# News
NEWS_API_KEY=your_newsapi_key

# Notifications
SENDGRID_API_KEY=SG.xxx
TWILIO_SID=ACxxx
TWILIO_AUTH_TOKEN=xxx
TWILIO_FROM_NUMBER=+1234567890

# Firebase
GOOGLE_APPLICATION_CREDENTIALS=./firebase-service-account.json

# App
PORT=4000
FRONTEND_URL=http://localhost:3000
NODE_ENV=production
```

### 3. Database setup

```bash
psql -U postgres -c "CREATE DATABASE stocksense;"
psql -U postgres -d stocksense -f src/db/schema.sql
```

### 4. Run

```bash
# Development
npm run dev          # nodemon with hot-reload

# Production
npm start            # node src/index.js

# Docker
docker-compose up -d
```

---

## Cron Job Schedule (IST)

| Time  | Job                          | Description                                    |
|-------|------------------------------|------------------------------------------------|
| 07:00 | `pre_market_signals`         | Full ML pipeline — all 25 indicators + Claude  |
| 08:00 | `alert_dispatch`             | Send email/SMS/push to all users               |
| 09:15–15:30 | `live_quote_refresh` | Market data every 60 seconds                   |
| 18:00 | `eod_accuracy_update`        | Evaluate expired predictions, update win rate  |
| 23:00 | `bulk_deal_fetch`            | NSE bulk/block deal ingestion                  |

---

## Technical Indicators Suite (25)

### Phase 1 (MVP — Launched)
| Category   | Indicators                                      |
|------------|-------------------------------------------------|
| Momentum   | RSI(14), MACD(12,26,9)                         |
| Trend      | SMA(20), SMA(50), EMA(9)                       |
| Volume     | OBV                                             |
| Volatility | Bollinger Bands(20,2)                           |

### Phase 2 (Current)
| Category   | Indicators Added                                |
|------------|-------------------------------------------------|
| Momentum   | + Stochastic(%K/%D), ROC(10), CCI(20)          |
| Trend      | + Supertrend(10,3), Parabolic SAR, ADX(14)     |
| Volume     | + VWAP, A/D Line, CMF(20), MFI(14)            |
| Volatility | + ATR(14), Keltner Channels, Donchian(20)      |
| Market     | + Put/Call Ratio, VIX India, Fear & Greed      |

### Phase 3 (Roadmap)
- Ichimoku Cloud (currently basic implementation → full Kumo)
- Volume Profile (VPOC, VAH, VAL)
- Elliott Wave detection (ML-assisted)
- Options chain analysis (OI, IV skew)

---

## Signal Generation Pipeline

```
1. fetchLiveQuotes()          → market_quotes table
2. fetchBulkDeals()           → bulk_deals table
3. computeAllIndicators()     → 25 indicators → TA score
4. runSentimentPipeline()     → NLP via Claude → sentiment_scores
5. getFundamentals()          → P/E, ROE, revenue growth
6. buildSignal()              → weighted ensemble score
7. generateReasoning()        → Claude writes explanation
8. persistSignal()            → signals table
9. dispatchAlerts()           → email + SMS + push
```

### Scoring Weights (Moderate Profile)

| Factor        | Weight |
|---------------|--------|
| Fundamental   | 30%    |
| Technical     | 30%    |
| Sentiment NLP | 25%    |
| Volume/Deals  | 15%    |

---

## Risk Profiles

| Profile      | Min Conf | Max Positions | Stop-Loss | Styles Allowed              |
|--------------|----------|---------------|-----------|------------------------------|
| Conservative | 75%      | 2             | 4%        | Long-term only               |
| Moderate     | 65%      | 5             | 6%        | Long-term + Swing            |
| Aggressive   | 55%      | 10            | 9%        | All (incl. short-term)       |

---

## Trust & Compliance Architecture

### What we are
- **Analytics + Insights Platform** (not SEBI-registered IA)
- Predictive scoring tool — users make their own investment decisions

### Legal safeguards
- Every signal page shows SEBI disclaimer
- Email footer includes regulatory notice
- "Insights, not advice" framing throughout UI
- Future: Apply for SEBI Research Analyst (RA) registration for full advisory

### Data security
- AES-256 encryption at rest (PostgreSQL TDE)
- TLS 1.3 for all API traffic
- Zero third-party portfolio data sharing
- SOC2 Type II target (Year 2)

---

## Revenue Model

| Tier     | Price      | Features                                            |
|----------|------------|-----------------------------------------------------|
| Free     | ₹0/month   | 5 stocks, basic signals, email-only                 |
| Pro      | ₹499/month | 20 stocks, all indicators, SMS + push, AI Advisor   |
| Elite    | ₹1499/month| Unlimited stocks, priority signals, API access      |
| Broker   | Revenue share | White-label + referral commission model          |

---

## Roadmap

### Q3 2025 — v1.0 (MVP)
- [x] Portfolio management (20 stocks)
- [x] NSE/BSE live data via Kite
- [x] Phase 1 indicators (RSI, MACD, SMA, BB, OBV)
- [x] Claude-powered signal reasoning
- [x] Pre-market email alerts

### Q4 2025 — v2.0
- [x] All 25 indicators
- [x] Sentiment NLP pipeline
- [x] Risk profiles (conservative/moderate/aggressive)
- [x] SMS + push notifications
- [x] Trust dashboard + accuracy tracking
- [x] AI Advisor chat

### Q1 2026 — v3.0
- [ ] SEBI RA registration
- [ ] Options chain analysis
- [ ] Broker integrations (Zerodha, Groww, Angel)
- [ ] Mobile app (React Native)
- [ ] Institutional data insights product

---

## API Reference

```
GET    /api/portfolio/:userId           → Holdings + current signals
GET    /api/signals/:symbol             → Signal detail + indicators
GET    /api/predictions/:symbol         → ML predictions + probability
GET    /api/alerts/:userId              → Alert history
POST   /api/advisor/chat                → AI Advisor (Claude proxy)
GET    /api/trust/accuracy              → Win rate + performance stats
PUT    /api/users/:userId/risk-profile  → Update risk appetite
GET    /api/indicators/:symbol          → All 25 indicators for symbol
POST   /api/alerts/:alertId/acted-on   → Mark alert as acted on
```

---

*StockSense AI — Predictive Portfolio Intelligence. Built for the Indian retail investor.*
*© 2025 StockSense Technologies. SEBI Disclaimer: Not a registered investment advisor.*
