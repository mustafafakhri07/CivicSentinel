# MANUAL_TASKS.md: things you must do yourself

Part 1 never touches your Supabase project for you. Do these steps once, in order.
Nothing here requires the `service_role` key, and you must **never** put that key in this app or in any `VITE_` variable.

---

## 1. Create a Supabase project
1. Go to <https://supabase.com/dashboard> and click **New project**.
2. Pick a name, region and database password, then wait until the project is ready.

## 2. Copy the public API values into `.env.local`
1. In the dashboard open **Project Settings → API** (or **API Keys**).
2. Copy:
   - **Project URL**, e.g. `https://abcdefghijklmnop.supabase.co`
   - the **anon public** key (or the newer **publishable** key, `sb_publishable_...`)
3. In the project root:
   ```bash
   cp .env.example .env.local
   ```
4. Edit `.env.local`:
   ```env
   VITE_SUPABASE_URL=https://YOUR-PROJECT-REF.supabase.co
   VITE_SUPABASE_ANON_KEY=YOUR-ANON-OR-PUBLISHABLE-KEY
   ```
5. Restart `npm run dev` (Vite only reads env files at start-up).

> Do **not** use the `service_role` / secret key. Everything prefixed `VITE_` is shipped to every visitor's browser.

## 3. Create the `profiles` table, RLS policies and trigger
Open **SQL Editor → New query**, paste **all** of the SQL below (it is also saved as `supabase/schema.sql`) and click **Run**. It is safe to re-run.

```sql
-- CivicSentinel Part 1: profiles + roles.
-- Run this once in Supabase Dashboard -> SQL Editor. It is safe to re-run.

-- 1) Profiles table: one row per auth user. `role` lives HERE, never in user_metadata.
create table if not exists public.profiles (
  id         uuid primary key references auth.users (id) on delete cascade,
  email      text not null,
  role       text not null default 'citizen' check (role in ('citizen', 'admin')),
  created_at timestamptz not null default now()
);

alter table public.profiles enable row level security;

-- 2) Helper used by policies (here and in Part 2). SECURITY DEFINER so it can read profiles
--    without recursing into the profiles RLS policies.
create or replace function public.is_admin()
returns boolean
language sql
stable
security definer
set search_path = public
as $$
  select exists (
    select 1 from public.profiles where id = auth.uid() and role = 'admin'
  );
$$;

revoke all on function public.is_admin() from public, anon;
grant execute on function public.is_admin() to authenticated;

-- 3) Read-only access from the client. There are deliberately NO insert/update/delete
--    policies and the table privileges are revoked, so a signed-in user can never
--    change their own role (or anyone else's) from the browser.
drop policy if exists "profiles_select_own" on public.profiles;
create policy "profiles_select_own" on public.profiles
  for select to authenticated
  using (id = auth.uid());

drop policy if exists "profiles_select_admin" on public.profiles;
create policy "profiles_select_admin" on public.profiles
  for select to authenticated
  using (public.is_admin());

revoke all on public.profiles from anon, authenticated;
grant select on public.profiles to authenticated;

-- 4) Auto-create a profile for every new auth user. Always 'citizen': the role is NOT taken
--    from signup metadata, so nobody can sign up as admin.
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = public
as $$
begin
  insert into public.profiles (id, email, role)
  values (new.id, coalesce(new.email, ''), 'citizen')
  on conflict (id) do nothing;
  return new;
end;
$$;

revoke all on function public.handle_new_user() from public, anon, authenticated;

drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();

-- 5) Backfill profiles for users that already existed before this script ran.
insert into public.profiles (id, email, role)
select id, coalesce(email, ''), 'citizen'
from auth.users
on conflict (id) do nothing;
```

## 4. Check the email/password provider
**Authentication → Providers → Email** must be enabled (it is by default).
While testing you can turn **Confirm email** off (Authentication → Sign In / Providers → Email), or tick **Auto Confirm User** when creating users in step 5.

## 5. Create your two test users
**Authentication → Users → Add user → Create new user**
- Enter an email and password for a **citizen**, tick **Auto Confirm User**, create.
- Do the same for an **admin** account (a different email).

The trigger from step 3 gives both of them the role `citizen`.

## 6. Promote ONE user to admin
Role changes are only possible from the SQL Editor (or later, a trusted backend), never from the browser.
In **SQL Editor** run (use your admin's email):

```sql
update public.profiles
set role = 'admin'
where email = 'admin@example.com';
```

Check the result:

```sql
select email, role, created_at from public.profiles order by created_at;
```

If the admin signed in before this step, ask them to sign out and in again.

## 7. (Recommended) Decide about public sign-ups
Part 1 has no sign-up screen, but Supabase's API still allows anyone holding your anon key to register unless you turn it off.
New accounts are always `citizen`, so this is not a privilege problem, but you may still prefer to close it until Part 2 adds registration:
**Authentication → Sign In / Providers → "Allow new users to sign up"** → off.

## 8. Before deploying
- **Authentication → URL Configuration**: set **Site URL** to your production URL and add it (and `http://localhost:5173`) under **Redirect URLs**.
- Add `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` to your host's environment variables (Vercel, Netlify, ...).
- Configure an SPA fallback so deep links such as `/admin` serve `index.html`
  (Vercel: rewrite `/(.*)` → `/index.html`; Netlify: `/* /index.html 200` in `public/_redirects`).

---

## Manual test checklist

| # | Test | Expected |
|---|------|----------|
| 1 | Open `/` signed out | Redirects to `/login` |
| 2 | Open `/user` and `/admin` signed out | Both redirect to `/login` |
| 3 | Wrong password | "Invalid email or password." and you stay on `/login` |
| 4 | Sign in as the citizen | Lands on `/user` |
| 5 | As citizen, type `/admin` in the address bar | Sent back to `/user` |
| 6 | Refresh any page while signed in | Stays signed in, same page |
| 7 | Citizen → Profile → **Sign out** | Back on `/login`; `/user` now redirects to `/login` |
| 8 | Sign in as the admin | Lands on `/admin` dashboard |
| 9 | As admin, open `/user` | Sent to `/admin` |
| 10 | Admin → avatar menu → **Sign out** | Back on `/login` |
| 11 | As the citizen, try to `update` your own `profiles.role` to `admin` (app code or REST call with your token) | Rejected, because there is no update privilege or policy |
| 12 | Sign in with a user that has no `profiles` row (delete it in SQL to test) | "We couldn't load your account" screen, **no** access |

## Running without Supabase (demo mode)
If the two env values are missing, `npm run dev` starts in **demo mode** with built-in fake accounts so you can explore the UI:

| Role | Email | Password |
|------|-------|----------|
| Citizen | `citizen@demo.test` | `demo1234` |
| Admin | `admin@demo.test` | `demo1234` |

Demo mode is switched off automatically as soon as Supabase is configured, and it is **not** available in production builds unless you set `VITE_ALLOW_DEMO=true`.
