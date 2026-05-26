---
project: "MonRate"
version: 1
status: draft
created: 2026-05-26
updated: 2026-05-26
prd_version: 1
main_goal: market-feedback
top_blocker: external
---

# Roadmap: MonRate

> Derived from `context/foundation/prd.md` (v1) + auto-researched codebase baseline.
> Edit-in-place; archive when superseded.
> Slices below are listed in dependency order. The "At a glance" table is the index.

## Vision recap

MonRate is a casual-first Pokémon team analyzer: enter up to 6 Pokémon, get a plain-language verdict on your team's strengths and weaknesses — no login, no EV spreads, no competitive jargon. The gap it fills: tools like Pokémon Showdown are built for competitive players, not the casual trainer who just wants to know "am I weak to Fire?" or "should I swap this Pokémon?" MonRate targets three moments — before building a team, mid-game (stuck on a gym), and after a loss — and delivers a verdict and ranked recommendation, not raw data.

## North star

**S-01: Guest team analysis** — the smallest end-to-end slice whose successful delivery would prove the core product hypothesis — the belief that casual players prefer a plain-language verdict over raw type data — placed as early as prerequisites allow because everything else only matters if this works. S-01 is both the only user story and the primary Success Criterion; the core flow ships as a single slice because a partial result (rating without coverage, or coverage without recommendations) does not prove the hypothesis.

> "North star" here means the single slice that, if it ships and works, confirms the product's reason to exist — not just the first item on the list, but the one whose success or failure changes everything downstream.

## At a glance

| ID   | Change ID           | Outcome (user can …)                                                                                                            | Prerequisites | PRD refs                                      | Status |
| ---- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------- | --------------------------------------------- | ------ |
| S-01 | guest-team-analysis | enter up to 6 Pokémon (autocomplete), see team rating, type coverage, weighted weaknesses, and ranked recommendations — no login | —             | US-01, FR-001, FR-002, FR-003, FR-004, FR-005 | ready  |

## Baseline

What's already in place in the codebase as of 2026-05-26 (auto-researched + user-confirmed).
Foundations below assume these are present and do NOT re-scaffold them.

- **Frontend:** Partial — Astro 6 + React 19 + TypeScript + Tailwind CSS scaffold present; shadcn/ui button component; no MonRate pages, team input, or analysis UI built yet
- **Backend / API:** Absent — only starter auth endpoints (`src/pages/api/auth/`); no PokeAPI integration, no analysis logic
- **Data:** Absent — no migrations, no custom tables; Supabase present but intentionally inactive for v1 (stateless guest flow)
- **Auth:** Present — `src/middleware.ts` + sign-in/sign-up/sign-out endpoints wired; not required for v1 guest flow
- **Deploy / infra:** Present — `wrangler.jsonc` (Cloudflare Workers), GitHub Actions CI (lint + build on push/PR)
- **Observability:** Absent — no logging library, no error tracking; not required for v1

## Foundations

None — all prerequisite infrastructure layers are already in place (deploy/infra present, auth scaffold present but not needed for v1), and v1 requires no database (stateless guest-only flow). PokeAPI integration is bundled into S-01 rather than split into a separate foundation: it has a single consumer and `market-feedback` bias favors shipping the end-to-end loop, not isolating layers.

## Slices

### S-01: Guest team analysis

- **Outcome:** user can enter up to 6 Pokémon by name via autocomplete, submit their team, and see in a single view: an overall team rating with visible reasoning, offensive type coverage (which types the team hits super-effectively and which are gaps), defensive weaknesses with severity weighting (how many team members share each vulnerability), and ranked plain-language improvement recommendations prioritised by impact — no login required at any step
- **Change ID:** guest-team-analysis
- **PRD refs:** US-01, FR-001, FR-002, FR-003, FR-004, FR-005
- **Prerequisites:** —
- **Parallel with:** —
- **Blockers:** —
- **Unknowns:**
  - Scoring formula specifics: how to weight base stats and define "severity" quantitatively for the analysis engine — Owner: user. Block: no (pragmatic decision during `/10x-plan`; PRD §Business Logic provides sufficient conceptual direction).
- **Risk:** The entire product value rests on this slice's correctness — wrong type data or an unexplained rating destroys trust immediately (PRD Guardrail: "wrong data destroys trust immediately"). PokeAPI is the only external dependency with no SLA; graceful degradation (NFR: partial result or explanatory error, not a blank screen) is load-bearing, not optional. `market-feedback` bias means this slice must ship end-to-end as a unit; partial delivery (e.g., UI without scoring, or scoring without PokeAPI) does not validate the hypothesis.
- **Status:** ready

## Backlog Handoff

| Roadmap ID | Change ID           | Suggested issue title                            | Ready for `/10x-plan` | Notes                               |
| ---------- | ------------------- | ------------------------------------------------ | --------------------- | ----------------------------------- |
| S-01       | guest-team-analysis | MonRate: guest Pokémon team analyzer (core flow) | yes                   | Run `/10x-plan guest-team-analysis` |

## Open Roadmap Questions

1. **Scoring formula specifics** — How to weight base stats and define "severity" quantitatively for the analysis engine (PRD §Business Logic describes the concept but not the formula). Owner: user. Block: no — roadmap is not blocked, but `/10x-plan` should surface this as an early design decision before implementation begins.
2. **PokeAPI caching / fallback strategy** — The NFR requires graceful degradation ("partial result or explanatory error, not a blank screen") but does not specify whether responses should be cached to reduce external dependency exposure. Owner: user. Block: no — `/10x-plan` can make a pragmatic call, but a stated preference reduces re-work.

## Parked

- **Pokémon sprite images (FR-007)** — Why parked: `market-feedback` bias; sprites are aesthetic, not functional; PRD §Polish Socrates note: "analysis must be functionally complete before polish is added."
- **Named Pokémon suggestions in recommendations (FR-008)** — Why parked: nice-to-have; named suggestions require game-agnostic pick curation; PRD §Polish marks as nice-to-have with v2 game-context caveat.
- **Shareable team URL** — Why parked: moved from v1 to v2 in PRD Socrates round (FR-v2-004); "sharing is premature before the core analysis is proven useful."
- **Authenticated accounts + team saving** — Why parked: explicitly v2 scope per PRD §Non-Goals (FR-v2-001, FR-v2-002); Supabase auth scaffold is present but intentionally inactive for v1.
- **Opponent matchup analysis (gym leaders / rivals)** — Why parked: explicitly v2 scope per PRD §Non-Goals (FR-v2-003).
- **Competitive meta analysis (EV spreads, IVs, speed tiers)** — Why parked: different persona (competitive players); PRD §Non-Goals.
- **Game-specific Pokémon availability context** — Why parked: v2+ feature per PRD §Non-Goals; recommendations in v1 are game-agnostic.

## Done

(Empty on first generation. `/10x-archive` appends an entry here — and flips the matching item's `Status` to `done` — when a change whose `Change ID` matches a roadmap item is archived.)
