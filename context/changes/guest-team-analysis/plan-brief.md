# MonRate: Guest Team Analysis — Plan Brief

> Full plan: `context/changes/guest-team-analysis/plan.md`

## What & Why

Build the complete v1 guest Pokémon team analyzer: autocomplete search, PokeAPI data fetching cached at the Cloudflare edge, a two-component scoring engine, and a single-view results panel — all without login. S-01 is the north star slice: the only user story and the product's entire reason to exist. Nothing downstream matters until this works and casual players prove they prefer a plain-language verdict over raw type data.

## Starting Point

The 10x Astro Starter scaffold is in place — Astro 6, React 19, TypeScript, Tailwind CSS, Cloudflare Workers deployed live. Auth endpoints and middleware exist but are intentionally inactive for v1. The main page shows a starter template (`Welcome.astro`). No PokeAPI integration, no analysis logic, no custom types, no team UI.

## Desired End State

A user visits `/`, types a Pokémon name into an autocomplete field, selects up to 6 Pokémon (each slot immediately shows the Pokémon's type badges), clicks "Analyze", and sees — inline, without a page reload — an overall score (e.g. **78 / 100 — A**), offensive type coverage breakdown, a severity-weighted defensive weakness table, and a ranked plain-language recommendation list. The full flow completes in under 3 seconds. If any Pokémon's data failed to load, that slot shows an inline error and is excluded from the analysis with an explanatory note.

## Key Decisions Made

| Decision | Choice | Why (1 sentence) | Source |
|---|---|---|---|
| Rating format | Numeric 0–100 + letter grade (S/A/B/C/D/F) | Combines continuous precision with an instant categorical read | Plan |
| Scoring formula | Two-component: offense score (% of 18 types covered) + defense score (penalty per shared weakness × member count × stat weight) | Matches PRD §Business Logic exactly — visible reasoning, stat-weighted, separate offensive/defensive breakdown | Plan |
| Move-type resolution | Fetch all 18 type endpoints (each lists its moves) → build reverse map; no per-move API calls | 18 cached calls vs 100+ per Pokémon — the only approach that satisfies both "actual movepool" PRD requirement and the <3s NFR | Plan |
| Autocomplete source | Pre-load full ~1000 name list on mount, filter client-side | Zero per-keystroke PokeAPI exposure; top_blocker is external (PokeAPI no SLA) | Plan |
| Fetch timing | Per Pokémon selection (not batch on submit) | Distributes load; user sees type info building up in real time | Plan |
| Ability immunities | Hardcoded list of 12 mechanical immunity abilities | Covers 95% of real cases with zero extra API calls | Plan |
| Graceful degradation | Partial result + inline error per failed Pokémon | Matches PRD NFR exactly; showing nothing contradicts the spec | Plan |
| Page structure | Replace `/` with analyzer (delete Welcome.astro) | The analyzer is the product; a landing page adds a click with no value at MVP | Plan |
| Severity ranking | Weakness count × member count × inverse defensive stat | Matches FR-004 (severity weighting) and FR-005 (most impactful first) | Plan |
| Testing scope | Unit tests for analysis engine only (Node built-in test runner) | Trust-critical logic that's pure data-in/data-out; no test framework overhead | Plan |

## Scope

**In scope:** Autocomplete search (FR-001), team rating with visible reasoning (FR-002), offensive coverage (FR-003), defensive weaknesses with severity weighting (FR-004), ranked recommendations (FR-005), graceful PokeAPI degradation, mobile-responsive layout.

**Out of scope:** Sprites (FR-007), named replacement suggestions (FR-008), shareable URL (FR-v2-004), auth, team saving, opponent matchup, competitive meta analysis, game-specific availability.

## Architecture / Approach

```
Browser                   Cloudflare Worker              PokeAPI
  │                             │                           │
  │── GET /api/pokemon-list ───►│── fetch + CF Cache ──────►│
  │◄─ [{name, url}, ...] ───────│                           │
  │                             │                           │
  │── GET /api/pokemon/{name} ─►│── fetch pokemon ─────────►│
  │                             │── fetch 18 type endpoints►│ (cached)
  │                             │   build moveTypeMap        │
  │                             │   compute coverage/weaknesses
  │◄─ TeamMember JSON ──────────│                           │
  │                             │
  │  [user clicks Analyze]
  │  analyzeTeam(team[]) — runs client-side (pure TS, no network)
  │  render AnalysisPanel
```

Key files: `src/types.ts` (shared types), `src/lib/services/pokeapi.ts` (PokeAPI + CF Cache), `src/lib/services/analysis.ts` (scoring engine), `src/pages/api/pokemon-list.ts`, `src/pages/api/pokemon/[name].ts`, `src/components/TeamAnalyzer.tsx` (root), plus sub-components `PokemonSearch`, `TeamSlot`, `TypeBadge`, `AnalysisPanel`.

## Phases at a Glance

| Phase | What it delivers | Key risk |
|---|---|---|
| 1. PokeAPI Integration Layer | Cached server endpoints; `GET /api/pokemon/pikachu` returns pre-processed TeamMember JSON | Move-type resolution depends on all 18 type endpoints returning valid `moves` arrays — verify this against PokeAPI before Phase 2 |
| 2. Analysis Engine | Pure scoring functions + unit tests pass | Scoring constants (penalty factor `8`, reference stat `75`) may need calibration; manual sanity-check with a known team |
| 3. Analyzer UI | Full end-to-end flow in browser; main page replaced | shadcn `command` component integration; CF Cache warmup time on first real use |

**Prerequisites:** None — deploy/infra already live, no DB or auth needed.  
**Estimated effort:** ~3–4 focused sessions across 3 phases.

## Open Risks & Assumptions

- PokeAPI type endpoints (`/api/v2/type/{name}`) are assumed to include a complete `moves` array — verify in Phase 1 before the analysis engine depends on it.
- Scoring constants (`8` penalty factor, `75` reference stat, grade boundaries) are calibrated estimates; a sanity-check with a known team is required in Phase 2 before shipping.
- Cloudflare CF Cache API may behave differently in local `wrangler dev` vs. production — cold-cache <3s testing should be done against a deployed preview, not local.
- `--experimental-strip-types` for Node 22 TypeScript tests requires no path-alias resolution in test files; analysis.ts imports must use relative paths.

## Success Criteria (Summary)

- A guest user can enter up to 6 Pokémon, see type info per slot, click Analyze, and receive all four output sections in a single view — no login, no page reload.
- The analysis result for a Charizard-containing team correctly shows Rock as a vulnerability and NOT Ground (Flying-type immunity in the type chart).
- Full analysis completes < 3 seconds on a cold Cloudflare cache.
