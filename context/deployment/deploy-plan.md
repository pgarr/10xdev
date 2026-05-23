# First Deployment — Cloudflare Workers

## Context

The project (MonRate) targets Cloudflare Workers per `infrastructure.md`. The `wrangler.jsonc` was already on the Workers + Static Assets config. Worker has been renamed to `monrate`. `SUPABASE_URL`/`SUPABASE_KEY` are now `optional: false` in `astro.config.mjs`, so secrets **must** be set on the `monrate` worker before deploy or the app will fail at runtime.

Secrets were previously set on the old `10x-astro-starter` worker — they need to be re-set on `monrate`.

---

## Steps

### ✅ Step 1 — Rename Worker — DONE
`wrangler.jsonc` `name` changed from `10x-astro-starter` → `monrate`.

---

### ✅ Step 2 — Set secrets on `monrate` — DONE
Secrets must be set interactively — **do not use the `!` prefix** (it's non-interactive and sets an empty value).

Get the values from: **Supabase Dashboard → your project → Settings → API**
- `SUPABASE_URL` = Project URL (e.g. `https://xxxx.supabase.co`)
- `SUPABASE_KEY` = `anon` / `public` key

Run in your terminal:
```bash
npx wrangler secret put SUPABASE_URL   # paste Project URL when prompted
npx wrangler secret put SUPABASE_KEY   # paste anon key when prompted
```
Confirm with: `npx wrangler secret list` — both keys should appear.

---

### ✅ Step 3 — Build — DONE

### ✅ Step 4 — Deploy — DONE
Production URL: **https://monrate.garlej-p.workers.dev**
Version ID: `4f4306cf-d423-4cff-addc-ffdedd5c0621`

---

### ✅ Step 5 — Verify — DONE
Page renders correctly at https://monrate.garlej-p.workers.dev

---

### Step 6 — Clean up (optional)
Delete the now-unused `10x-astro-starter` worker from the Cloudflare dashboard.

---

## Files changed
- `wrangler.jsonc` — `name` renamed to `monrate` ✅

## Pre-conditions
- `wrangler login` completed ✅
- Worker renamed to `monrate` ✅
- `SUPABASE_URL`/`SUPABASE_KEY` are `optional: false` — secrets must be set on `monrate` before deploy ⚠️
- `wrangler.jsonc` uses Workers + Static Assets (`assets.directory`) ✅
- `package.json` `dev` uses `astro dev` ✅
