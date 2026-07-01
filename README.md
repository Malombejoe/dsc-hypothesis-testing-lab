# SQL Academy — Full-Stack SQL Learning Platform

A production-ready SQL learning platform: 16 curriculum modules, 100 hand-written challenges, a real PostgreSQL sandbox, an AI tutor powered by Claude, gamification, and dedicated tracks for banking, business analytics, Power BI/DAX migration, and Python+SQL.

## Architecture

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│   React SPA  │ ───▶ │  Express API  │ ───▶ │ PostgreSQL  │
│  (Vite, JS)  │ JWT  │  (Node 22)    │  pg  │  (sandbox + │
└─────────────┘      └──────┬───────┘      │   app data)  │
                             │              └─────────────┘
                             ▼
                     ┌───────────────┐
                     │ Anthropic API  │
                     │ (AI Tutor)     │
                     └───────────────┘
```

- **Frontend**: React 18 + Vite, no external UI framework — all components hand-rolled with CSS variables for theming (light/dark).
- **Backend**: Node.js + Express, JWT auth, rate limiting, helmet security headers.
- **Database**: PostgreSQL. App data (users, progress, XP) lives in `public` schema. Learning datasets live in an isolated `sandbox` schema.
- **SQL Sandbox**: User queries run as real PostgreSQL against the `sandbox` schema, with a statement allowlist (SELECT/WITH only by default), 5s timeout, and 500-row cap. A separate `demo-write` endpoint allows INSERT/UPDATE/DELETE practice inside a transaction that is **always rolled back**.
- **AI Tutor**: Calls the Anthropic API (`claude-sonnet-4-6`) for chat, query review, error explanation, and progressive hints.

## Quick start (Docker)

```bash
cp backend/.env.example backend/.env
# edit backend/.env and set ANTHROPIC_API_KEY, JWT_SECRET, DB_PASSWORD

docker compose up --build
```

- Frontend: http://localhost:3000
- Backend API: http://localhost:3001/api/health
- Postgres: localhost:5432

The backend container runs migrations + seed data automatically on first boot.

## Local development (without Docker)

### 1. Database
```bash
# Requires a local PostgreSQL 14+
createdb sql_academy
```

### 2. Backend
```bash
cd backend
cp .env.example .env   # fill in DB creds + ANTHROPIC_API_KEY
npm install
npm run seed            # creates schema + seeds sandbox data
npm run dev              # http://localhost:3001
```

### 3. Frontend
```bash
cd frontend
npm install
npm run dev               # http://localhost:3000
```

## Project structure

```
sql-academy/
├── backend/
│   ├── src/
│   │   ├── index.js              # Express app entry
│   │   ├── db/
│   │   │   ├── pool.js           # pg connection pool
│   │   │   ├── migrate.js        # schema creation
│   │   │   └── seed.js           # seed data generator
│   │   ├── middleware/
│   │   │   └── auth.js           # JWT auth middleware
│   │   ├── routes/
│   │   │   ├── auth.js           # register/login/me
│   │   │   ├── courses.js        # 16-module curriculum data
│   │   │   ├── progress.js       # lessons/quiz/XP/leaderboard
│   │   │   ├── playground.js     # sandbox query execution
│   │   │   ├── challenges.js     # 100 challenges + submission grading
│   │   │   └── ai.js             # Anthropic-powered tutor endpoints
│   │   └── services/
│   │       └── sqlSandbox.js     # safe SQL execution layer
│   ├── Dockerfile
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── App.jsx                # entire SPA (Home/Learn/Playground/
│   │   │                          #  Challenges/Quiz/AI Tutor/Dashboard)
│   │   └── main.jsx
│   ├── index.html
│   ├── nginx.conf
│   ├── Dockerfile
│   └── vite.config.js
├── docker-compose.yml
└── .github/workflows/ci.yml
```

## Curriculum (16 modules)

| # | Module | Level |
|---|---|---|
| 1 | Database Fundamentals | Beginner |
| 2 | Basic Queries | Beginner |
| 3 | Filtering Data | Beginner |
| 4 | Aggregation & Grouping | Beginner |
| 5 | Joins | Intermediate |
| 6 | Advanced Queries (subqueries, CTEs) | Intermediate |
| 7 | Window Functions | Intermediate |
| 8 | Data Manipulation (DML) | Intermediate |
| 9 | Database Design (normalization, star schema) | Intermediate |
| 10 | Business Analytics Projects | Intermediate |
| 11 | SQL for Business Analysts | Intermediate |
| 12 | SQL for Banking (PAR, NPL, ALM) | Intermediate |
| 13 | Collections & Recoveries | Intermediate |
| 14 | Power BI / DAX → SQL migration | Intermediate |
| 15 | Python + SQL Integration | Intermediate |
| 16 | SQL Interview Preparation | Intermediate |

100 challenges span all 16 modules (30 easy, 40 medium, 30 hard) — see `backend/src/routes/challenges.js` for the full bank, including banking-specific (PAR30, NPL ratio, collection recovery rate), DAX-conversion, DML, and recursive-CTE challenges.

## Sandbox datasets

The `sandbox` schema ships with 10 interrelated tables seeded with realistic data:

`customers`, `orders`, `products`, `employees`, `bank_customers`, `accounts`, `loans`, `transactions`, `collections`, `sales`.

## Security notes

- Sandbox queries are restricted to `SELECT`/`WITH` by pattern-matching + statement timeout; DML practice runs inside a transaction that's rolled back unconditionally.
- Passwords hashed with bcrypt (cost 12).
- JWT tokens expire after 7 days.
- Rate limiting on `/api/auth` (20/15min), `/api/playground` (60/min), `/api/ai` (10/min).
- Helmet sets standard security headers; CORS restricted to `FRONTEND_URL`.

This sandbox approach is good for an educational MVP. For a public production deployment at scale, additionally consider: a dedicated read-only DB role for the sandbox connection, query plan cost limits, and running the sandbox in a separate container/DB instance from app data.

## Scaling to 100k+ users

- Add a read replica for the sandbox schema; route all `/api/playground` and `/api/challenges/submit` reads there.
- Move AI tutor calls to a queue (e.g., BullMQ + Redis) if latency/rate limits become an issue at scale.
- Add Redis for session/leaderboard caching (`progress/leaderboard` is a good caching candidate — refresh every 60s).
- Move static frontend to a CDN (Vercel/Cloudflare) — the included Dockerfile/nginx setup is for self-hosting; Vercel deployment is simpler for the frontend alone (`vercel deploy` from `/frontend`, set `VITE_API_URL`).
- Horizontally scale the Express backend behind a load balancer; it's stateless (JWT, no server-side sessions) so this requires no code changes.

## Testing

See `backend/src/__tests__` (smoke tests for sandbox query allowlisting) and `.github/workflows/ci.yml` for the CI pipeline (lint + build + sandbox tests on every push).
