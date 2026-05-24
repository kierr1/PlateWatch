# PlateletWatch — Supabase Setup Guide

## Step 1 — Create a Supabase Project

1. Go to **https://supabase.com** and sign in (or create a free account).
2. Click **New Project**, give it a name (e.g. `plateletwatch`), choose a region close to the Philippines, and set a database password.
3. Wait ~2 minutes for the project to be ready.

---

## Step 2 — Get Your API Credentials

1. In the Supabase dashboard, go to **Project Settings → API**.
2. Copy:
   - **Project URL** (looks like `https://abcdefgh.supabase.co`)
   - **anon / public** key (long string starting with `eyJ…`)
3. Open `supabase.config.js` in this folder and paste them:

```js
const SUPABASE_URL      = 'https://YOUR_PROJECT.supabase.co';
const SUPABASE_ANON_KEY = 'eyJ...your-anon-key...';
```

---

## Step 3 — Create the Database Tables

In the Supabase dashboard, go to **SQL Editor → New Query** and run the following SQL:

```sql
-- ============================================================
-- 1. PROFILES TABLE (extends auth.users)
-- ============================================================
create table public.profiles (
  id             uuid references auth.users on delete cascade primary key,
  first_name     text,
  last_name      text,
  middle_name    text,
  sex            text,
  birthdate      date,
  contact        text,
  street_address text,
  province       text,
  city           text,
  barangay       text,
  role           text default 'staff',
  created_at     timestamptz default now()
);

-- Enable Row Level Security
alter table public.profiles enable row level security;

-- Policies: users can only access their own profile
create policy "Users can view own profile"
  on public.profiles for select
  using (auth.uid() = id);

create policy "Users can insert own profile"
  on public.profiles for insert
  with check (auth.uid() = id);

create policy "Users can update own profile"
  on public.profiles for update
  using (auth.uid() = id);


-- ============================================================
-- 2. AUTO-CREATE PROFILE ON SIGNUP (optional trigger)
-- ============================================================
create or replace function public.handle_new_user()
returns trigger language plpgsql security definer as $$
begin
  insert into public.profiles (id, first_name, last_name)
  values (
    new.id,
    new.raw_user_meta_data->>'first_name',
    new.raw_user_meta_data->>'last_name'
  );
  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();
```

Click **Run** (▶).

---

## Step 4 — Configure Email Auth

1. In Supabase → **Authentication → Providers**, make sure **Email** is enabled.
2. Under **Authentication → Email Templates**, you can customise the confirmation and password-reset emails.
3. For the **Forgot Password** flow, Supabase sends a reset link to the user's email. When they click it, they are redirected back to `Forgotpassword.html` where they can set a new password.

---

## Step 5 — (Optional) Enable Google / Facebook OAuth

### Google
1. Go to **Google Cloud Console → APIs & Services → Credentials** and create an OAuth 2.0 Client ID (Web application).
2. Set **Authorised redirect URI** to:  
   `https://YOUR_PROJECT.supabase.co/auth/v1/callback`
3. In Supabase → **Authentication → Providers → Google**, enable it and paste the Client ID and Secret.

### Facebook
1. Go to **Meta for Developers → My Apps** and create a new app.
2. Add **Facebook Login** product and set the redirect URI similarly.
3. In Supabase → **Authentication → Providers → Facebook**, enable and paste credentials.

---

## File Structure

```
plateletwatch/
├── index.html          ← Landing page
├── Signin.html         ← Sign in (Supabase auth)
├── Register.html       ← Registration (Supabase signUp + profile insert)
├── Forgotpassword.html ← Password reset (Supabase resetPasswordForEmail)
├── all-tab.html        ← Dashboard (session-guarded)
├── styles.css          ← Landing page styles
├── supabase.config.js  ← ⚠️  PUT YOUR CREDENTIALS HERE
└── pics/
    ├── pic1.png
    └── pic2.png
```

---

## How Auth Works (Summary)

| Page | Supabase Call | On Success |
|---|---|---|
| Sign In | `signInWithPassword({ email, password })` | → `all-tab.html` |
| Register | `signUp({ email, password })` + profiles insert | → `Signin.html` |
| Forgot PW step 1 | `resetPasswordForEmail(email)` | Sends email with reset link |
| Forgot PW step 3 | `updateUser({ password })` | → `Signin.html` |
| Log Out | `signOut()` | → `Signin.html` |
| Dashboard load | `getSession()` — no session → redirect | Blocks unauthenticated access |

---

## Hosting

To serve the files locally you **cannot** just double-click `index.html` — browsers block local scripts.
Use a simple local server:

```bash
# With Python
python -m http.server 5500

# With Node.js (npx)
npx serve .

# With VS Code — install "Live Server" extension and click "Go Live"
```

Then open `http://localhost:5500` in your browser.

For production, deploy to **Netlify**, **Vercel**, or **GitHub Pages** (all free). Make sure to add your production domain to Supabase → **Authentication → URL Configuration → Site URL**.
