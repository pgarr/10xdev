---
starter_id: 10x-astro-starter
package_manager: npm
project_name: monrate
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-pages
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: first-class
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: false
  has_payments: false
  has_realtime: false
  has_ai: false
  has_background_jobs: false
---

## Why this stack

MonRate is a guest-first Pokémon team analyzer with no auth or persistence required in v1, a 4-week after-hours timeline, and medium user scale. The 10x Astro Starter — Astro 6 + React 19 + TypeScript + Supabase + Cloudflare Pages — is the recommended default for the (web-app, JS) cell and clears all four agent-friendly gates: explicit TypeScript contracts throughout, file-based routing and island architecture enforce strong conventions, the Astro + Supabase + Cloudflare surface is well-represented in JS training data, and official docs are current and version-linked. PokeAPI calls sit naturally in Astro server routes; the Cloudflare edge runtime satisfies the <3-second analysis NFR without additional latency tuning. Supabase is bundled but inactive for v1 — the guest-only analysis flow needs no persistence — and activates cleanly for v2 team saving without re-scaffolding. CI runs on GitHub Actions with auto-deploy-on-merge, the starter's natural default. Bootstrapper confidence is first-class: the CLI is registered and expected to work, with occasional manual steps possible.
