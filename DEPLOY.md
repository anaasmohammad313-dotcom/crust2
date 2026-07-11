# Deploying Crust POS to Vercel

This repo is already wired for Vercel — the config below just needs your
environment variables filled in on the Vercel dashboard, then a normal
`git push` / import will build and serve it correctly.

## What's already set up (no changes needed)

- `vercel.json` — build command (builds the API server + POS frontend) and
  rewrites `/api/*` to the serverless function.
- `api/index.js` — the Vercel serverless function entry point; wraps the
  same Express app used in development.
- `.vercelignore` — excludes dev-only files from the deployment.
- Session cookies automatically switch to `secure: true` + a
  Postgres-backed session store in production (`NODE_ENV=production`, set
  automatically by Vercel), so logins survive serverless cold starts.
- `package.json` pins `packageManager: pnpm@10.26.1`, so Vercel will use the
  right pnpm version automatically via corepack.

## 1. Import the project into Vercel

1. Push this repo to GitHub (if it isn't already).
2. In Vercel: **Add New → Project**, import the repo.
3. Framework preset: leave as "Other" — the `vercel.json` build command and
   output directory are already configured, no need to change anything in
   the UI.

## 2. Set environment variables in Vercel

Go to **Project → Settings → Environment Variables** and add these for the
**Production** (and Preview, if you want preview deploys to work too)
environment:

| Key | Value | Notes |
|---|---|---|
| `SESSION_SECRET` | *(your own long random string)* | Generate with `openssl rand -base64 48`. This is **not** the same value as your Replit `SESSION_SECRET` secret — generate a fresh one for production, or reuse it if you want existing Replit-issued sessions to also validate on Vercel (they won't, since session storage differs, so a fresh one is simplest). |
| `SUPABASE_PROJECT_REF` | `oaogyxzufjgesjgqcwah` | Same Supabase project already connected to Replit — safe to copy, not sensitive. |
| `SUPABASE_DB_HOST` | `aws-0-ap-southeast-2.pooler.supabase.com` | Same as above. |
| `SUPABASE_DB_PORT` | `5432` | Same as above. |
| `SUPABASE_DB_PASSWORD` | *(your Supabase DB password)* | I can't paste this for you — it's stored as a Replit **secret**, which means even I can't read its value back once saved. You already have it (from your Supabase dashboard → Project Settings → Database), paste it here directly. |

I'm intentionally not writing `SUPABASE_DB_PASSWORD` or a real `SESSION_SECRET`
into any file in this repo — hardcoding secrets into source files means
they'd be exposed to anyone with repo access (and to GitHub, if this repo is
public). Vercel's dashboard is the correct place for them; the app reads
them from `process.env` exactly like it does on Replit.

You do **not** need `DATABASE_URL` — this app prefers the four `SUPABASE_*`
vars above when they're all present (see `lib/db/src/supabase-url.ts`), so
`DATABASE_URL` can be left unset.

## 3. Deploy

Click **Deploy**. Vercel will run:

```
pnpm --filter @workspace/api-server run build && BASE_PATH=/ pnpm --filter @workspace/pos run build
```

and serve the static frontend from `artifacts/pos/dist/public`, with
`/api/*` routed to the serverless function.

## 4. Push the database schema (one-time, or after schema changes)

The Supabase database already has your real menu data — you don't need to
seed it. But if you ever change the Drizzle schema (`lib/db/src/schema`),
push it from your local machine or Replit shell (not from Vercel — this is
a one-off migration step, not part of the request-serving app):

```
pnpm --filter @workspace/db run push
```

Run this with the same `SUPABASE_*` env vars set in your shell.

## 5. Verify

After the first deploy, open the production URL and confirm:
- The login page loads.
- Logging in as `admin` works (`admin` account already exists in Supabase
  from initial setup, unless you've since changed its password).
- The Menu tab shows your real categories/items.

## Notes / gotchas

- `pnpm run typecheck` currently fails due to a pre-existing `paymentSplits`
  type mismatch unrelated to this deploy (tracked separately as a follow-up
  task) — this does **not** block the Vercel build, which only runs the
  `build` scripts, not `typecheck`.
- Menu item images are stored as base64 in Postgres (no object storage), so
  very large menus with many images will bloat your Supabase database and
  API response sizes — fine for the current image cap (2MB/item) but worth
  knowing if you scale up.
