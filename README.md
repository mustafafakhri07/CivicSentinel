# CivicSentinel — Part 1: Frontend + Auth

AI infrastructure-reporting platform. **This part ships the two interfaces and authentication only.**
Incident data, maps markers and AI results are **mock**; real AI vision, incident processing, the risk engine and backend integration are Part 2.

| Route | Who | What |
|-------|-----|------|
| `/login` | everyone | Email/password sign-in (Supabase Auth) |
| `/user/*` | `citizen` | Mobile-first app: home map, report flow, nearby incidents, my reports, incident details, profile |
| `/admin/*` | `admin` | Desktop command center: dashboard, map, incidents, settings (Reports/Analytics are placeholders) |

## Stack
React 18 · Vite · React Router 6 · Supabase JS (auth + `profiles`) · Leaflet + OpenStreetMap (no API key) · lucide-react icons · plain CSS (no UI framework).

## Quick start
```bash
npm install
cp .env.example .env.local     # then fill in your Supabase values (see MANUAL_TASKS.md)
npm run dev                    # http://localhost:5173
```

**No Supabase yet?** Just run `npm run dev`. The app starts in *demo mode* with `citizen@demo.test` / `admin@demo.test` (password `demo1234`). The login page shows a clear banner, and the app never crashes on missing env values.

**Supabase setup:** follow [`MANUAL_TASKS.md`](./MANUAL_TASKS.md): project, env values, SQL (`supabase/schema.sql`), test users, promoting an admin.

## Scripts
| Command | Purpose |
|---------|---------|
| `npm run dev` | Dev server |
| `npm run build` / `npm run preview` | Production build / preview |
| `npm test` | Unit tests for the role-routing rules (`node:test`, no extra dependencies) |

## How auth and roles work
- `profiles(id, email, role citizen|admin, created_at)`, one row per auth user, created by a database trigger.
- The app reads the role **only** from `profiles` (never from `user_metadata`, which users can edit).
- Clients have **read-only** access to `profiles` (RLS + revoked privileges), and the signup trigger always assigns `citizen`. Admins are promoted manually via SQL, so a role can't be changed from the browser.
- Sessions persist in `localStorage` via supabase-js and survive refresh.
- Routing rules live in `src/auth/roles.js` (pure + unit-tested): signed out → `/login`; citizen → `/user`; admin → `/admin`; a citizen opening `/admin` is sent to `/user`; an admin opening `/user` is sent to `/admin`; a signed-in user without a valid role gets an "account problem" screen and **no access**.
- **Important:** route guards are a UX layer. Anything sensitive in Part 2 must be enforced by Supabase RLS (use `public.is_admin()`), not by the React app.

## Project layout
```
src/
  auth/        AuthContext, ProtectedRoute, roles.js (+ test)
  lib/         supabase client (env-safe), demo auth fallback
  data/        mock incidents / stats
  components/  Logo, badges, RiskRing, MapView (Leaflet), IncidentPhoto (placeholder art)
  citizen/     mobile screens
  admin/       desktop screens
  pages/       Login
supabase/schema.sql
```

## What is mocked in Part 1
Incidents, counts, AI insight text, the "Analyzing with AI" screen and its "Pothole detected" result, citizen stats, and the incident photos (generated placeholder art; your own uploaded photo is only previewed locally and never sent anywhere). The map's **Satellite** toggle is shown but disabled (OpenStreetMap only).
