# CLAUDE.md

## Hard rules

- `SUPABASE_URL` and `SUPABASE_KEY` must be imported from `astro:env/server` only — never expose them to the client.
- Always enable RLS on new Supabase tables with per-operation, per-role policies.

## Commands

Pre-commit hooks (husky + lint-staged) run `eslint --fix` on `*.{ts,tsx,astro}` and `prettier --write` on `*.{json,css,md}`.

No test suite is configured.

## Architecture

Full SSR (`output: "server"` in `astro.config.mjs`) — every page is server-rendered. There is no static pre-rendering.

### Auth flow

- `src/lib/supabase.ts` — creates a Supabase SSR client (cookie-based sessions via `@supabase/ssr`). `SUPABASE_URL` and `SUPABASE_KEY` are declared as server-only secrets in `astro.config.mjs` `env.schema`.
- `src/middleware.ts` — runs on every request, resolves `context.locals.user` (typed in `src/env.d.ts`). To protect a route, add its path to `PROTECTED_ROUTES`; unauthenticated users are redirected to `/auth/signin`.
- API endpoints: `src/pages/api/auth/{signin,signup,signout}.ts`
- Auth pages: `src/pages/auth/{signin,signup,confirm-email}.astro`

### Conventions

- **Path alias**: `@/*` → `./src/*` (tsconfig paths).
- **Component split**: Astro components for layout/static content; React components only when interactivity is required.
- **Class merging**: use `cn()` from `@/lib/utils` (clsx + tailwind-merge) — never concatenate Tailwind class strings manually.
- **shadcn/ui**: components live in `src/components/ui/`, "new-york" style variant. Add new ones with `npx shadcn@latest add [name]`.
- **React**: no Next.js directives (`"use client"` etc.). Extract reusable hooks to `src/components/hooks/`.
- **Services/helpers** go in src/lib/. Move to src/lib/services/ when the code (a) is called from two or more
  pages/endpoints, or (b) makes Supabase queries or external API calls directly.
- **Shared types** (entities, DTOs) go in `src/types.ts`.
- **Supabase migrations**: `supabase/migrations/YYYYMMDDHHmmss_short_description.sql`.

### Environment

@README.md

## CI

CI also runs npx astro sync before lint — see @.github/workflows/ci.yml.

<!-- BEGIN @przeprogramowani/10x-cli -->

## 10xDevs AI Toolkit - Module 2, Lesson 1

Move from sprint-zero setup to project orchestration with the **roadmap chain**:

```
(Module 1 foundation docs) -> /10x-roadmap -> backlog-ready roadmap items
```

`/10x-roadmap` is the lesson focus. `/10x-new` is intentionally introduced in Module 2, Lesson 2, when a selected roadmap item becomes an implementation change folder.

### Task Router - Where to start

| Skill | Use it when |
| --- | --- |
| **Roadmap (lesson focus)** | |
| `/10x-roadmap` | You have `context/foundation/prd.md` and a scaffolded project baseline, and you need a vertical-first MVP roadmap. The skill reads the PRD, inspects the code baseline, uses available foundation docs such as `tech-stack.md`, `infrastructure.md`, and `deploy-plan.md`, then writes `context/foundation/roadmap.md`. Use it BEFORE creating per-change folders or implementation plans. |
| **Re-run upstream if needed** | |
| `/10x-shape` / `/10x-prd` / `/10x-tech-stack-selector` / `/10x-bootstrapper` / `/10x-agents-md` / `/10x-infra-research` | Bundled from Module 1 so foundation contracts can be fixed before roadmap sequencing. If roadmap generation exposes a PRD gap, repair the PRD before pretending the backlog is ready. |

### How the chain hands off

- `/10x-roadmap` bridges product and implementation. It does not choose frameworks, design schemas, or write a per-change implementation plan.
- The output is `context/foundation/roadmap.md`: ordered milestones, vertical slices, bounded foundations, dependencies, unknowns, risk, and backlog handoff fields.
- Roadmap items should receive stable human-readable identifiers in backlog tools. The actual `context/changes/<change-id>/` folder is created in Lesson 2 with `/10x-new`.

### Roadmap boundaries

- Default to vertical slices: user-visible outcomes that cross UI, data, business logic, and integrations.
- Horizontal work is allowed only as a bounded enabler that names the downstream vertical milestone it unlocks.
- Avoid orphan horizontal work such as "build the whole database", "build all API endpoints", or "design the whole UI" before the first user-visible flow.
- Roadmap is not a calendar estimate. Do not invent dates, story points, or sprint velocity unless the user explicitly asks for a separate planning artifact.

### Foundation paths used by this lesson

- `context/foundation/prd.md` - input
- `context/foundation/tech-stack.md` - optional input
- `context/foundation/infrastructure.md` - optional input
- `context/deployment/deploy-plan.md` - optional input
- `context/foundation/roadmap.md` - output
- `context/foundation/lessons.md` - recurring rules and pitfalls
- `docs/reference/contract-surfaces.md` - load-bearing names registry

Skills must not write to `context/archive/`. Archived changes are immutable; if a resolved target path starts with `context/archive/`, abort with: "This change is archived. Open a new change with `/10x-new` instead."

<!-- END @przeprogramowani/10x-cli -->
