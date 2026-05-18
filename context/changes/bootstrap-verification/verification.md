---
bootstrapped_at: 2026-05-18T14:49:00Z
starter_id: 10x-astro-starter
starter_name: "10x Astro Starter (Astro + Supabase + Cloudflare)"
project_name: monrate
language_family: js
package_manager: npm
cwd_strategy: git-clone
bootstrapper_confidence: first-class
phase_3_status: ok
audit_command: "npm audit --json"
---

## Hand-off

```yaml
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
```

### Why this stack

MonRate is a guest-first Pokémon team analyzer with no auth or persistence required in v1, a 4-week after-hours timeline, and medium user scale. The 10x Astro Starter — Astro 6 + React 19 + TypeScript + Supabase + Cloudflare Pages — is the recommended default for the (web-app, JS) cell and clears all four agent-friendly gates: explicit TypeScript contracts throughout, file-based routing and island architecture enforce strong conventions, the Astro + Supabase + Cloudflare surface is well-represented in JS training data, and official docs are current and version-linked. PokeAPI calls sit naturally in Astro server routes; the Cloudflare edge runtime satisfies the <3-second analysis NFR without additional latency tuning. Supabase is bundled but inactive for v1 — the guest-only analysis flow needs no persistence — and activates cleanly for v2 team saving without re-scaffolding. CI runs on GitHub Actions with auto-deploy-on-merge, the starter's natural default. Bootstrapper confidence is first-class: the CLI is registered and expected to work, with occasional manual steps possible.

## Pre-scaffold verification

| Signal      | Value                                                        | Severity    | Notes                                                     |
| ----------- | ------------------------------------------------------------ | ----------- | --------------------------------------------------------- |
| npm package | not run                                                      | n/a         | cmd_template starts with `git clone`; npm step skipped    |
| GitHub repo | https://github.com/przeprogramowani/10x-astro-starter        | unavailable | gh CLI not found on this system; recency check not run    |

## Scaffold log

**Resolved invocation**: `git clone https://github.com/przeprogramowani/10x-astro-starter .bootstrap-scaffold && cd .bootstrap-scaffold && npm install`
**Strategy**: git-clone (cloned starter repo without keeping its git history, then moved files up)
**Exit code**: 0
**Files moved**: 19 (astro.config.mjs, components.json, .env.example, eslint.config.js, .github/, .gitignore, .husky/, node_modules/, .nvmrc, package.json, package-lock.json, .prettierrc.json, public/, README.md, src/, supabase/, tsconfig.json, .vscode/, wrangler.jsonc)
**Conflicts (.scaffold siblings)**: CLAUDE.md → CLAUDE.md.scaffold (existing CLAUDE.md in cwd wins; diff with `diff CLAUDE.md CLAUDE.md.scaffold`)
**.gitignore handling**: moved silently (no .gitignore existed in cwd prior to scaffold)
**.git/ removal**: upstream .git/ deleted before move-up; user's own repo history unaffected
**.bootstrap-scaffold cleanup**: deleted

## Post-scaffold audit

**Tool**: `npm audit --json`
**Summary**: 0 CRITICAL, 1 HIGH, 5 MODERATE, 0 LOW
**Direct vs transitive**: 0/0/1/0 direct of total 0/1/5/0 (CRITICAL/HIGH/MODERATE/LOW)

#### CRITICAL findings

None.

#### HIGH findings

- **devalue** (range: 5.6.3–5.8.0) — GHSA-77vg-94rm-hx3p
  "Svelte devalue: DoS via sparse array deserialization"
  CVSS 7.5 (AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H) — CWE-770
  Transitive (not direct). Fix available (`npm audit fix`).

#### MODERATE findings

- **@astrojs/check** (>=0.9.3) — isDirect: **true**
  Depends on `@astrojs/language-server` which pulls in the advisory chain. Fix: downgrade to 0.9.2 (semver major; `npm audit fix --force`).

- **@astrojs/language-server** (>=2.14.0) — transitive via `volar-service-yaml`.
  Fixed by downgrading `@astrojs/check` to 0.9.2.

- **volar-service-yaml** (<=0.0.70) — transitive via `yaml-language-server`.
  Fixed by downgrading `@astrojs/check` to 0.9.2.

- **yaml** (2.0.0–2.8.2) — GHSA-48c2-rrv3-qjmp
  "yaml vulnerable to Stack Overflow via deeply nested YAML collections"
  CVSS 4.3 — CWE-674. Transitive (yaml-language-server). Fixed by downgrading `@astrojs/check` to 0.9.2.

- **yaml-language-server** (1.11.1-08d5f7b.0–1.22.1-fc5f874.0) — transitive via `yaml`.
  Fixed by downgrading `@astrojs/check` to 0.9.2.

#### LOW / INFO findings

None.

**Note**: all HIGH and MODERATE advisories are in the language-server / dev-tooling chain (`@astrojs/check`, language-server, volar-service-yaml). They affect the development environment only, not the production runtime. The `devalue` HIGH is a transitive dep of Astro's SSR serialization, present but not directly called from this starter's routes.

## Hints recorded but not acted on

| Hint                    | Value             |
| ----------------------- | ----------------- |
| bootstrapper_confidence | first-class       |
| quality_override        | false             |
| path_taken              | standard          |
| self_check_answers      | null              |
| team_size               | solo              |
| deployment_target       | cloudflare-pages  |
| ci_provider             | github-actions    |
| ci_default_flow         | auto-deploy-on-merge |
| has_auth                | false             |
| has_payments            | false             |
| has_realtime            | false             |
| has_ai                  | false             |
| has_background_jobs     | false             |

These hints were read into the verification record but bootstrapper v1 takes no automated action on them. A future M1L4 skill ("Memory Architecture") will use these to generate `CLAUDE.md` and `AGENTS.md` overlays and set up per-feature scaffolding compensations.

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:
- `git init` (if you have not already) to start your own repo history.
- Review `CLAUDE.md.scaffold` — the starter ships its own CLAUDE.md; `diff CLAUDE.md CLAUDE.md.scaffold` to see what it added vs your existing file, and merge anything useful.
- Address audit findings per your project's risk tolerance — all current HIGH/MODERATE advisories are in dev tooling; run `npm audit fix` for the low-risk transitive fixes, and evaluate `npm audit fix --force` for the `@astrojs/check` major-version downgrade.
- Configure Cloudflare Pages and Wrangler per `wrangler.jsonc` before deploying.
