# Playwright E2E Test Overview

An inventory of the Playwright test suite under `e2e-tests/playwright/specs/`.

> Counts gathered via static grep of `specs/` on **2026-06-07**. For authoritative
> per-project execution counts (tests × browser projects), run `npx playwright test --list`.

---

## Totals

- **152** spec files (`*.spec.ts`)
- **~558** test cases (individual `test(...)` blocks)
- Run across **3 browser "projects"** — `chrome`, `firefox`, `ipad` (iPad Pro 11) — gated by a
  `setup` project that runs first. Executed count = tests × the projects they're enabled for.

---

## By type (directory = category)

| Type | Files | Test cases | What it checks |
|------|------:|-----------:|----------------|
| **Functional** | 128 | 478 | Real UI behavior — channels, messages, threads, search, drafts, system console, plugins, settings. The bulk of the suite. |
| **Accessibility** | 17 | 66 | WCAG / a11y compliance (keyboard nav, ARIA, contrast); many also capture a11y snapshots. |
| **Visual** | 6 | 8 | Visual-regression snapshot comparisons (pixel diffs of pages/components). |
| **Client** | 1 | 6 | API-level tests using `Client4` directly — **no browser UI** (e.g. file upload via REST). |

Plus `specs/test_setup.ts` (the `setup` project) — 3 bootstrap steps, not feature tests.

---

## Tags (used for filtering with `--grep`)

Genuine Playwright filter tags (passed via `{tag: ...}` on each test):

| Tag | ~Count | Notes |
|-----|------:|-------|
| `@accessibility` | 55 | a11y tests |
| `@snapshots` | 43 | snapshot-based (visual / a11y) |
| `@settings` | 41 | settings area |
| `@autotranslation` | 38 | auto-translate feature |
| `@system_console` | 28 | admin console |
| `@classification_markings` | 23 | |
| `@user_attributes` | 21 | ABAC / custom attributes |
| `@custom_profile_attributes` | 12 | CPA |
| `@smoke` | 10 | quick stack-health checks |
| `@visual` | 9 | visual-regression |
| others | — | `@slash_commands`, `@global_banner`, `@scheduled_messages`, `@ai_recaps`, `@team_menu`, `@display_settings`, `@anonymous_urls`, `@managed_categories`, `@playwright`, … |

**Not filter tags (don't be misled by raw grep):**
- `@objective`, `@precondition`, `@reference`, `@testcase`, `@example` — JSDoc doc-comment
  annotations describing each test's intent.
- `@mattermost` (199 occurrences) — the npm import scope (`@mattermost/playwright-lib`), not a tag.

---

## Run mapping

| Command | Runs |
|---------|------|
| `npm run test:smoke` | the **10 `@smoke`** tests (Chrome, 1 retry) |
| `npm run test:ci` | everything **except `@visual`** (Chrome) — functional + accessibility + client |
| `npm run test` | all ~558 across **Chrome + Firefox + iPad**, including visual snapshots |

See `LOCAL.md` (repo root) for how to run these against a local server.
