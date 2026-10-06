# CivicSentinel: Final Complete Release (Parts 1 + 2 + 3)

CivicSentinel is an end-to-end AI infrastructure-reporting platform connecting citizens and municipal operations.
A citizen photographs an issue (pothole, streetlight, drainage, road damage, garbage), a vision AI analyzes it, priority scoring is enforced server-side, incidents are stored in Supabase, and municipal administrators monitor, prioritize, and manage repairs with live telemetry and duplicate detection.

```
Citizen:  photo -> validate -> AI analysis -> type/confidence/severity -> confirm + location -> submit
                                           |
                                 Supabase incident (+ photo in private Storage)
                                 Server-side priority scoring & trigger enforcement
                                           |
Admin:    live telemetry -> map marker -> details -> duplicate detection -> priority -> status update
                                           |
                                 Realtime status update
                                           |
Citizen:  My Reports reflects updated status instantly
```

| Route | Who | What |
|-------|-----|------|
| `/login` | everyone | Email/password sign-in (Supabase Auth + demo mode fallback) |
| `/user/*` | `citizen` | Mobile app: home map, report flow, community nearby incidents (privacy-safe), My reports, details, profile |
| `/admin/*` | `admin` | Desktop console: dashboard (map, selected incident, status management, related reports, live telemetry), map, incidents table, settings |

## Two Modes, One Codebase

| Mode | When Active | Data Storage | AI Analysis |
|------|-------------|--------------|-------------|
| **DEMO MODE** *(Default)* | Missing env variables or `npm run dev` with blank `.env.local` | In-memory + browser `localStorage`, seeded from city dataset | Deterministic, clearly labelled **DEMO / FALLBACK** analysis |
| **REAL DEPLOYMENT** | `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` configured | Real PostgreSQL tables + private Storage buckets with RLS | Vision AI model via `analyze-image` Supabase Edge Function |

Fallback results are **never** disguised as real AI: badges and the database `ai_source` column explicitly distinguish `real` vs `demo`.

## Quick Start (Demo Mode - No Credentials Needed)

```bash
npm install
npm run dev                  # Open http://localhost:5173
```

Demo Accounts:
- **Citizen**: `citizen@demo.test` / `demo1234`
- **Admin**: `admin@demo.test` / `demo1234`

Full step-by-step instructions for real Supabase and AI setup are in **[MANUAL_TASKS.md](./MANUAL_TASKS.md)**.

## Key Architecture & Features

1. **Server-Side Priority Scoring (`supabase/schema.sql`, `src/lib/priority.js`)**:
   - Deterministic 0–100 scoring based on severity base (low 25, medium 45, high 65, critical 85), AI confidence adjustment (-10..+10), and contextual issue type weight.
   - Enforced server-side in PostgreSQL via `calculate_priority()` and trigger `trg_incidents_enforce_priority` before insert or update to prevent client manipulation.

2. **Citizen Privacy & Public Community Incidents (`supabase/schema.sql`, `src/lib/incidents.js`)**:
   - Community safety map displays nearby issues without leaking private citizen information.
   - Database view `public.public_incidents` exposes only safe columns (`id`, `ticket_no`, `type`, `severity`, `priority_score`, `status`, coordinates).
   - **Never exposes** citizen ID, email, private description, AI evidence, or private storage paths.

3. **Admin Intelligence & Live Telemetry (`src/lib/incidentModel.js`, `src/admin/Dashboard.jsx`)**:
   - Dynamic calculations from real incident data: total cases, high/critical breakdown, severity distribution, status resolution counts, and city-wide average priority.
   - Dynamic pattern detection detects cluster hotspots, dominant issue types, and recent activity trends.

4. **Related & Duplicate Reports Detection (`src/lib/incidentModel.js`, `src/admin/Dashboard.jsx`)**:
   - Lightweight, deterministic duplicate detection using geographic proximity (Haversine distance <= 800m) and matched/related issue types.
   - Displayed directly inside the Admin selected incident drawer with distance badges.

5. **Real-time Status Synchronization (`src/lib/incidents.js`)**:
   - Supports Supabase Realtime (`postgres_changes`) publication subscriptions.
   - Instant cross-tab and in-memory event dispatching in demo mode so that Admin status changes (e.g. `submitted` -> `under_review`) immediately reflect in Citizen's "My Reports" view.
   - Resilient polling fallback (every 20s) ensures reliable updates under all network conditions.

## Scripts & Verification

| Command | Purpose |
|---------|---------|
| `npm run dev` | Run Vite development server |
| `npm run build` | Run production Vite build |
| `npm run preview` | Preview production build locally |
| `npm test` | Unit tests: roles, parser, server AI call, priority, model, related reports, intelligence, privacy |
| `npm run test:demo` | Data layer integration test in demo mode |
| `npm run test:contract` | Supabase-mode request contract tests against stubbed server |
| `npm run test:e2e` | End-to-end Part 3 verification (Citizen flow, Admin flow, duplicate detection, privacy, sync) |
| `npm run test:all` | Run all test suites |

## Setup Instructions

### REQUIRED FOR REAL DEPLOYMENT
1. Follow **[MANUAL_TASKS.md](./MANUAL_TASKS.md)** to:
   - Create Supabase project & configure `.env.local`
   - Run `supabase/schema.sql` in the Supabase SQL Editor
   - Configure Auth and promote admin user via SQL
   - Deploy `analyze-image` Edge Function with `GEMINI_API_KEY`
2. Run `npm run build` and deploy dist to host with SPA redirects.

### DEMO MODE (Out-of-the-Box)
- Requires zero API keys or external services.
- Simply run `npm run dev` or `npm run preview`.
