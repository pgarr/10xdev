---
project: monrate
researched_at: 2026-05-18
recommended_platform: Cloudflare Workers (with Static Assets)
runner_up: Vercel
context_type: mvp
tech_stack:
  language: javascript/typescript
  framework: astro-6-react-19
  runtime: cloudflare-workers-workerd
---

## Recommendation

**Deploy on Cloudflare Workers with Static Assets.**

The tech stack is already wired for this platform — `@astrojs/cloudflare` adapter, `wrangler.jsonc`, and the workerd dev runtime are all in place; zero adapter changes needed. Cloudflare's free tier (100k req/day ≈ 3M/month) covers the projected MVP traffic at $0, and 300+ global PoPs satisfy the global-reach requirement and the 3-second analysis NFR without additional latency tuning. Every agent-friendly criterion passes (CLI deploy/rollback/tail, managed serverless, agent-readable docs, stable `wrangler deploy` API, full MCP suite). One adjustment is required on day one: migrate the starter config from the deprecated Cloudflare Pages target to Workers + Static Assets to avoid the `nodejs_compat` + middleware bug (#15434) and to stay on Cloudflare's active development path.

---

## Platform Comparison

| Platform | CLI-first | Managed/Serverless | Agent docs | Deploy API | MCP | Total |
|---|---|---|---|---|---|---|
| **Cloudflare Workers** | Pass | Pass | Pass | Pass | Pass | **5/5** |
| Vercel | Pass | Pass | Pass | Pass | Partial | 4.5/5 |
| Netlify | Partial | Pass | Partial | Partial | Pass | 3.5/5 |
| Render | Partial | Pass | Pass | Partial | Pass | 3.5/5 |
| Railway | Partial | Partial | Pass | Pass | Partial | 3/5 |
| Fly.io | Partial | Partial | Fail | Pass | Partial | 2.5/5 |

**Scoring notes:**

- **Cloudflare Workers** — Full wrangler CLI coverage including versioned rollback and log tailing. True serverless edge with no infrastructure to manage. Per-product `llms.txt` + `llms-full.txt` + public GitHub source. `wrangler deploy` is deterministic; `wrangler rollback [VERSION_ID]` is equally deterministic. Remote MCP suite covers 2,500+ API endpoints with a dedicated Claude Code integration page. *Partial* on nothing.

- **Vercel** — `vercel` CLI covers deploy/rollback/logs. Hobby rollback is limited to the immediately previous deployment only (Partial for CLI-first under cost constraint). Vercel MCP is public beta as of May 2026 with no new tools added in 8 months (Partial for MCP). Requires adapter swap to `@astrojs/vercel`. Active Astro 6 SSR bug #16258 needs verification at `@astrojs/vercel ≥ 10.0.4`.

- **Netlify** — No CLI rollback command; rollback is UI-only (Partial for CLI-first and Deploy API). `llms.txt` exists but no `llms-full.txt` and no public GitHub doc source (Partial for agent docs). Official MCP Server is GA since June 2025. New credit-based free tier (300 credits/month) is tight for active dev workflows.

- **Render** — Official GA MCP server and `llms.txt`/`llms-full.txt`. No CLI rollback (API + dashboard only). Free tier cold starts are ~1 minute — user-visible interstitial page; $7/month Starter eliminates this. Requires adapter swap to `@astrojs/node`.

- **Railway** — Excellent docs accessibility (`llms-full.txt`, public GitHub). No CLI rollback; image retention window on free tier is 24 hours only. MCP server labeled "work in progress" with no explicit GA. Requires adapter swap and mandatory `server.host = '0.0.0.0'` config. No hard spend cap.

- **Fly.io** — `fly.io/llms.txt` returns HTTP 404; no public doc source repo (Fail for agent docs). No dedicated rollback command — requires manual image redeployment. `fly mcp-server` labeled experimental. Free tier eliminated in 2024. Requires adapter swap.

---

### Shortlisted Platforms

#### 1. Cloudflare Workers (Recommended)

Already the target runtime of this stack — `@astrojs/cloudflare` v13.5.1 is GA with Astro 6 first-class support, and the workerd dev runtime now matches production exactly. The free tier (100k req/day) covers MonRate's MVP traffic at $0 given the low-QPS, medium-users profile from the PRD. True 300+ PoP edge deployment satisfies global latency requirements without configuration. The only action before first deploy: migrate from the starter's Cloudflare Pages target to Workers + Static Assets to avoid the open middleware bug and stay on Cloudflare's active development path.

#### 2. Vercel

Strong second: full CLI tooling, `llms-full.txt`, excellent Astro ecosystem support, and a true global edge network. Falls short of Cloudflare on two axes: requires an adapter swap (adding friction to an already-configured stack), and the Hobby plan's non-commercial restriction means any real production traffic requires Pro at $20/month. The Vercel MCP is public beta with no GA timeline announced. A solid fallback if Cloudflare Workers migration complexity proves higher than expected.

#### 3. Netlify

GA MCP server (the best MCP story in the shortlist aside from Cloudflare), Astro 6 day-one support confirmed, and a first-party Neon Postgres integration if v2 data layer needs arise. Loses points for the credit-based free tier (hard cap at 300 credits/month, easily consumed by ~10 deploys + moderate traffic) and the absence of CLI rollback. Viable for $20/month Pro; less comfortable as a zero-cost MVP option.

---

## Anti-Bias Cross-Check: Cloudflare Workers

### Devil's Advocate — Weaknesses

1. **Bug #15434: `nodejs_compat` + middleware renders SSR as `[object Object]` in Pages deployments.** This project has `src/middleware.ts` running on every request. Any npm package requiring `nodejs_compat` combined with active middleware will hit this bug in Cloudflare Pages. Mitigation: migrate from Pages to Workers + Static Assets before adding any `nodejs_compat`-requiring package.

2. **Cloudflare Pages is deprecated (April 2025).** The 10x Astro Starter targets Pages via `wrangler.jsonc`. No forced migration deadline, but all new platform features (Secrets Store, Workflows, Containers, MCP) ship Workers-only. Starting on a deprecated product means migration work is guaranteed — better done at project start than mid-development.

3. **Free tier 10ms CPU/request limit.** The limit applies to CPU time, not wall-clock time — `fetch()` calls to PokeAPI don't count. But synchronous type matrix computation (cross-referencing 6 Pokémon × 18 types × abilities × moves) consumes CPU. A naive O(n²) implementation can approach the limit. The paid plan ($5/month) removes this constraint, but it's an unexpected cost if not planned for.

4. **`Astro.locals.runtime` removed in `@astrojs/cloudflare` v13+.** Code or documentation examples using this accessor will fail at runtime. The project's `astro:env/server` pattern (SUPABASE_URL/SUPABASE_KEY) is correct and unaffected, but any copy-pasted snippet from pre-v13 docs is silently wrong.

5. **Preview deployments are public without protection by default.** Every PR branch creates a public `*.pages.dev` URL. Fine for v1 (no auth). When v2 adds Supabase Auth, OAuth redirect URIs on preview deployments will be reachable by anyone with the URL. Set up Cloudflare Access on the preview subdomain before implementing auth.

### Pre-Mortem — How This Could Fail

The MonRate deploy went perfectly on day one — `wrangler deploy`, zero configuration changes, production URL live in 30 seconds. First few weeks of development were smooth. Then the PokeAPI layer got refactored to use a richer data library that internally required `node:events`, which meant adding `nodejs_compat` to `wrangler.jsonc`. That same week, a logging middleware was added to `src/middleware.ts`. The combination triggered bug #15434: every SSR page started returning the literal string `[object Object]`. The error gave no stack trace and React DevTools showed correct props on the client side, making it appear to be a hydration issue rather than a server rendering problem. Three days of debugging followed before the root cause was identified from a GitHub issue comment.

The fix was migrating from Cloudflare Pages to Workers with Static Assets — a well-documented migration, but one that took two days of Wrangler config work and required understanding the difference between Pages Functions and Workers entrypoints. After the migration, the app was stable. Four months later, at higher organic traffic, the free tier's 10ms CPU limit started occasionally triggering for users submitting full 6-Pokémon teams with complex ability parsing. The billing alert had not been configured, so the upgrade to the $5/month paid plan came as a surprise invoice rather than a planned cost.

The core failure was not starting on Workers + Static Assets from day one, and not testing CPU budget against the full analysis workload before launch.

### Unknown Unknowns

- **The starter's `wrangler.jsonc` targets Cloudflare Pages.** Pages is deprecated. Migrate to Workers + Static Assets config on day one — `wrangler.jsonc` changes from `pages_build_output_dir` to `assets.directory`. This is a 10-minute change that avoids all Pages-specific bugs and keeps the project on Cloudflare's active development path.

- **`npm run dev` may invoke `wrangler pages dev` (legacy).** Verify the `dev` script in `package.json` — it should invoke `wrangler dev` (Workers) not `wrangler pages dev` (Pages). With `@astrojs/cloudflare` v12+, `astro dev` runs on the actual workerd runtime. Make sure the dev script is aligned with the target deployment model.

- **CPU time ≠ wall-clock time.** The 10ms free tier limit counts only active CPU execution. Async I/O (PokeAPI fetches) is free. Measure CPU usage in the first iteration of the analysis engine — if synchronous type chart logic approaches 8ms, refactor to async or upgrade the plan.

- **CJS-only npm packages hard-fail on workerd.** Run new dependencies through `publint` or check for `exports` in `package.json` before installing. Packages shipping only CommonJS `require()` without an ESM export will fail at build time or runtime. This is particularly relevant for data-processing and utility libraries.

- **Preview deployments are public.** `*.pages.dev` URLs are reachable by anyone with the link, no auth wall by default. Fine for v1. Before implementing Supabase Auth in v2, add Cloudflare Access to the preview subdomain to prevent OAuth callback URLs from being publicly accessible.

---

## Operational Story

- **Preview deploys**: Each branch pushed to GitHub triggers an automatic Cloudflare Pages preview deploy (or Workers deploy if migrated) with a unique `*.pages.dev` URL. Preview URLs are public by default — no Cloudflare Access protection. Do not use preview URLs as Supabase OAuth redirect targets without first adding Access protection.

- **Secrets**: `SUPABASE_URL` and `SUPABASE_KEY` are set via `wrangler secret put SUPABASE_URL` and `wrangler secret put SUPABASE_KEY`, or through the Cloudflare dashboard → Workers & Pages → Settings → Variables. Secrets are encrypted at rest, readable only by the Worker at runtime, and never exposed in `wrangler.jsonc`. Rotation: `wrangler secret put <NAME>` overwrites the old value; a new deploy is required to pick up the change. For preview environments, use `--env preview` flag.

- **Rollback**: `wrangler rollback` (omit VERSION_ID to roll back to previous version) — takes effect immediately, creates a new deployment entry in the version history. For a specific version: `wrangler rollback <VERSION_ID>`. Typical time-to-revert: under 30 seconds globally. Important: rollback restores code only — it does not reverse database migrations. If Supabase migrations were applied between versions, rollback requires a manual migration reversal.

- **Approval**: Production deploy (`wrangler deploy`) can be run unattended by an agent. Rollback (`wrangler rollback`) can be run unattended by an agent. Rotating secrets (`wrangler secret put`) requires human approval — the secret value must come from a secure store, not be composed by the agent. Deleting a Workers project, changing custom domains, or modifying billing require human action in the Cloudflare dashboard.

- **Logs**: `wrangler tail [WORKER] --format json` streams live logs to stdout — filterable by `--status`, `--method`, `--search`, `--version-id`, `--sampling-rate`. Historical logs are not retained by default on the free plan; upgrade to Workers Logs (paid) for persistent log storage. The Cloudflare MCP server (`mcp.cloudflare.com/workers`) exposes log querying as a structured tool call without requiring CLI output parsing.

---

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| `nodejs_compat` + middleware → SSR renders as `[object Object]` (bug #15434) | Devil's advocate | H | H | Migrate from Pages to Workers + Static Assets before adding any `nodejs_compat`-requiring package. Never combine `nodejs_compat` flag with active middleware in a Pages deployment. |
| Cloudflare Pages deprecated — new features ship Workers-only | Devil's advocate | H | M | Migrate `wrangler.jsonc` to Workers + Static Assets config on day one. 10-minute change; documented migration guide available. |
| Free tier 10ms CPU/req limit hit by complex type analysis | Devil's advocate | M | M | Measure CPU time on first full 6-Pokémon analysis. Use async where possible. Budget $5/month paid plan as a known potential cost. |
| CJS-only npm package fails silently or at build on workerd | Pre-mortem | M | M | Run `publint` on new dependencies before merging. Prefer packages with explicit ESM `exports` in `package.json`. |
| Preview deployments expose OAuth callbacks in v2 | Unknown unknowns | L | H | Add Cloudflare Access to `*.monrate.pages.dev` (or Workers preview subdomain) before implementing Supabase Auth. |
| `wrangler pages dev` vs `wrangler dev` — dev/prod runtime mismatch | Unknown unknowns | M | M | Verify `package.json` `dev` script calls `wrangler dev`, not `wrangler pages dev`. Run `wrangler dev` for all local SSR testing. |
| Rollback does not reverse Supabase migrations | Research finding | L | H | Write backwards-compatible migrations. Test rollback scenario in staging before production. Keep migration and deploy steps loosely coupled. |

---

## Getting Started

1. **Migrate from Cloudflare Pages to Workers + Static Assets.** Update `wrangler.jsonc`: replace `pages_build_output_dir = "dist"` with the Workers Static Assets config (`assets.directory = "dist"`). This eliminates the deprecated Pages product and the `nodejs_compat` + middleware bug class. Reference: [Migrate from Pages to Workers](https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/).

2. **Verify the dev script.** Check that `package.json` `dev` script invokes `wrangler dev` (Workers runtime), not `wrangler pages dev` (legacy Pages). With `@astrojs/cloudflare` v12+, `astro dev` runs on workerd — ensure the local dev loop uses the same runtime as production.

3. **Set secrets via wrangler.** Run `wrangler secret put SUPABASE_URL` and `wrangler secret put SUPABASE_KEY` — enter the values from `.dev.vars`. These are stored encrypted in Cloudflare's secret store and injected at runtime; they must not appear in `wrangler.jsonc` or be committed to git.

4. **First deploy.** Run `wrangler deploy` from the project root. Wrangler reads `wrangler.jsonc`, builds the Astro project (`npm run build`), and publishes to your Workers account. The command returns the production URL on success. Verify the URL renders the app correctly.

5. **Verify CPU budget.** After implementing the core type analysis logic, test with a full 6-Pokémon team and measure CPU time. Use `wrangler tail --format json` and observe `cpuTime` in the log output. If CPU approaches 8ms, review the analysis algorithm for synchronous loops and consider memoizing the type chart matrix.

---

## Out of Scope

The following were not evaluated in this research:
- Docker image configuration
- CI/CD pipeline setup (GitHub Actions auto-deploy-on-merge is the assumed default per `tech-stack.md`)
- Production-scale architecture (multi-region, HA, DR)
