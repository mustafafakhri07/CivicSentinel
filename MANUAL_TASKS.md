# MANUAL_TASKS.md: things you must do yourself

Nothing here needs the `service_role` key. **Never** put that key, or the AI key, in this app or in any `VITE_` variable
(everything prefixed `VITE_` is shipped to every visitor's browser).

The app works **before** you do any of this (demo mode, see the bottom). Do these steps once, in order, when you are ready.

| Step | Needed for | If you skip it |
|------|------------|----------------|
| 1-2 | Any real backend | App stays in demo mode |
| 3 | Incidents, RLS, photo storage | Reports cannot be saved |
| 4-6 | Logging in as real citizen/admin | No real accounts |
| 7-8 | **Real AI** photo analysis | App still works, using the clearly labelled **DEMO/FALLBACK** analysis |

---

## 1. Create a Supabase project
1. <https://supabase.com/dashboard> -> **New project**. Pick a name, region and database password, wait until it is ready.
2. Note the **Project ref** (the `abcdefghijklmnop` in `https://abcdefghijklmnop.supabase.co`).

## 2. Put the public API values in `.env.local`
1. **Project Settings -> API** (or **API Keys**): copy the **Project URL** and the **anon public** key (or the newer **publishable** key `sb_publishable_...`).
2. In the project root:
   ```bash
   cp .env.example .env.local
   ```
3. Edit `.env.local`:
   ```env
   VITE_SUPABASE_URL=https://YOUR-PROJECT-REF.supabase.co
   VITE_SUPABASE_ANON_KEY=YOUR-ANON-OR-PUBLISHABLE-KEY
   ```
4. Restart `npm run dev` (Vite only reads env files at start-up). Demo mode switches off automatically.

## 3. Run the database SQL (tables, RLS, storage bucket)
**SQL Editor -> New query**, paste the **entire** `supabase/schema.sql`, click **Run**. It is safe to re-run.

It creates, in this order:

| Part | What |
|------|------|
| 1 | `public.profiles` (+ `is_admin()` helper, signup trigger that always assigns `citizen`) |
| 2 | `public.incidents` table, indexes, `updated_at` trigger |
| 2 | **RLS policies** (below) and column-level grants |
| 2 | Storage bucket `incident-images` (private, 5 MB, jpeg/png/webp) + its policies |
| 3 | **Server-side Priority Trigger** `calculate_priority()` and `trg_incidents_enforce_priority` |
| 3 | **Citizen Privacy View & Function** `public.public_incidents` and `get_public_incidents()` |
| 3 | **Supabase Realtime Publication** for instant status update sync |

### `incidents` columns
`id` (uuid), `ticket_no` (shown as `INF-00042`), `citizen_id` -> `profiles.id`, `type`, `description`, `evidence`,
`image_url` (**storage path**, not a URL), `latitude`, `longitude`, `address`, `confidence` (0-100), `severity`,
`priority_score` (0-100, **server-enforced** via DB trigger), `status` (`submitted | under_review | in_progress | resolved`),
`ai_source` (`real | demo`: `demo` means the fallback result, not a real AI analysis), `created_at`, `updated_at`.

### RLS policies on `incidents`
| Policy | Who | Rule |
|--------|-----|------|
| `incidents_select_own` | citizen | `select` where `citizen_id = auth.uid()` |
| `incidents_select_admin` | admin | `select` everything (`public.is_admin()`) |
| `incidents_insert_own` | citizen | `insert` only with `citizen_id = auth.uid()` **and** `status = 'submitted'` |
| `incidents_update_admin` | admin | `update` any incident, limited by column grant to `status, priority_score, severity, type, description` |
| (none) | everyone | **No update/delete for citizens, no delete for anyone**, `anon` has no access at all |

### Citizen Privacy View (`public.public_incidents`)
Provides community awareness while strictly protecting citizen anonymity and privacy:
- Selects only `id`, `ticket_no`, `type`, `severity`, `priority_score`, `status`, `latitude`, `longitude`, `created_at`.
- **Never exposes**: `citizen_id`, email, private descriptions, AI evidence, or storage image paths.
- RLS on `public.incidents` remains uncompromised.

### Storage policies on `storage.objects` (bucket `incident-images`)
| Policy | Rule |
|--------|------|
| `incident_images_insert_own` | a signed-in user may upload only into the folder named after their own user id (`<uid>/<file>`) |
| `incident_images_select_own_or_admin` | read your own folder; admins read all |
| (none) | no update/delete: uploaded evidence is immutable from the browser |

The bucket is **private**; the app shows photos through 1-hour signed URLs.

Verify (SQL editor):
```sql
select policyname, cmd from pg_policies where tablename = 'incidents';
select id, public, file_size_limit from storage.buckets where id = 'incident-images';
```
You should see 4 policies and `public = false`. Also check **Storage** in the dashboard shows the `incident-images` bucket.

## 4. Check Auth settings
**Authentication -> Providers -> Email** must be enabled. While testing, turn **Confirm email** off, or tick **Auto Confirm User** in step 5.

## 5. Create two test users
**Authentication -> Users -> Add user -> Create new user**: one **citizen**, one **admin** (different emails), both with **Auto Confirm User**.
The trigger gives both the role `citizen`.

## 6. Promote ONE user to admin
Roles can only be changed from the SQL Editor, never from the browser:
```sql
update public.profiles set role = 'admin' where email = 'admin@example.com';
select email, role from public.profiles order by created_at;
```
If the admin was already signed in, sign out and in again.

## 7. Get an AI key (REAL AI)
The photo analysis uses a vision model through Google Gemini's REST API. Create an API key at <https://aistudio.google.com/>.
The key lives **only** as a Supabase Edge Function secret. It is never in the browser, never in `.env.local`.

## 8. Deploy the AI Edge Function
The function (`supabase/functions/analyze-image`) needs the Supabase CLI (no install needed with `npx`):
```bash
npx supabase login
npx supabase link --project-ref YOUR-PROJECT-REF
npx supabase secrets set GEMINI_API_KEY=PASTE-YOUR-KEY-HERE
# optional: npx supabase secrets set GEMINI_MODEL=gemini-1.5-flash
npx supabase functions deploy analyze-image
```
Leave **JWT verification on** (the default): only signed-in users can call it.

**How to tell which AI you got:** on the report result screen a green **"Real AI analysis"** badge means the model analysed the photo.
An amber **"Demo / fallback analysis"** badge plus a yellow notice (with the reason) means *no AI looked at it*.
The same label is stored per incident (`ai_source`) and shown to admins. "AI Verified Incidents" on the dashboard counts only `real`.

**Turn real AI off:** `npx supabase secrets unset GEMINI_API_KEY`. The app falls back to the demo analysis automatically.
If the model name is rejected, set `GEMINI_MODEL` to a vision-capable Gemini model you have access to.

## 9. (Recommended) Decide about public sign-ups
There is no sign-up screen, but the Supabase API still allows registration with your anon key unless you turn it off.
New accounts are always `citizen`, so it is not a privilege problem. To close it: **Authentication -> Sign In / Providers -> "Allow new users to sign up"** -> off.

## 10. Before deploying
- **Authentication -> URL Configuration**: set **Site URL** to your production URL and add it (and `http://localhost:5173`) under **Redirect URLs**.
- Add `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` to your host's environment variables (Vercel, Netlify...). The AI key is **not** added there.
- Add an SPA fallback so deep links like `/admin` serve `index.html` (Vercel: rewrite `/(.*)` -> `/index.html`; Netlify: `/* /index.html 200` in `public/_redirects`).

---

## Manual test checklist (run after steps 1-8)

### Auth and roles (Part 1)
| # | Test | Expected |
|---|------|----------|
| 1 | Open `/` signed out | Redirects to `/login` |
| 2 | Wrong password | "Invalid email or password." |
| 3 | Sign in as citizen | Lands on `/user`; typing `/admin` sends you back to `/user` |
| 4 | Sign in as admin | Lands on `/admin`; opening `/user` sends you to `/admin` |
| 5 | As citizen, in the browser console try `update profiles set role='admin'` through the client | Rejected |

### Core flow (Part 2)
| # | Test | Expected |
|---|------|----------|
| 6 | Citizen -> Report a problem -> choose a **photo of a road problem** | "Analyzing with AI", then result with type, confidence, severity and a green **Real AI analysis** badge |
| 7 | Choose a PDF or a >10 MB image | Clear error, no upload |
| 8 | Edit type/severity/description, tap the map to move the pin, **Submit report** | Success screen with an `INF-xxxxx` code |
| 9 | Citizen -> Incidents (My reports) | The new report is listed with its status `Submitted`; open it: photo, location, priority score |
| 10 | Dashboard -> Table Editor -> `incidents` | Row exists with your `citizen_id`, `status = submitted`, `ai_source = real`, `image_url` like `<uid>/<uuid>.jpg` |
| 11 | Storage -> `incident-images` | Object under your user-id folder |
| 12 | Sign in as admin -> Dashboard | New incident is in the stats, the map (marker) and the table; click the marker: details, photo, severity, confidence, priority, location, submission time |
| 13 | Admin -> change **Status** to Under review / In progress / Resolved | Persists after refresh; citizen sees the new status in My reports |
| 14 | Admin -> Incidents | All incidents, with photo, severity, AI confidence, submitted time |
| 15 | Create a **second citizen** and sign in | My reports is empty: they cannot see the first citizen's report |
| 16 | As a citizen, in the console: `supabase.from('incidents').update({status:'resolved'}).eq('id', '<any id>')` | 0 rows changed / error (no policy) |
| 17 | As a citizen, `supabase.from('incidents').select()` | Only that citizen's own rows |
| 18 | Unset the AI secret (step 8), report again | Amber **Demo / fallback analysis** badge and notice with the reason; report still submits and is stored with `ai_source = demo` |
| 19 | Temporarily rename the bucket / remove storage policy, submit | Report is saved, the success screen says the photo could not be stored |

### Part 3 Advanced Intelligence & Privacy
| # | Test | Expected |
|---|------|----------|
| 20 | Server-side priority integrity | Inserting an incident via client automatically has `priority_score` computed and verified by DB trigger `trg_incidents_enforce_priority` |
| 21 | Citizen Privacy | Citizen visits `/user/map`: sees nearby community markers; other citizens' reports are sanitized (`isPublicSanitized: true`) with no private email, description, or image URLs |
| 22 | Related / Duplicate Reports | Admin selects an incident on Dashboard: sees "Related / Duplicate Reports" listing nearby issues (<800m) of matching or related problem types |
| 23 | Live Admin Intelligence | Admin Dashboard shows live telemetry: dynamic AI Insights, cluster hotspot analysis, resolution counts, and average priority score computed from real incidents |
| 24 | Realtime Status Sync | Admin updates status (e.g. `submitted` -> `under_review`); Citizen viewing "My Reports" sees the updated status immediately |

## Running without Supabase (demo mode)
If the two Supabase values are missing, `npm run dev` starts in **demo mode**:

| Role | Email | Password |
|------|-------|----------|
| Citizen | `citizen@demo.test` | `demo1234` |
| Admin | `admin@demo.test` | `demo1234` |

In demo mode incidents are sample data + whatever you report, stored **in this browser only** (`localStorage`); photos are saved as small thumbnails;
the AI is the **DEMO/FALLBACK** analysis (always labelled). Demo mode is off as soon as Supabase is configured, and is not available in production builds unless `VITE_ALLOW_DEMO=true`.
To reset the demo data: clear site data for `localhost:5173` (or remove the `cs_demo_incidents_v2` key).
