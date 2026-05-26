# MonRate: Guest Team Analysis — Implementation Plan

## Overview

Build the complete v1 MonRate guest Pokémon team analyzer: PokeAPI integration with Cloudflare edge caching, a two-component scoring engine (offensive coverage + defensive resilience), and a React UI that replaces the starter landing page. No login required at any step.

## Current State Analysis

The codebase is the 10x Astro Starter with only auth wired: sign-in/sign-up/signout endpoints, auth middleware, and the Supabase SSR client. The main page (`src/pages/index.astro`) renders `Welcome.astro`, a starter template component with no MonRate content. No PokeAPI integration exists, no analysis logic, no custom types, no team UI.

Infrastructure is fully in place: Cloudflare Workers adapter deployed live at `monrate.garlej-p.workers.dev`, GitHub Actions CI, `wrangler.jsonc` with `nodejs_compat` flag. Native `fetch()` is available in the Workers runtime; the CF Cache API (`caches.default`) is available but unused.

## Desired End State

A user visits `/` and immediately sees a team input: an autocomplete field that searches across all ~1000 Pokémon as they type. They select up to 6 Pokémon; each selection immediately shows the Pokémon's types in the team slot. Clicking "Analyze" runs the scoring engine against the pre-fetched data and renders — in the same view — an overall score (e.g., **78 / 100 — A**), an offensive coverage breakdown (which of 18 types the team can hit super-effectively), a defensive weakness table weighted by severity (how many team members share each vulnerability), and a ranked plain-language recommendation list. If any Pokémon's data failed to fetch, that slot shows an inline error and is excluded from the analysis with an explanatory note. The full flow completes within 3 seconds of team submission on a cold cache.

### Key Discoveries

- `src/pages/api/auth/signin.ts:1` — canonical API endpoint pattern: export `const POST: APIRoute`, parse body, return `new Response(JSON.stringify(...))` for JSON or `context.redirect()` for form posts. New endpoints follow the same shape.
- `src/middleware.ts:4` — `PROTECTED_ROUTES` array; the analyzer page lives at `/` which is not protected.
- `src/lib/supabase.ts:3` — `SUPABASE_URL`/`SUPABASE_KEY` imported from `astro:env/server`. No new env vars are needed for v1 (PokeAPI is public).
- `src/components/auth/SignInForm.tsx:43` — React form pattern: `useState` for values, `useFormStatus()` for loading state, `form method="POST"`. The analyzer diverges here: it uses `fetch()` for per-selection API calls (not form POST) and runs analysis purely in client state.
- `src/components/ui/button.tsx:1` — only shadcn component installed; `command` (combobox) must be added.
- `wrangler.jsonc` — `nodejs_compat` flag present; `caches.default` (CF Cache API) is available without any additional Workers bindings.
- `astro.config.mjs:16` — env schema only declares `SUPABASE_URL` and `SUPABASE_KEY`. No new entries needed.

## What We're NOT Doing

- No auth, no login gate, no Supabase queries (v2 scope per PRD §Non-Goals)
- No Pokémon sprite images (parked — PRD §Polish, FR-007)
- No named Pokémon replacement suggestions (parked — FR-008)
- No shareable team URL (parked — FR-v2-004)
- No persistent team storage
- No competitive meta analysis (EV spreads, IVs, speed tiers)
- No game-specific Pokémon availability filtering
- No Cloudflare KV or D1 setup (not needed for stateless v1)
- No E2E test infrastructure — unit tests for the analysis engine only

## Implementation Approach

Three phases, each fully verifiable before the next begins:

1. **PokeAPI integration layer** — server-side endpoints with CF Cache API wrapping. The key design: build a `moveTypeMap` (move name → type) by fetching all 18 PokeAPI type endpoints (each returns its full move list). This avoids one-fetch-per-move (which would be 100+ calls per Pokémon) while still satisfying PRD's "actual movepool" requirement. Each Pokémon endpoint returns pre-processed data (type list, stats, ability immunity, offensive coverage, effective weaknesses) so the client needs no further PokeAPI calls.

2. **Analysis engine** — pure TypeScript functions operating on pre-processed `TeamMember[]` data. No network calls. Two-component formula: offensive coverage score (% of 18 types covered) + defensive resilience score (penalty per shared weakness, scaled by member count and defensive stats). Unit-tested with Node's built-in test runner.

3. **Analyzer UI** — React component tree wired to the Phase 1 endpoints. Autocomplete (shadcn Command component), team slot display, and a results panel composing all four required output sections. Replaces `src/pages/index.astro` content.

## Critical Implementation Details

**CF Cache API pattern** — Cloudflare Workers expose `caches.default` for HTTP-level response caching. The codebase has no prior usage. The pattern: `const cached = await caches.default.match(req); if (cached) return cached.json();` then after fetching, `await caches.default.put(req, new Response(body, { headers: { 'Cache-Control': 'max-age=86400' } }))`. This must be used in `pokeapi.ts` for all PokeAPI outbound fetches — without it, every request re-fetches PokeAPI and the <3s NFR fails on anything beyond a trivial team.

**Move-type resolution via type endpoints** — PokeAPI's `/api/v2/type/{name}` returns a `moves` array listing every move of that type. Fetching all 18 type endpoints (in parallel, cached) builds a complete `moveTypeMap: Map<string, TypeName>` in one pass. This is the only way to resolve move types without one API call per move — the per-move endpoint approach would require 100+ calls per Pokémon selection, far exceeding the <3s budget.

**Dual-type effectiveness formula** — for a Pokémon with types `[T1, T2]`, the effective multiplier of attacking type `A` is `(chart[A][T1] ?? 1) * (chart[A][T2] ?? 1)`. The type chart only stores non-1× entries, so missing keys default to 1×. Ability immunity overrides the computed value to 0 regardless of type multipliers.

---

## Phase 1: PokeAPI Integration Layer

### Overview

Create the types, the cached PokeAPI service, and two server-side API endpoints. At the end of this phase, `curl /api/pokemon-list` returns the full Pokémon name catalogue and `curl /api/pokemon/pikachu` returns typed, pre-processed team member data. No UI yet.

### Changes Required

#### 1. Shared types

**File**: `src/types.ts` (create)

**Intent**: Define the data shapes that all three phases share. Types live here per CLAUDE.md convention so they're importable by both server-side services and client-side React components without circular dependencies.

**Contract**: Export the following named types:

```typescript
export type TypeName =
  | 'normal' | 'fire' | 'water' | 'electric' | 'grass' | 'ice'
  | 'fighting' | 'poison' | 'ground' | 'flying' | 'psychic' | 'bug'
  | 'rock' | 'ghost' | 'dragon' | 'dark' | 'steel' | 'fairy';

export const ALL_TYPES: TypeName[] = [
  'normal','fire','water','electric','grass','ice','fighting','poison',
  'ground','flying','psychic','bug','rock','ghost','dragon','dark','steel','fairy'
];

export interface PokemonStats {
  hp: number; attack: number; defense: number;
  specialAttack: number; specialDefense: number; speed: number;
}

export interface TeamMember {
  name: string;                       // lowercase, as returned by PokeAPI
  displayName: string;                // capitalized for display
  types: TypeName[];                  // 1 or 2 types
  stats: PokemonStats;
  abilityImmunity: TypeName | null;   // type this ability nullifies (e.g. 'ground' for Levitate)
  offensiveCoverage: TypeName[];      // types this member can hit ≥2× via full learnset
  effectiveWeaknesses: { type: TypeName; multiplier: 2 | 4 }[];
  fetchError: boolean;
}

// TypeChart[attackingType][defendingType] → effectiveness multiplier (omitted = 1×)
export type TypeEffectiveness = Partial<Record<TypeName, 0 | 0.5 | 2>>;
export type TypeChart = Record<TypeName, TypeEffectiveness>;

export interface WeaknessEntry {
  attackingType: TypeName;
  affectedCount: number;
  affectedNames: string[];
  avgRelevantDefStat: number;
  hasQuadWeak: boolean;       // any member has 4× to this type
}

export interface CoverageResult {
  covered: TypeName[];
  gaps: TypeName[];
}

export interface ScoreResult {
  offenseScore: number;   // 0–100
  defenseScore: number;   // 0–100
  overall: number;        // 0–100
  grade: 'S' | 'A' | 'B' | 'C' | 'D' | 'F';
}

export interface Recommendation {
  text: string;
  category: 'defense' | 'offense';
  relatedType: TypeName;
  severity: 'critical' | 'moderate' | 'minor';
}

export interface AnalysisResult {
  score: ScoreResult;
  coverage: CoverageResult;
  weaknesses: WeaknessEntry[];
  recommendations: Recommendation[];
  teamSize: number;
  failedMembers: string[];
}
```

---

#### 2. PokeAPI service

**File**: `src/lib/services/pokeapi.ts` (create)

**Intent**: Centralise all PokeAPI access behind a CF Cache API wrapper. Called only from server-side API endpoints — never imported by client-side React. Exports two public functions: `buildTypeData` and `buildTeamMember`.

**Contract**: Export the following:

`fetchCached(url: string): Promise<unknown>`
— wraps `fetch(url)` with `caches.default`. Cache key = `new Request(url)`. On miss: fetch → store response clone with `Cache-Control: max-age=86400` → return parsed JSON. On hit: return parsed JSON from cached response.

`buildTypeData(): Promise<{ typeChart: TypeChart; moveTypeMap: Map<string, TypeName> }>`
— fetches all 18 type endpoints (`https://pokeapi.co/api/v2/type/{name}`) in parallel via `Promise.all`. From each response, extract `damage_relations` to populate `typeChart[typeName]` (mapping each defending type to its effectiveness multiplier — use `double_damage_to`, `half_damage_to`, `no_damage_to` fields). Also extract the `moves` array from each type response to build `moveTypeMap[moveName] = typeName`. All 18 fetches go through `fetchCached`.

`buildTeamMember(name: string): Promise<TeamMember>`
— fetches `https://pokeapi.co/api/v2/pokemon/${name}` via `fetchCached`. Extracts:
- `types`: array of type names from `pokemon.types[].type.name`
- `stats`: map `base_stat` values to `PokemonStats` fields (stat names: `hp`, `attack`, `defense`, `special-attack` → `specialAttack`, `special-defense` → `specialDefense`, `speed`)
- `abilityName`: `pokemon.abilities.find(a => !a.is_hidden)?.ability.name` (primary non-hidden ability)
- `moveNames`: `pokemon.moves.map(m => m.move.name)`

Then calls `buildTypeData()` to get `typeChart` and `moveTypeMap`. Uses them to:
- Compute `abilityImmunity`: check `abilityName` against the hardcoded immunity table below
- Compute `offensiveCoverage`: for each move in `moveNames`, look up its type in `moveTypeMap`; collect the set of unique move types; for each move type, find which defending types it hits ≥2× (using `typeChart[moveType]` entries where value ≥ 2); union all covered defending types
- Compute `effectiveWeaknesses`: for each of the 18 attacking types, compute effective multiplier against this Pokémon's type combo (dual-type formula: `(chart[A][T1] ?? 1) * (chart[A][T2] ?? 1)`); if abilityImmunity matches the attacking type, set multiplier to 0; collect types where multiplier ≥ 2

Returns assembled `TeamMember` with `fetchError: false`. On any fetch failure, return a minimal `TeamMember` with `fetchError: true` (name populated, all arrays empty).

**Hardcoded ability immunity table** (ability name → immune type):

| Ability name      | Immune to    |
|-------------------|--------------|
| levitate          | ground       |
| flash-fire        | fire         |
| water-absorb      | water        |
| dry-skin          | water        |
| storm-drain       | water        |
| volt-absorb       | electric     |
| lightning-rod     | electric     |
| motor-drive       | electric     |
| sap-sipper        | grass        |
| earth-eater       | ground       |
| well-baked-body   | fire         |
| wind-rider        | flying       |

---

#### 3. Pokémon name list endpoint

**File**: `src/pages/api/pokemon-list.ts` (create)

**Intent**: Provide the full Pokémon name catalogue for the client-side autocomplete. Runs on the server so the PokeAPI call goes through CF Cache instead of the user's browser.

**Contract**: `export const GET: APIRoute`. Calls `fetchCached('https://pokeapi.co/api/v2/pokemon?limit=2000')`. Returns `new Response(JSON.stringify(data), { headers: { 'Content-Type': 'application/json' } })`. On fetch error, returns `new Response(JSON.stringify({ error: 'Failed to load Pokémon list' }), { status: 503 })`.

---

#### 4. Per-Pokémon data endpoint

**File**: `src/pages/api/pokemon/[name].ts` (create)

**Intent**: Return pre-processed `TeamMember` data for one Pokémon. Called per selection from the autocomplete; the client stores the result in team state and never calls PokeAPI directly.

**Contract**: `export const GET: APIRoute`. Extract `name = context.params.name` (lowercase). Call `buildTeamMember(name)` from `@/lib/services/pokeapi`. If the returned `TeamMember` has `fetchError: true`, return `new Response(JSON.stringify({ error: 'Could not fetch data for ${name}' }), { status: 503, headers: { 'Content-Type': 'application/json' } })`. On success, return the serialised `TeamMember` with status 200. Return 400 if `name` is missing or not a string.

---

### Success Criteria

#### Automated Verification

- `npm run lint` passes (no TypeScript errors)
- `GET /api/pokemon-list` returns JSON with `results` array of Pokémon names (verify via `curl http://localhost:4321/api/pokemon-list | jq '.results | length'` — should be ~1000)
- `GET /api/pokemon/pikachu` returns JSON with `name: 'pikachu'`, `types: ['electric']`, non-empty `offensiveCoverage`, and `fetchError: false`
- `GET /api/pokemon/notarealmon` returns status 503

#### Manual Verification

- Confirm the pikachu response includes Levitate-bearing Pokémon correctly: `GET /api/pokemon/gengar` should have `abilityImmunity: null` (Gengar's primary ability is Cursed Body from Gen 6+) or `abilityImmunity: 'normal'` — verify against PokeAPI directly
- Confirm a dual-type Pokémon's effective weaknesses are correct: `GET /api/pokemon/charizard` should show `effectiveWeaknesses` containing `rock` (4×), `water` (2×), `electric` (2×) — NOT ground (Flying immunity nullifies it)
- Confirm `offensiveCoverage` is non-trivial for a Pokémon with a wide learnset (e.g. `/api/pokemon/smeargle` which can learn any move — should cover most or all 18 types)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Analysis Engine

### Overview

Implement the pure scoring logic as a single service module and verify it with unit tests. No network calls, no UI. Takes `TeamMember[]` (from Phase 1) as input and returns `AnalysisResult`.

### Changes Required

#### 5. Analysis engine

**File**: `src/lib/services/analysis.ts` (create)

**Intent**: All scoring, aggregation, and recommendation logic lives here as pure, exported functions with no side effects. The analysis runs client-side (in the React component) at the moment the user clicks "Analyze", using team data already fetched per-selection.

**Contract**:

`computeOffensiveCoverage(team: TeamMember[]): CoverageResult`
— Union all `member.offensiveCoverage` arrays across non-error members. `covered` = the resulting unique set. `gaps` = `ALL_TYPES` minus `covered`.

`computeDefensiveWeaknesses(team: TeamMember[]): WeaknessEntry[]`
— For each of 18 attacking types, collect the non-error team members that have it in their `effectiveWeaknesses`. Group by attacking type. For each group: count members, collect names, compute `avgRelevantDefStat` (use `stats.defense` for physical types, `stats.specialDefense` for special types — see classification below), flag `hasQuadWeak` if any member has multiplier 4 in their `effectiveWeaknesses` for this type. Return only entries where `affectedCount ≥ 1`, sorted by `affectedCount` descending, then `avgRelevantDefStat` ascending (lower stat = higher severity tiebreak).

**Physical attacking types** (use `stats.defense`): normal, fighting, poison, ground, flying, bug, rock, ghost, steel

**Special attacking types** (use `stats.specialDefense`): fire, water, grass, electric, ice, psychic, dragon, dark, fairy

`computeScore(coverage: CoverageResult, weaknesses: WeaknessEntry[], teamSize: number): ScoreResult`
— 
```
offenseScore = Math.round((coverage.covered.length / 18) * 100)

// For each weakness entry:
penalty(entry) = entry.affectedCount * 8 * (75 / entry.avgRelevantDefStat) * (entry.hasQuadWeak ? 1.5 : 1.0)

defenseScore = Math.max(0, Math.round(100 - Σ penalty))

overall = Math.round(offenseScore * 0.4 + defenseScore * 0.6)

grade:
  overall >= 90 → 'S'
  overall >= 75 → 'A'
  overall >= 60 → 'B'
  overall >= 45 → 'C'
  overall >= 30 → 'D'
  else          → 'F'
```
The constant `8` and `75` reference value may need calibration — they're tuned for a typical 6-member team where 3 shared weaknesses of average-def Pokémon should land in the C–B range.

`generateRecommendations(coverage: CoverageResult, weaknesses: WeaknessEntry[], teamSize: number): Recommendation[]`
— Produce a ranked list:
1. **Critical defense** (`affectedCount ≥ 3` or `hasQuadWeak`): "N of your M Pokémon are weak to [Type] — consider adding a Pokémon resistant to [Type]."
2. **Moderate defense** (`affectedCount = 2`): "2 of your M Pokémon are weak to [Type]."
3. **Offensive gap** (type in `coverage.gaps` + no defense recommendation already covers it): "Your team has no moves that hit [Type] Pokémon super-effectively."
4. **Minor defense** (`affectedCount = 1`): only include if total recommendations so far < 3.

Cap at 5 recommendations total. Return empty array if `teamSize === 0`.

`analyzeTeam(team: TeamMember[]): AnalysisResult`
— Orchestrates all above. Separates `failedMembers = team.filter(m => m.fetchError).map(m => m.name)`. Passes only non-error members to coverage/weakness/score functions. Returns assembled `AnalysisResult`.

---

#### 6. Unit tests

**File**: `src/lib/services/analysis.test.ts` (create)

**Intent**: Verify the trust-critical scoring logic (PRD: "wrong data destroys trust immediately") before the UI is built. Node built-in test runner avoids adding a test framework dependency.

**Contract**: Use `import { test, describe } from 'node:test'` and `import assert from 'node:assert/strict'`. Import analysis functions with relative paths (`'./analysis.js'` — note `.js` extension required for Node ESM). Include at minimum:

- `computeOffensiveCoverage`: team of one electric-type member with electric coverage → `covered` includes 'water', 'flying'; `gaps` contains all others
- `computeDefensiveWeaknesses`: two fire-type members both weak to water → `WeaknessEntry` for water has `affectedCount: 2`
- `computeScore` boundary: `coverage.covered.length = 18`, `weaknesses = []` → `overall = 100`, `grade = 'S'`
- `computeScore` boundary: `coverage.covered.length = 0`, `weaknesses` with 6 members all weak to fire at low def → `grade` is 'D' or 'F'
- `generateRecommendations`: 3 members weak to rock → first recommendation is 'critical' defense for rock
- `analyzeTeam` with mixed team (some `fetchError: true`) → `failedMembers` populated, failed members excluded from scoring

Run with: `node --experimental-strip-types --test src/lib/services/analysis.test.ts`

---

### Success Criteria

#### Automated Verification

- `node --experimental-strip-types --test src/lib/services/analysis.test.ts` exits 0 with all tests passing
- `npm run lint` passes

#### Manual Verification

- Review test output: all test names printed, no assertion errors, coverage/score values look intuitively right for the test cases chosen

**Implementation Note**: After this phase, pause for manual review of test output and a sanity-check of the scoring formula with a real example (e.g. a classic well-balanced team like Charizard/Vaporeon/Raichu/Machamp/Starmie/Snorlax should score above 60).

---

## Phase 3: Analyzer UI

### Overview

Install the autocomplete component, build the React component tree, and replace the starter landing page. At the end of this phase the full end-to-end flow works in a browser.

### Changes Required

#### 7. shadcn Command component

**File**: run `npx shadcn@latest add command` from project root

**Intent**: Add the shadcn Command (combobox) component for the autocomplete Pokémon search input. This installs `src/components/ui/command.tsx` plus any peer dependencies (`cmdk`).

**Contract**: After running, `src/components/ui/command.tsx` exists and `npm run lint` passes.

---

#### 8. Type badge component

**File**: `src/components/TypeBadge.tsx` (create)

**Intent**: Render a small colored pill showing a Pokémon type name. Used in team slots and the results panel. The 18 Pokémon types have established community colors — use those to make the badges instantly recognisable.

**Contract**: Props: `{ type: TypeName; size?: 'sm' | 'md' }`. Renders a `<span>` with `cn()` for class merging. The color mapping (background + text) covers all 18 `TypeName` values — choose colors that approximate the conventional Pokémon type palette (fire = orange, water = blue, grass = green, electric = yellow, etc.). Defaults to a neutral gray for any unmapped type as a safe fallback.

---

#### 9. Pokémon search component

**File**: `src/components/PokemonSearch.tsx` (create)

**Intent**: Autocomplete input for adding Pokémon to the team. Fetches the full name list once on mount; all filtering is local. Calls `onSelect` when the user picks a name.

**Contract**: Props: `{ onSelect: (name: string) => void; disabled?: boolean }`. On mount, `fetch('/api/pokemon-list')` and store the results array in local state. While loading, show the input as disabled with a loading indicator. Render the shadcn `Command` component with a `CommandInput` and a scrollable `CommandList` of `CommandItem` entries filtered by the current input value. On item selection, call `onSelect(name)` and clear/close the input. `disabled` prop blocks interaction while a team slot is being fetched.

---

#### 10. Team slot component

**File**: `src/components/TeamSlot.tsx` (create)

**Intent**: Display one Pokémon in the team builder: name, types, and a remove button. Shows a loading skeleton while the per-selection data fetch is in flight, and an inline error state if the fetch failed.

**Contract**: Props: `{ member: TeamMember; loading?: boolean; onRemove: () => void }`. Three visual states:
1. `loading = true`: show a pulsing placeholder skeleton in the slot shape
2. `member.fetchError = true`: show the Pokémon name + a red inline message "Data unavailable — excluded from analysis" + a remove button
3. Default: show `member.displayName`, one `TypeBadge` per type, and a remove (×) button using `lucide-react`'s `X` icon

---

#### 11. Analysis results panel

**File**: `src/components/AnalysisPanel.tsx` (create)

**Intent**: Render all four required analysis outputs in a single scrollable view beneath the team builder. Receives `AnalysisResult` and a list of failed member names as props.

**Contract**: Props: `{ result: AnalysisResult }`. Render four sections in order:

**Section 1 — Overall score**: Large numeric score + letter grade (colour-coded: S/A = green, B = blue, C = yellow, D/F = red) + one-line plain-language verdict ("Your team has solid coverage but shares notable weaknesses.").

**Section 2 — Offensive coverage**: Two groups of type badges — "Covered" (types the team can hit SE) in their type colors, "Gaps" (types the team cannot) in a muted/grey style. A brief sentence: "Your team covers N of 18 types super-effectively."

**Section 3 — Defensive weaknesses**: List of `WeaknessEntry` items from `result.weaknesses`, sorted as returned from the engine (most severe first). For each entry: the attacking type (as `TypeBadge`), "N members affected" count, and the member names inline. Entries where `affectedCount ≥ 3` are visually highlighted (e.g. a warning icon or stronger color).

**Section 4 — Recommendations**: Ordered list of `result.recommendations`. Each item shows the recommendation text. Critical recommendations use a distinct visual treatment (e.g. red dot or warning icon). If `result.failedMembers.length > 0`, show a notice above the recommendations: "Note: [name1], [name2] could not be loaded and are excluded from this analysis."

---

#### 12. Team analyzer root component

**File**: `src/components/TeamAnalyzer.tsx` (create)

**Intent**: Root interactive component. Manages all state — the team list, per-slot loading flags, and the analysis result. Orchestrates the per-selection fetch → analysis → display flow.

**Contract**: No external props. Internal state:

```typescript
team: TeamMember[]        // max 6, appended per selection
loadingSlot: string | null  // name of the Pokémon currently being fetched
analysis: AnalysisResult | null  // null until "Analyze" is clicked
analyzing: boolean           // true while analyzeTeam() runs (should be instant, but for UX)
```

Behaviour:
- On Pokémon selected from `PokemonSearch`: set `loadingSlot = name`; `fetch('/api/pokemon/${name}')`; on success append `TeamMember` to `team`, clear `loadingSlot`; on 503/error append a `TeamMember` with `fetchError: true` and `name = name`, clear `loadingSlot`. Do not add to team if `team.length >= 6`.
- Disable `PokemonSearch` while `loadingSlot !== null` or `team.length >= 6`.
- "Analyze" button: disabled if `team.filter(m => !m.fetchError).length === 0`. On click: call `analyzeTeam(team)` from `@/lib/services/analysis`, set result in state, scroll to results.
- "Reset" button: clears team and analysis result.
- Render layout: page title ("MonRate"), tagline, `PokemonSearch`, team slot grid (up to 6 `TeamSlot` components), Analyze/Reset buttons, and — when `analysis !== null` — `AnalysisPanel`.

---

#### 13. Main page

**File**: `src/pages/index.astro` (update — replace content)

**Intent**: Replace the starter landing page with the team analyzer. The analyzer is the entire product for v1; no landing page is needed.

**Contract**: Remove the `Welcome` import and usage. Import `TeamAnalyzer` from `@/components/TeamAnalyzer`. Render `<Layout title="MonRate — Pokémon Team Analyzer"><TeamAnalyzer client:load /></Layout>`. The `client:load` directive hydrates the React component immediately on page load (same pattern as `SignInForm` in `src/pages/auth/signin.astro:16`).

---

#### 14. Remove Welcome component

**File**: `src/components/Welcome.astro` (delete)

**Intent**: The component is no longer referenced after the main page update. Leaving it as dead code contradicts CLAUDE.md's "avoid backwards-compatibility hacks" guideline.

**Contract**: Delete the file. Confirm `npm run lint` still passes (no other files import it).

---

### Success Criteria

#### Automated Verification

- `npm run build` exits 0 — production build with no TypeScript errors
- `npm run lint` passes
- `node --experimental-strip-types --test src/lib/services/analysis.test.ts` still passes

#### Manual Verification

- Visit `http://localhost:4321` — shows the analyzer UI, not the starter landing page
- Type "pika" in the search input — autocomplete shows "pikachu" and related names within 1–2 seconds of first keystroke
- Select "pikachu" — slot appears with `TypeBadge` showing "electric"; no login prompt
- Select 5 more Pokémon — sixth slot fills in; search input is disabled after the sixth
- Click "Analyze" — results panel appears in the same view with all four sections populated
- Verify the Charizard defensive weaknesses match the expected values from Phase 1 manual check (rock 4×, water 2×, electric 2× — NOT ground)
- Test graceful degradation: temporarily break the PokeAPI service (modify `buildTeamMember` to throw) — the slot should show the inline error, and "Analyze" should still run on remaining members with a "Note: X excluded" message
- Resize to mobile viewport (≤ 375px) — layout is usable; type badges and recommendations are readable
- Full analysis takes < 3 seconds on a cold Cloudflare cache (warm local cache will be faster)

**Implementation Note**: After completing this phase and manual verification passes, the change is ready for `/10x-plan-review` or direct `/10x-implement`.

---

## Testing Strategy

### Unit Tests

- `computeOffensiveCoverage`: union of coverage arrays across members
- `computeDefensiveWeaknesses`: correct grouping, sorting, and `hasQuadWeak` detection
- `computeScore`: boundary cases (perfect coverage + no weaknesses = 100/S; zero coverage + all weak = low grade)
- `generateRecommendations`: critical defense surfaces first; cap at 5 items; empty team → empty list
- `analyzeTeam`: failed members excluded from scoring; `failedMembers` array populated

### Integration Tests

None for v1. The Phase 1 manual curl checks serve as integration verification.

### Manual Testing Steps

1. Fresh browser visit → analyzer at `/`, no landing page redirect
2. Autocomplete: type partial name → filtered list appears; select → slot populates with correct types
3. Add 6 Pokémon → search disabled; remove one → search re-enabled
4. Analyze a team with known weaknesses (e.g. 4 Fire-type Pokémon) → Water weakness should appear as 'critical' in recommendations
5. Analyze a well-balanced team → overall score ≥ 60 (B or better)
6. Test with 1 Pokémon (minimum valid input) → analysis produces meaningful output, not crash
7. Mobile viewport: all controls reachable; no horizontal overflow

---

## Performance Considerations

The <3s NFR covers the full round-trip for a team analysis click. With per-selection pre-fetching:
- At "Analyze" click time, all team member data is already in React state — the analysis engine runs synchronously in < 1ms
- The response time budget is spent during team building (one `fetch('/api/pokemon/{name}')` per selection)
- Per-selection latency target: < 1s warm, < 2s cold (first request for a Pokémon before CF Cache is warm)
- The CF Cache API (`caches.default`) is the primary mechanism; without it the <3s NFR fails for cold requests

The `/api/pokemon-list` call (autocomplete data) happens on component mount and runs in parallel with the user reading the page — it does not block the first interaction.

---

## References

- Roadmap: `context/foundation/roadmap.md` → S-01
- PRD: `context/foundation/prd.md` → §Business Logic, §User Stories US-01, §Functional Requirements FR-001 – FR-005
- PokeAPI endpoint pattern: `src/pages/api/auth/signin.ts:1`
- React component pattern: `src/components/auth/SignInForm.tsx:1`
- shadcn config: `components.json`
- CF Workers config: `wrangler.jsonc`

---

## Progress

> Convention: `- [ ]` pending, `- [x]` done. Append ` — <commit sha>` when a step lands. Do not rename step titles.

### Phase 1: PokeAPI Integration Layer

#### Automated

- [ ] 1.1 `npm run lint` passes with types.ts, pokeapi.ts, and both endpoints in place
- [ ] 1.2 `GET /api/pokemon-list` returns JSON with `results` array of ≥ 900 names
- [ ] 1.3 `GET /api/pokemon/pikachu` returns `types: ['electric']` and `fetchError: false`
- [ ] 1.4 `GET /api/pokemon/notarealmon` returns HTTP 503

#### Manual

- [ ] 1.5 Charizard endpoint shows `rock` as a 4× weakness and `ground` not in weaknesses (Flying immunity)
- [ ] 1.6 A Levitate Pokémon (e.g. `bronzong`) shows `abilityImmunity: 'ground'`
- [ ] 1.7 Smeargle's `offensiveCoverage` is non-trivially wide (covers ≥ 15 of 18 types)

### Phase 2: Analysis Engine

#### Automated

- [ ] 2.1 `node --experimental-strip-types --test src/lib/services/analysis.test.ts` exits 0
- [ ] 2.2 `npm run lint` passes

#### Manual

- [ ] 2.3 Sanity-check: a balanced 6-member team (e.g. fire/water/grass/electric/fighting/psychic) scores B or higher

### Phase 3: Analyzer UI

#### Automated

- [ ] 3.1 `npm run build` exits 0 (production build, no TypeScript errors)
- [ ] 3.2 `npm run lint` passes
- [ ] 3.3 Analysis unit tests still pass

#### Manual

- [ ] 3.4 Autocomplete populates within 2s of first keystroke on a cold load
- [ ] 3.5 Selecting 6 Pokémon fills all team slots with correct type badges
- [ ] 3.6 Analysis results show all four sections with correct data for a known team
- [ ] 3.7 Graceful degradation: a failed Pokémon slot shows inline error; analysis runs on remaining members
- [ ] 3.8 Mobile viewport (≤ 375px): layout usable, no horizontal overflow
- [ ] 3.9 Full team analysis completes < 3 seconds on a cold Cloudflare cache
