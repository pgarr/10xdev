---
project: 10x-astro-starter (monrate)
checked_at: 2026-05-18T14:55:00Z
health_status: critical-issues
context_type: brownfield
language_family: js
stack_assessment_available: false
checks_run:
  - lockfile
  - dependency_audit
  - outdated_deps
  - test_runner
  - ci_cd
  - configuration
audit_findings:
  critical: 0
  high: 1
  moderate: 5
  low: 0
test_runner_detected: false
ci_provider: github-actions
recommended_fixes: 5
---

## Dependency Health

### Lockfile

```
Status:          present (package-lock.json)
Package manager: npm
```

Lockfile is present and up-to-date. Dependency versions are pinned.

### Security Audit

```
Tool:    npm audit --json
Summary: 0 CRITICAL, 1 HIGH, 5 MODERATE, 0 LOW
Direct vs transitive: 0/0/1/0 direct of total 0/1/5/0 (CRITICAL/HIGH/MODERATE/LOW)
```

#### CRITICAL findings

None.

#### HIGH findings

- **devalue** (5.6.3–5.8.0) — GHSA-77vg-94rm-hx3p: "Svelte devalue: DoS via sparse array deserialization". CVSS 7.5, CWE-770. **Transitive** (not a direct dependency). Fix available: `npm audit fix`.

#### MODERATE findings

- **@astrojs/check** (>=0.9.3) — **direct dependency**. Pulls in advisory chain via `@astrojs/language-server`. Fix: downgrade to 0.9.2 (`npm audit fix --force` — semver major).
- **@astrojs/language-server** (>=2.14.0) — transitive via `volar-service-yaml`. Fixed by resolving `@astrojs/check`.
- **volar-service-yaml** (<=0.0.70) — transitive via `yaml-language-server`. Fixed by resolving `@astrojs/check`.
- **yaml** (2.0.0–2.8.2) — GHSA-48c2-rrv3-qjmp: "Stack overflow via deeply nested YAML collections". CVSS 4.3, CWE-674. Transitive.
- **yaml-language-server** (range) — transitive via `yaml`. Fixed by resolving `@astrojs/check`.

All HIGH and MODERATE findings are in the **language-server / dev-tooling chain** — they affect the development environment, not the production Cloudflare Pages runtime.

### Outdated Dependencies

```
Packages with major version gaps: 0
```

No packages are 2+ major versions behind. Three packages have 1 major version gap (informational — no action required now):

- **typescript**: 5.9.3 → 6.0.3 (1 major behind)
- **eslint**: 9.39.4 → 10.4.0 (1 major behind)
- **lint-staged**: 16.4.0 → 17.0.5 (1 major behind)

Remaining outdated packages are minor/patch bumps and can be addressed with `npm update`.

---

## Test Suite

```
Test runner:    not detected
Tests found:    n/a
Test execution: not attempted
```

⚠ **No test runner detected.** The `package.json` has no `test` script and no test runner (`vitest`, `jest`, `playwright`, `cypress`) is installed as a dependency. No test configuration files were found.

This is the most impactful gap for agent-assisted development. An AI agent generating code changes has no automated way to verify correctness — regressions can go undetected until a user notices them in the browser.

**Recommended**: Set up Vitest (the natural choice for Astro + TypeScript projects):

```bash
npm install --save-dev vitest @vitest/ui
```

Then add to `package.json` scripts:

```json
"test": "vitest run",
"test:watch": "vitest",
"test:ui": "vitest --ui"
```

---

## CI/CD

```
Provider:      GitHub Actions
Configuration: .github/workflows/ci.yml
```

| Stage      | Status | Notes                                                  |
|------------|--------|--------------------------------------------------------|
| Lint       | ✓      | `npm run lint` → ESLint with typescript-eslint         |
| Test       | ✗      | No test step — no test runner installed yet            |
| Build      | ✓      | `npm run build` with Supabase env secrets              |
| Type check | ✗      | No `astro check` or `tsc --noEmit` step                |
| Security   | ✗      | No audit, Snyk, CodeQL, or Dependabot configured       |

The CI pipeline covers the essentials (lint + build) but is missing a test step and a type-check step. Both are quick additions once the test runner is in place.

---

## Configuration

### High severity

None.

### Medium severity

- **No test step in CI** — even after installing a test runner, CI will not catch test failures until the workflow is updated. Fix: add `- run: npm test` to `.github/workflows/ci.yml` after installing Vitest.

- **No type-check step in CI** — `tsconfig.json` extends `astro/tsconfigs/strict` (excellent), but strict types are not enforced in the pipeline. Agent-generated code with type errors will pass CI. Fix: add `- run: npx astro check` to `.github/workflows/ci.yml` before the build step.

### Low severity

- **`.editorconfig` missing** — ensures consistent indentation/line-endings across editors and AI tools. Not critical but useful for multi-tool workflows. Fix: create a `.editorconfig` with standard settings (takes < 2 minutes).

---

## Stack Assessment Cross-Reference

```
No stack-assessment.md found. Run /10x-stack-assess for quality-gate analysis.
```

---

## Recommended Fixes

### Fix before agent work (Category A)

#### 1. Install a test runner

**Impact**: Without a test runner, the agent cannot verify its own changes. It generates code but cannot confirm correctness — each change requires manual browser testing to catch regressions. This is the single biggest bottleneck for productive agent collaboration.
**Severity**: critical
**Effort**: moderate (15–30 min)
**Fix**:

```bash
npm install --save-dev vitest @vitest/ui
```

Add to `package.json` `scripts`:

```json
"test": "vitest run",
"test:watch": "vitest"
```

Create an initial smoke test (e.g., `src/lib/utils.test.ts`) to confirm the runner works. Then write tests covering any existing utility logic in `src/lib/`.

---

#### 2. Add a type-check step to CI

**Impact**: The project has `strict` TypeScript (via `astro/tsconfigs/strict`) — a strong foundation. But CI doesn't enforce types, so agent-generated type errors pass CI undetected. Cheap to fix now; expensive to discover in production.
**Severity**: high
**Effort**: quick (< 5 min)
**Fix**:

In `.github/workflows/ci.yml`, add after `npm ci` and `astro sync`:

```yaml
- run: npx astro check
```

This runs the Astro type-checker (already installed as `@astrojs/check`) and fails CI on type errors.

---

#### 3. Add a test step to CI

**Impact**: Once tests exist (fix #1), CI must run them or they provide no safety net. A test suite that CI ignores is documentation, not a safety gate.
**Severity**: high
**Effort**: quick (< 5 min)
**Fix**:

In `.github/workflows/ci.yml`, add after the type-check step:

```yaml
- run: npm test
```

Do this after completing fix #1.

---

#### 4. Resolve HIGH audit advisory

**Impact**: The `devalue` HIGH advisory (CVSS 7.5) is in a transitive dependency within Astro's SSR chain. It describes a DoS via sparse array deserialization. Risk for this project (static Astro + Cloudflare Pages) is low-to-medium, but it's worth resolving.
**Severity**: high
**Effort**: quick (< 5 min)
**Fix**:

```bash
npm audit fix
```

This handles all auto-fixable advisories. For the `@astrojs/check` moderate chain (requires a semver-major downgrade), evaluate separately — the risk is dev-tooling only.

---

#### 5. Add `.editorconfig`

**Impact**: Ensures consistent formatting between your editor, AI coding tools, and the Prettier config. Minor, but helpful when multiple tools edit the same files.
**Severity**: low
**Effort**: quick (< 5 min)
**Fix**:

Create `.editorconfig`:

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
```

---

### Addressed in upcoming lessons (Category B)

#### Missing AGENTS.md

**Lesson**: [Agent Onboarding: Agents.md, AI Rules i feedback loops (M1L4)](https://platforma.przeprogramowani.pl/external/10xdevs-3/m1-l4)
**What you'll do there**: Build `AGENTS.md` and enrich `CLAUDE.md` with project-specific agent rules, routing conventions, and feedback loops. Generating a stub now would be premature — the lesson walks you through building the right content.

#### No security scan in CI

**Lesson**: [Sprint Zero z Agentem: infrastruktura, walking skeleton i pierwszy deploy (M1L5)](https://platforma.przeprogramowani.pl/external/10xdevs-3/m1-l5)
**What you'll do there**: Set up Dependabot, `npm audit` in CI, and other security automation as part of the infrastructure sprint. Dependabot configuration, CodeQL, and automated advisory handling are all covered there.

#### No deployment configuration

**Lesson**: [Sprint Zero z Agentem: infrastruktura, walking skeleton i pierwszy deploy (M1L5)](https://platforma.przeprogramowani.pl/external/10xdevs-3/m1-l5)
**What you'll do there**: Wire up Cloudflare Pages, configure `wrangler.jsonc`, and set up the `SUPABASE_URL` / `SUPABASE_KEY` secrets in CI. The `wrangler.jsonc` is already scaffolded — the lesson covers connecting it to a live project.

---

## Summary

```
Health status: critical-issues
```

The project scaffolded cleanly and has strong fundamentals: locked dependencies, strict TypeScript, ESLint + Prettier + Husky pre-commit hooks, and a working lint + build CI pipeline. The critical gap is the **absence of a test runner** — without one, an AI agent generating code changes has no automated verification loop and every change requires manual validation. This is fixable in under 30 minutes (Vitest is the natural fit for this Astro + TypeScript stack). Secondary gaps are a missing type-check step in CI (5 minutes to add `npx astro check`) and one HIGH audit advisory in a transitive dev-tooling dependency (resolvable with `npm audit fix`). The security findings are entirely in the development toolchain and do not affect the production Cloudflare Pages runtime.

Next step: address the Category A fixes above — install Vitest and wire up the CI type-check and test steps — then proceed to agent onboarding (M1L4) where you'll build out `AGENTS.md` and the full agent instruction layer.
