# CivicSentinel:

AI infrastructure-reporting platform. A citizen photographs a problem, a vision AI classifies it, the report is stored
in Supabase, and city admins see it on the map and dashboard.

```
Citizen: photo -> validate -> AI analysis -> type/confidence/severity -> confirm + location -> submit
                                   |
                         Supabase incident (+ photo in Storage)
                                   |
Admin:   real incident -> map marker -> details -> priority -> status
```

| Route | Who | What |
|-------|-----|------|
| `/login` | everyone | Email/password sign-in (Supabase Auth) |
| `/user/*` | `citizen` | Mobile app: home map, report flow, incidents, My reports, details, profile |
| `/admin/*` | `admin` | Desktop console: dashboard (map, selected incident, status), map, incidents table, settings |

## Two modes, one codebase
| Mode | When | Data | AI |
|------|------|------|----|
| **Real** | `VITE_SUPABASE_URL` + `VITE_SUPABASE_ANON_KEY` set | Supabase tables + private Storage, enforced by RLS | Real vision AI via the `analyze-image` Edge Function if its key is set |
| **Demo** | those values missing (`npm run dev`) | Sample incidents + your reports, in this browser only | **DEMO/FALLBACK** analysis, always labelled "Demo / fallback analysis" |

A fallback result is **never** shown as real AI: the badge, the notice, and the stored `ai_source` column all say which it was.
The fallback also kicks in when the AI key is missing, the AI call fails or times out, or the answer can't be parsed.

## Quick start
```bash
npm install
cp .env.example .env.local   # optional: leave blank for demo mode
npm run dev                  # http://localhost:5173
```
Demo accounts: `citizen@demo.test` / `admin@demo.test`, password `demo1234`.
Real setup (Supabase, SQL, AI key): **[MANUAL_TASKS.md](./MANUAL_TASKS.md)**.

## How it works
- **Photo**: JPEG/PNG/WebP, max 10 MB, min 200 px side; decoded and resized to 1280 px JPEG in the browser (`src/lib/image.js`).
- **AI** (`src/lib/ai.js`, `supabase/functions/`): the browser calls the `analyze-image` Edge Function; the function holds the API key, calls the vision model and returns
  `{type, confidence 0-100, severity, description, evidence}`. `_shared/parseAnalysis.js` tolerates fences, extra prose, `"87%"`, 0-1 fractions, synonyms (`streetlight` -> `broken_streetlight`) and rejects anything unusable. The browser re-validates the response.
  Types: `pothole | road_damage | broken_streetlight | drainage | garbage | other`.
- **Priority score** (`src/lib/priority.js`), 0-100, deterministic:
  `severity base (low 25, medium 45, high 65, critical 85)` `+ confidence adjustment (-10..+10, centred on 50%)` `+ type nudge (drainage +5, pothole +3, streetlight +3, road damage +2, garbage 0, other -5)`.
  Labels: >=85 Critical, >=70 High, >=45 Medium, else Low. All numbers are constants at the top of the file.
- **Data layer** (`src/lib/incidents.js` + pure `incidentModel.js`): the only place that decides Supabase vs demo store. Screens use `useIncidents`, `useIncident`, `createIncident`, `updateIncident`.
- **Security**: row-level security in `supabase/schema.sql`. Citizens read/create only their own reports and cannot edit anything; admins read all and update status/priority; photos are in a private bucket under `<user id>/`. React route guards are UX only.
- **Role** comes only from `profiles.role` (never `user_metadata`); admins are promoted by SQL.

## Scripts
| Command | Purpose |
|---------|---------|
| `npm run dev` / `build` / `preview` | Dev server / production build / preview |
| `npm test` | Unit tests: roles, parser, server AI call (fake fetch), priority, model, image rules |
| `npm run test:demo` | Data-layer integration test in demo mode (create, role-scoped lists, status update, error paths) |
| `npm run test:contract` | Supabase-mode request shapes + AI labelling against a **stubbed** server |
| `npm run test:all` | All of the above |

## Project layout
```
src/
  auth/        AuthContext, ProtectedRoute, roles.js
  lib/         supabase.js, demoAuth.js, incidents.js (data), incidentModel.js, priority.js, ai.js, image.js
  data/        constants + demo seed data
  components/  Logo, badges, RiskRing, MapView (Leaflet + location picker), IncidentPhoto
  citizen/     Report (flow), MyReports, IncidentDetail, Home, NearbyIncidents, Profile
  admin/       Dashboard, IncidentMap, IncidentTable, IncidentsPage, MapPage, Settings
supabase/
  schema.sql                       profiles, incidents, RLS, storage bucket + policies
  functions/_shared/               parseAnalysis.js, analyze.js (pure, tested)
  functions/analyze-image/index.ts Edge Function (thin wrapper)
scripts/       demo-flow.mjs, supabase-contract.mjs
```

## Known limits (Part 2)
- With RLS, a citizen's "Nearby" map shows **their own** reports only (other citizens' reports are private). A privacy-safe public view is a Part 3 decision.
- Type/severity/priority are computed in the browser from the AI result and the citizen's edits, so a malicious citizen could submit inflated values through the API. Moving scoring into the Edge Function or a database trigger is a recommended hardening step.
- The admin "AI Insights" card is static sample text. Reports/Analytics pages are placeholders. No notifications, worker app or repair verification.
- Admin data refreshes by polling every 20 s (no realtime subscription).
