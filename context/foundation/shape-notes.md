---
project: "MonRate"
context_type: greenfield
created: 2026-05-18
updated: 2026-05-18
product_type: web-app
target_scale:
  users: medium
timeline_budget:
  mvp_weeks: 4
  hard_deadline: null
  after_hours_only: true
checkpoint:
  current_phase: 8
  phases_completed: [1, 2, 3, 4, 5, 6, 7]
  frs_drafted: 8
  quality_check_status: accepted
  gray_areas_resolved:
    - topic: "pain category"
      decision: "multiple shapes — workflow friction + missing capability + decision paralysis; all three are valid entry points for the primary persona"
    - topic: "primary persona"
      decision: "casual Pokémon player building a story/single-player team; not a competitive/VGC player"
    - topic: "pain moment"
      decision: "three moments: before team-building (planning), mid-game (stuck on gym/boss), after losing (post-mortem)"
    - topic: "insight / differentiation"
      decision: "existing tools (Showdown) are built for competitive meta analysis — overwhelming for casual players who just want: am I weak to Fire? Should I swap this Pokémon?"
    - topic: "access model"
      decision: "two-tier: guest (no login, team entered fresh each session) + authenticated user (saves teams to DB, rates teams against named opponents)"
    - topic: "roles"
      decision: "flat within each tier — no admin, no premium; just guest vs. logged-in"
    - topic: "logged-in sub-features"
      decision: "both: (1) save multiple teams to database, (2) rate a saved team against a named opponent (gym leader, rival)"
    - topic: "MVP scope"
      decision: "guest analysis only in v1 — enter team, get rating, done; login + team saving + opponent matchup are v2"
    - topic: "timeline"
      decision: "3–4 weeks after-hours; user accepted slightly-over-3-week cost"
  frs_drafted: 0
  quality_check_status: pending
---

## Vision & Problem Statement

Casual Pokémon players — specifically those playing story/single-player modes — don't know if their team composition is balanced or has exploitable type weaknesses until they've already lost. Diagnosing the problem requires manually cross-referencing type charts, Bulbapedia pages, and forum posts — scattered, time-consuming, and confusing for non-experts who lack the vocabulary to ask the right questions.

The insight: existing analysis tools (e.g., Pokémon Showdown's teambuilder) are designed for competitive players who care about EV spreads, speed tiers, and meta threats. For a casual player who just wants to know "am I weak to Fire?" or "should I swap this Pokémon?", those tools produce more noise than signal. A casual-first team analyzer — one that gives a plain-language verdict, not a spreadsheet — does not exist as a focused product.

## User & Persona

**Primary persona: The Casual Trainer**

A Pokémon player engaged in story/single-player mode — could be a returning adult fan picking up a new game, a kid playing their first title, or an occasional player who doesn't follow the competitive scene. They assemble their team based on Pokémon they like or happened to catch, not from an optimized draft.

They reach for this product at one of three moments:
- **Before building**: "I want to pick a balanced team but don't know what 'balanced' means."
- **Mid-game**: "I keep losing to this gym — is my team the problem?"
- **After losing**: "What went wrong? Was it my team composition or just my strategy?"

They don't know what EV spreads are and don't care. They want a verdict and a recommendation, not raw data.

## Access Control

**Guest (unauthenticated):** Full team analyzer available without login. User enters their Pokémon each session; no data is saved. Zero friction — no account required to get value.

**Authenticated user:** Same analysis as guest, plus:
- Save multiple named teams to a server-side database.
- Rate a saved team against a named opponent (e.g., a gym leader, a rival) to see how the matchup looks.
- Standard login mechanism — sign-up / sign-in flow (method TBD at stack selection).

**Role model:** Flat within each tier. No admin, no premium, no moderation surface. Two user states: guest and logged-in.

## Success Criteria

### Primary
- A guest user enters up to 6 Pokémon by name, the app fetches their type data from PokeAPI, and displays: an overall team rating, type coverage breakdown, strong sides, weak sides, and plain-language recommendations for improvement — without requiring any login.

### Secondary
- Shareable team link: user can copy a URL that encodes their team and send it to a friend, who opens the same analysis without any account.
- Pokémon sprites shown alongside names in the team display, using images available via PokeAPI.
- Suggested replacement Pokémon named explicitly in recommendations (e.g., "add a Water-type — consider Vaporeon") rather than generic type advice.

### Guardrails
- Type matchup data must be accurate — sourced live or reliably cached from PokeAPI; wrong data destroys trust immediately.
- The guest analysis path must always be available — login must never be required to get a result.
- Analysis result delivered within a few seconds of team submission; a hung or indefinitely loading result is treated as broken.

_Timeline: 3–4 weeks after-hours. The core build is the PokeAPI integration + coverage analysis logic + recommendation engine. Secondary features add scope; primary must ship first._

## User Stories

### US-01: Guest analyzes their Pokémon team

- **Given** a guest user has opened the app
- **When** they enter the names of up to 6 Pokémon and submit their team
- **Then** they see: an overall team rating, type coverage, strong/weak sides, and plain-language improvement recommendations

#### Acceptance Criteria
- Team submission works with 1–6 Pokémon (full team of 6 not required)
- Each Pokémon name is validated against PokeAPI — invalid names surface a clear error
- All four outputs (rating, coverage, strong/weak sides, recommendations) appear together in a single view
- No login required at any point in this flow

## Functional Requirements

### Core analysis

- FR-001: Guest can search for Pokémon by name using an autocomplete input (not raw text) and add up to 6 to their team; type and base-stat data is fetched from PokeAPI per selection. Priority: must-have
  > Socrates: Counter-argument considered: "raw name input is fragile — spelling variations and regional names break it constantly." Resolution: revised FR-001 to require autocomplete/search rather than a raw text field; invalid names should surface inline errors, but the input mechanism itself should prevent most failures.

- FR-002: Guest can see an overall team rating with visible reasoning — the score or grade is accompanied by a plain-language explanation of what drove it. Priority: must-have
  > Socrates: Counter-argument considered: "opaque scores are distrusted — without seeing how the rating is calculated, it feels arbitrary." Resolution: revised FR-002 to require the reasoning to be visible alongside the score; a number without explanation is insufficient.

- FR-003: Guest can see their team's offensive type coverage — which types their team hits super-effectively and which types they fail to cover. Priority: must-have
  > Socrates: No counter-argument — stands as written. Offensive coverage is a core output alongside defensive analysis.

- FR-004: Guest can see their team's defensive weaknesses with severity weighting — the display distinguishes between one Pokémon being weak to a type vs. three Pokémon sharing that vulnerability. Priority: must-have
  > Socrates: Counter-argument considered: "aggregate weakness is misleading — flat 'weak to Fire' hides severity." Resolution: revised FR-004 to require severity weighting in the defensive weakness display (e.g., "3 of your 6 Pokémon are weak to Fire" ranks higher than a single weakness).

- FR-005: Guest can see ranked plain-language recommendations for improving the team — prioritised so the most impactful fix is listed first, not a flat list. Priority: must-have
  > Socrates: Counter-argument considered: "unranked recommendations are noisy — if there are 4 gaps, the user doesn't know which fix matters most." Resolution: revised FR-005 to require recommendations to be ranked by impact; the most severe type gap surfaces first.

### Polish

- FR-007: Guest can see official Pokémon sprite images alongside each Pokémon in their team display. Priority: nice-to-have
  > Socrates: Counter-argument considered: "sprites are aesthetic, not functional — ship without them first." Resolution: kept as nice-to-have but explicitly last-priority; the analysis must be functionally complete before polish is added.

- FR-008: Recommendations name specific Pokémon to fill identified gaps (e.g., "consider Vaporeon for a Water-type slot"), not just generic type advice. Priority: nice-to-have
  > Socrates: Counter-argument considered: "named suggestions go stale without game context — Vaporeon is useless if Eevee isn't available in this game." Resolution: kept as nice-to-have but scoped — v1 suggestions are game-agnostic common picks (widely available Pokémon), not game-specific availability recommendations. Game context is a v2 concern.

### v2 scope (explicitly out of v1 MVP)

- FR-v2-001: Authenticated user can create an account and log in.
- FR-v2-002: Authenticated user can save named teams to a server-side database.
- FR-v2-003: Authenticated user can rate a saved team against a named opponent (gym leader, rival) and see a matchup analysis.
- FR-v2-004: Guest can copy a shareable URL encoding their team for sharing. (Moved from v1 — sharing is premature before the core analysis is proven useful.)

## Business Logic

The app scores a team's type coverage completeness and defensive resilience across all member Pokémon — weighted by their base stats, adjusted for ability-modified immunities (e.g., Levitate removes Ground weakness), and grounded in the Pokémon's actual movepool rather than assumed type moves — then ranks improvement gaps by severity so the most impactful fix surfaces first.

The rule consumes four user-facing inputs per Pokémon: their type(s), their base stats (HP, Attack, Defense, Sp. Atk, Sp. Def, Speed), their ability, and their known moves. All four are fetched from PokeAPI on team submission.

The rule produces three outputs the user encounters in a single view: (1) an overall team score with visible reasoning, (2) a breakdown of offensive coverage (which types the team can hit super-effectively) and defensive weaknesses (which types threaten them, weighted by how many team members share that vulnerability), and (3) a ranked list of improvement recommendations — concrete gaps ordered by severity.

## Non-Functional Requirements

- A guest user sees the full analysis result within 3 seconds of submitting their team — this covers PokeAPI data fetching and all computation.
- The product is usable on the latest two major versions of mainstream desktop and mobile browsers (Chrome, Firefox, Safari, Edge) — no legacy browser support required.
- When PokeAPI is unavailable or slow, the product degrades gracefully — a cached or partial result is shown rather than a blank failure state.
- No data from a guest session is stored server-side — guest team inputs are processed and discarded; nothing is persisted without an authenticated account.
- The layout is responsive and usable on a phone-sized screen; no WCAG compliance target is set for v1.

## Non-Goals

- **No competitive meta analysis**: no EV spreads, IVs, speed tiers, or Smogon tier list references. The app targets casual players; competitive depth is a different product with a different persona.
- **No game-specific availability context**: recommendations in v1 are game-agnostic. The app does not know which Pokémon game the user is playing, which routes they've reached, or which Pokémon are catchable. Game context is a v2+ feature.
- **No social features in v1**: no leaderboards, no public team gallery, no sharing — the team-sharing URL (FR-006) was moved to v2 in the Socrates round. The app is a single-user analysis tool for MVP.
- **No accounts, team saving, or opponent matchup in v1**: authentication, database persistence, and gym-leader/rival matchup analysis are all explicitly v2 scope (confirmed in Phase 3).

_Product framing: web app / medium scale (dozens to a few hundred users) / after-hours / no hard deadline / mvp_weeks: 4_

## Open Questions

1. **What is the project name?** — TBD by user before or during `/10x-prd`. Block: no (PRD can be drafted with a placeholder, but the name should be set before the PRD is considered reviewed). Suggested: something functional ("TeamChecker", "PokeRate") or playful ("TypeDex", "MonTeam").

## Quality Cross-Check

All five greenfield quality elements present — status: accepted.

| Element | Status |
|---|---|
| Access Control | present — two-tier guest / authenticated model |
| Business Logic | present — one-sentence rule + inputs/outputs |
| Project artifacts | present — shape-notes.md with valid checkpoint |
| Timeline-cost ack | present — 3–4 weeks, slightly over threshold, accepted |
| Non-Goals | present — 4 explicit non-goals |

