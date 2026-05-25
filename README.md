# Axora — Smart Finance, Safe Future

> AI-driven personal finance management and real-time fraud detection dashboard, built for the modern Indian user.

---

## Project Overview

**Axora** is a full-stack web application that acts as a personal financial guardian. It combines:

- **Finance History Tracking** — complete transaction ledger with categories, statuses, and AI fraud scores
- **AI Fraud Detection** — rule-based + anomaly detection engine that scores every transaction and scans incoming messages, links, images, audio, video, and screenshots for threats
- **Savings Goals** — goal-tracking with AI-projected timelines and progress rings
- **Spending Analytics** — category breakdown, 30-day trend with anomaly highlights, weekend vs. weekday spend comparison
- **Finance Meter** — live donut gauge showing savings vs. expenses percentage with a 0–100 financial health score
- **Threat Scanner** — 5-mode deep scanner (text, image, audio, video, screenshot) that analyses suspicious content before the user opens it

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, TypeScript, Vite 7, Wouter (routing) |
| Styling | Tailwind CSS v4, custom glassmorphism utilities |
| Charts | Recharts |
| State / API | TanStack React Query v5, Orval (OpenAPI codegen) |
| Backend | Node.js 24, Express 5, TypeScript |
| Database | PostgreSQL (Neon/Replit managed) |
| ORM | Drizzle ORM + drizzle-zod |
| Validation | Zod v4 |
| API Contract | OpenAPI 3.1 (contract-first) |
| Package Manager | pnpm workspaces (monorepo) |
| Build | esbuild (server), Vite (client) |

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                    AXORA MONOREPO                        │
│                                                         │
│  artifacts/axora/         ← React + Vite frontend       │
│  artifacts/api-server/    ← Express 5 API server        │
│  lib/api-spec/            ← OpenAPI 3.1 contract        │
│  lib/api-client-react/    ← Generated React Query hooks │
│  lib/api-zod/             ← Generated Zod validators    │
│  lib/db/                  ← Drizzle ORM schema + client │
└─────────────────────────────────────────────────────────┘
```

### System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                        USER BROWSER                                   │
│                                                                       │
│   ┌───────────────────────────────────────────────────────────────┐  │
│   │                    REACT FRONTEND (Vite)                       │  │
│   │                                                               │  │
│   │  Dashboard  Transactions  Fraud Shield  Goals  Analytics      │  │
│   │       │           │            │           │        │         │  │
│   │       └───────────┴────────────┴───────────┴────────┘         │  │
│   │                            │                                   │  │
│   │              TanStack React Query (hooks)                      │  │
│   │           @workspace/api-client-react (Orval gen.)            │  │
│   └───────────────────────────┬───────────────────────────────────┘  │
└───────────────────────────────┼──────────────────────────────────────┘
                                │ HTTP / REST (via Replit shared proxy)
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    EXPRESS 5 API SERVER (:8080)                       │
│                                                                       │
│  /api/transactions   ──► Transaction Handler                          │
│  /api/fraud/*        ──► Fraud Detection Engine ──► analyzeMessage()  │
│  /api/goals          ──► Goal Manager                                 │
│  /api/analytics/*    ──► Analytics Aggregator                         │
│  /api/healthz        ──► Health Check                                 │
│                                                                       │
│  Middleware: pino-http (logging) · cors · express.json()             │
│  Validation: @workspace/api-zod (Zod schemas, Orval gen.)            │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    POSTGRESQL DATABASE                                 │
│                                                                       │
│   transactions             fraud_alerts             goals             │
│   ─────────────            ────────────             ─────             │
│   id (PK)                  id (PK)                  id (PK)           │
│   date                     type                     name              │
│   merchant                 content                  target_amount     │
│   category                 risk_score               current_amount    │
│   amount                   risk_level               deadline          │
│   currency                 reason                   category          │
│   status                   created_at               color             │
│   fraud_score              dismissed                created_at        │
│   ai_flag                  sender                                     │
│   location                                                            │
│   manual_flag                                                         │
│   notes                                                               │
└──────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Action
    │
    ▼
React Component
    │  useQuery / useMutation (React Query)
    ▼
Orval-generated API hook (@workspace/api-client-react)
    │  fetch() to /api/*
    ▼
Express Route Handler
    │  Zod schema validation (@workspace/api-zod)
    ├──► Fraud Engine (lib/fraud.ts) — regex pattern scoring
    ├──► Drizzle ORM query
    ▼
PostgreSQL
    │  Result
    ▼
Zod .parse() — shape guarantee
    │
    ▼
JSON response → React Query cache → UI re-render
```

---

## Database Schema

### `transactions`
| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | Auto-increment |
| date | timestamp | Transaction date/time |
| merchant | text | Merchant name |
| category | text | food/transport/bills/shopping/entertainment/health/travel/other |
| amount | numeric(12,2) | Transaction amount in INR |
| currency | text | Default: INR |
| status | text | pending/cleared/flagged/blocked |
| fraud_score | numeric(5,2) | 0–100 AI risk score |
| ai_flag | text | normal/suspicious/high_risk |
| location | text | Country/city of transaction |
| manual_flag | text nullable | fraudulent/safe (user override) |
| notes | text nullable | Optional notes |

### `fraud_alerts`
| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | Auto-increment |
| type | text | message/link/transaction/call |
| content | text | Raw content of the threat |
| risk_score | numeric(5,2) | 0–100 threat score |
| risk_level | text | low/medium/high/critical |
| reason | text | Semicolon-separated detection reasons |
| created_at | timestamp | When intercepted |
| dismissed | boolean | Whether user has seen/dismissed |
| sender | text nullable | Sender phone/email if known |

### `goals`
| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | Auto-increment |
| name | text | Goal name |
| target_amount | numeric(12,2) | Target amount in INR |
| current_amount | numeric(12,2) | Amount saved so far |
| deadline | timestamp | Target completion date |
| category | text | emergency/travel/home/car/education/retirement/other |
| color | text | Hex color for the goal card |
| created_at | timestamp | When goal was created |

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/healthz | Health check |
| GET | /api/transactions | List all transactions (filterable by category, period) |
| POST | /api/transactions | Add a new transaction (auto-scored by AI) |
| PATCH | /api/transactions/:id | Update a transaction |
| DELETE | /api/transactions/:id | Delete a transaction |
| PATCH | /api/transactions/:id/flag | Manually flag as fraudulent/safe |
| GET | /api/fraud/alerts | List fraud alerts |
| POST | /api/fraud/alerts | Create a new fraud alert (AI-analysed) |
| PATCH | /api/fraud/alerts/:id/dismiss | Dismiss an alert |
| POST | /api/fraud/simulate | Run threat scanner on content |
| GET | /api/goals | List savings goals |
| POST | /api/goals | Create a savings goal |
| PATCH | /api/goals/:id | Update a goal |
| DELETE | /api/goals/:id | Delete a goal |
| GET | /api/analytics/summary | Dashboard summary (balance, income, expenses) |
| GET | /api/analytics/spending | Category spending breakdown |
| GET | /api/analytics/trend | 30-day daily spend trend |
| GET | /api/analytics/meter | Finance meter / health score |
| GET | /api/analytics/weekend | Weekend vs weekday spending |
| GET | /api/analytics/ai-insight | AI-generated spending insight |

---

## Setup Instructions

### Prerequisites
- Node.js 20+ (Node 24 recommended)
- pnpm 9+
- PostgreSQL database (or use Replit's built-in DB)

### 1. Clone the repository
```bash
git clone <repo-url>
cd axora
```

### 2. Install dependencies
```bash
pnpm install
```

### 3. Set environment variables
Create a `.env` file or set in your environment:
```env
DATABASE_URL=postgresql://user:password@host:5432/axora
SESSION_SECRET=your-secret-here
NODE_ENV=development
```

### 4. Push database schema
```bash
pnpm --filter @workspace/db run push
```

### 5. Run codegen (regenerate API hooks)
```bash
pnpm --filter @workspace/api-spec run codegen
```

### 6. Start the API server
```bash
pnpm --filter @workspace/api-server run dev
```

### 7. Start the frontend
```bash
pnpm --filter @workspace/axora run dev
```

The app will be available at `http://localhost:<PORT>`.

---

## Project Structure

```
axora/
├── artifacts/
│   ├── axora/                    # React + Vite frontend
│   │   ├── src/
│   │   │   ├── pages/            # Dashboard, Transactions, Fraud, Goals, Analytics, Settings
│   │   │   ├── components/       # Layout, UI primitives
│   │   │   ├── hooks/            # Custom hooks
│   │   │   └── index.css         # Tailwind + theme variables
│   │   └── public/               # Static assets (logo, etc.)
│   └── api-server/               # Express 5 backend
│       └── src/
│           ├── routes/           # transactions, fraud, goals, analytics
│           ├── lib/              # fraud.ts — detection engine
│           └── app.ts            # Express setup
├── lib/
│   ├── api-spec/openapi.yaml     # OpenAPI 3.1 contract (source of truth)
│   ├── api-client-react/         # Orval-generated React Query hooks
│   ├── api-zod/                  # Orval-generated Zod validators
│   └── db/                       # Drizzle ORM schema + client
└── scripts/                      # Utility scripts
```

---

## Fraud Detection Engine

The AI fraud engine (`lib/fraud.ts`) uses **15+ regex pattern rules** to score incoming messages and transactions:

| Pattern | Weight | Example |
|---------|--------|---------|
| Urgency click bait | +20 | "Click here now" |
| Account verification phishing | +25 | "Verify your account" |
| Prize scam | +30 | "You have won ₹5,000" |
| OTP phishing | +25 | "Send your OTP" |
| Insecure HTTP link | +15 | `http://` in message |
| Shortened URL | +20 | `bit.ly`, `tinyurl` |
| Urgency pressure | +15 | "Act now", "Limited time" |
| Suspicious TLD | +25 | `.xyz`, `.tk`, `.ml` |
| Wire transfer request | +30 | "Western Union", "wire transfer" |

Transaction fraud scoring uses:
- **Anomaly detection**: amount > 2× average daily spend → flag
- **Location analysis**: international/offshore keywords → +35 score
- **Absolute thresholds**: > ₹1,000 → +10, > ₹5,000 → +15

---

## Presentation (Canva PPT)

Create a full presentation of Axora in Canva:

**[Open Canva Presentation Template](https://www.canva.com/design/DAGqM8AAAAA/axora-pitch-deck/edit)**

Or start from scratch:
**[Create new Canva Presentation](https://www.canva.com/presentations/templates/pitch-deck/)**

Suggested slides:
1. Cover — Axora logo + tagline
2. Problem Statement — Financial fraud in India
3. Solution Overview — Axora dashboard
4. Key Features — Finance Meter, Fraud Shield, Goals
5. System Architecture Diagram (use diagram from this README)
6. Database Schema
7. Tech Stack
8. Demo Screenshots
9. Roadmap
10. Team / Contact

---

## Commit History (5 logical commits)

| # | Commit | Changes |
|---|--------|---------|
| 1 | `feat: OpenAPI contract + Drizzle schema` | `lib/api-spec/openapi.yaml`, `lib/db/schema/*`, codegen output |
| 2 | `feat: Express API routes (transactions, fraud, goals, analytics)` | All route files, fraud engine (`lib/fraud.ts`), DB seed data |
| 3 | `feat: React frontend scaffold + design system` | `artifacts/axora/src/`, layout, pages, CSS theme |
| 4 | `feat: Dashboard redesign + INR currency + new logo` | `index.css` colour overhaul, `layout.tsx` logo, `dashboard.tsx` redesign |
| 5 | `feat: 5-mode Fraud Scanner + README + architecture docs` | `fraud.tsx` with text/image/audio/video/screenshot inputs, `README.md` |

---

## License

MIT — built with love for the Axora hackathon project.
