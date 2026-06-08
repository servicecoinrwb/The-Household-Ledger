# The Household Ledger — Deploy & Setup

Single-file family finance tracker. One `index.html`, Supabase backend, GitHub Pages host, **email + password** auth (no magic links, no Google). Same playbook as the Bookkeeper.

| | |
|---|---|
| **Repo** | `servicecoinrwb/The-Household-Ledger` (branch `main`) |
| **Live URL** | `https://servicecoinrwb.github.io/The-Household-Ledger/` |
| **Supabase project** | `tufvkuosrrymgzldlslv` |
| **Supabase URL** | `https://tufvkuosrrymgzldlslv.supabase.co` |
| **Table** | `household_finance` (one shared row, `id = 'estes'`) |
| **Auth** | Email + password (accounts created in the dashboard) |
| **Allowed users** | `brnestes@gmail.com`, `katieestes84@gmail.com` |

---

## How it works (30-second version)

The whole app state (income, bills, subs, savings, debts, settings) is one JSON blob in a single `jsonb` row keyed `estes`. Both users sign in with email + password, read/write that same row, and a realtime subscription pushes the other person's edits to your screen live. No server, no build step, no emails sent.

**Three fallback modes, same file:**
1. Supabase (signed in) → syncs across devices
2. `window.storage` (running inside Claude artifact) → local
3. `localStorage` (offline / not configured) → local

Leave `CONFIG` blank and it runs locally. Fill it in and it goes cloud.

---

## One-time setup

### 1. Database — run once in Supabase **SQL Editor**

```sql
create table if not exists household_finance (
  id text primary key,
  data jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now(),
  updated_by text
);
alter table household_finance enable row level security;

-- re-runnable: drop first so this never errors on re-run
drop policy if exists "hf read"   on household_finance;
drop policy if exists "hf insert" on household_finance;
drop policy if exists "hf update" on household_finance;

create policy "hf read"   on household_finance for select
  using (auth.jwt()->>'email' = any (array['brnestes@gmail.com','katieestes84@gmail.com']));
create policy "hf insert" on household_finance for insert
  with check (auth.jwt()->>'email' = any (array['brnestes@gmail.com','katieestes84@gmail.com']));
create policy "hf update" on household_finance for update
  using (auth.jwt()->>'email' = any (array['brnestes@gmail.com','katieestes84@gmail.com']))
  with check (auth.jwt()->>'email' = any (array['brnestes@gmail.com','katieestes84@gmail.com']));

-- realtime (ignore "already member of publication" — that's fine)
alter publication supabase_realtime add table household_finance;

-- optional: seed the row so it exists before first save
insert into household_finance (id, data, updated_by)
values ('estes', '{}'::jsonb, 'setup')
on conflict (id) do nothing;
```

**The RLS policy is the real security.** The `ALLOWED_EMAILS` array in the file is only a UI convenience.

### 2. Create the logins — Supabase → **Authentication → Users → Add user**

This is the whole auth setup. No magic links, no confirmation emails.

For each person:
- **Email:** their address (`brnestes@gmail.com`, then `katieestes84@gmail.com`)
- **Password:** the shared password you both know
- ✅ **Auto Confirm User** ← do not skip this; it's what avoids the confirmation email

Want a single shared login instead of two? Create just one user, and put only that one email in both `ALLOWED_EMAILS` and the RLS policy.

(Optional: Auth → Providers → Email — leave it enabled, defaults are fine. You do NOT need magic links or Google.)

### 3. GitHub → repo **Settings → Pages**
Source: **Deploy from a branch → `main` / `/ (root)`**. File is `index.html`, so the bare URL serves it.

### 4. Icons (already in repo root)
- `apple-touch-icon.png` (180×180) — iPhone home-screen icon, wired via `<link rel="apple-touch-icon">`
- `icon-512.png` (512×512) — for Android/PWA (only used if a manifest is added later)
- Tab favicon is an inline SVG in `index.html` — nothing to host.

Keep `apple-touch-icon.png` named exactly that, at the root.

---

## CONFIG block (top of `index.html`)

```js
const CONFIG = {
  SUPABASE_URL: 'https://tufvkuosrrymgzldlslv.supabase.co',
  SUPABASE_ANON_KEY: '<Project Settings → API → anon public key>',
  HOUSEHOLD_ID: 'estes',
  ALLOWED_EMAILS: ['brnestes@gmail.com','katieestes84@gmail.com']
};
```

- Use the **anon / public** key. **Never** the `service_role` key.
- Keep `ALLOWED_EMAILS` in sync with the SQL policy emails.

---

## Update / deploy flow

1. Edit `index.html`.
2. Commit + push to `main`.
3. Wait ~30–60s for Pages to rebuild.
4. **Hard refresh** the live page (Ctrl/Cmd + Shift + R) — Pages caches, so a normal refresh can serve the old file.

## Sign-in flow (both users)

1. Open the live URL.
2. Type email + the shared password → **Sign in** (Enter works).
3. Badge reads **"Synced across devices."** Done. No inbox, no waiting.

---

## Troubleshooting (stuff that actually bit us)

| Symptom | Cause | Fix |
|---|---|---|
| "That email isn't on the allow list" | `ALLOWED_EMAILS` has placeholders or a typo | Put the two real emails in, push, **hard refresh** |
| "Wrong email or password" | Typo, or account not created yet | Re-check; create the user in Auth → Users |
| "Confirm the user in Supabase → Auth → Users" | Forgot the **Auto Confirm User** checkbox | Open the user in the dashboard and confirm them |
| Signed in but data empty / forbidden | Email not in the **SQL policy** (≠ the file array) | Add it to the RLS policy too |
| `policy "hf read" already exists` | SQL ran before | Use the `drop policy if exists` version above |
| `already member of publication` | Realtime already on for the table | Harmless — ignore |
| Changes not showing after push | Pages cache | Hard refresh, wait a minute |

---

## Notes & gotchas

- **Last-write-wins.** If both people edit the *same* field in the same second, one overwrites the other. For two people this basically never happens — it is not true conflict merging.
- **Mortgage tab = calculator only.** It does **not** touch the budget. Enter **P&I only** there ($274.79), not the full $747.14 payment.
- **Bills tab = budget.** Put the mortgage **once** as the full payment ($747.14). That's the only place housing hits "Free each month."
- **Don't prepay the 4.125% mortgage.** Emergency fund + high-APR cards beat it every time. Leave extra-principal at 0.
- Owner labels are **Brian / Katie / Shared**; old "Me"/"Wife" data auto-remaps on load.
- Data lives only in this Supabase row + each browser's localStorage backup. No other copies.
