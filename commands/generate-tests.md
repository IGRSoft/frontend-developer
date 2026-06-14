---
description: Generate, register, and verify a runnable component/unit/e2e test suite for a web project using its existing framework (Vitest, Jest, Testing Library, Playwright, Cypress)
argument-hint: [path (default .)] [--type unit|component|e2e] [--coverage-gaps]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
estimated-cost:
  band: medium
  min-tokens: 3000
  max-tokens: 22000
  model-distribution:
    haiku: 15%
    sonnet: 75%
    opus: 10%
---

# Generate Tests
<!-- Updated: June 2026 -->

Generate a runnable test suite for a web component, module, or flow, register it with the project's test runner, and prove it runs before reporting success. The product is *passing, discoverable tests* — not test source files on disk.

[Extended thinking: The hard part of front-end test generation is not writing assertions — it is honoring the project's existing runner and renderer, wiring tests so the runner discovers them, and proving they execute. This command refuses to introduce a second framework into a project that already has one, routes the actual authoring to `frontend-developer:fe-test-generator` (which knows component-render + interaction + a11y + failure-mode coverage per framework), then runs a verification gate that reuses build-test's detect-build-test logic. A generated test that the runner never discovers, or that imports the wrong testing-library renderer, is a defect, not a deliverable. The command halts and routes the failure back rather than declaring victory.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Detect the framework FIRST — never introduce a second one.** Before generating, scan for the runner and renderer already in use (see Detection). If the project tests with Vitest + Testing Library, do NOT add Jest; if e2e uses Playwright, do NOT add Cypress. `--type` selects *which kind* of test to generate, not *which framework* — the in-use framework always wins. If no framework exists, scaffold the matrix default and announce it.
2. **Route generation to fe-test-generator.** Do NOT author test bodies yourself. The agent owns happy-path + interaction + edge-case + a11y + failure-mode coverage. You own detection, registration, and verification.
3. **Registration is part of the deliverable.** A test the runner cannot discover does not exist. After generation you MUST place files at the conventional path (`*.test.tsx`/`*.spec.ts`/`tests/e2e/*.spec.ts`), wire any config (`vitest.config`, `setupTests`, `playwright.config`), and verify by *listing* (`npx vitest list`, `npx playwright test --list`), not by eyeballing files.
4. **Verification gate is mandatory.** Generated tests MUST run. Reuse the detect + build + test logic from `/frontend-developer:build-test`. A suite that does not run is a FAILURE — report it and route the error back to the matching framework specialist. Do NOT report success on un-run tests.
5. **Single-command Bash invocations.** Use runner flags (`npx vitest run`, `npx playwright test`, `npm test --`). Never `cd`-chain or `&&`-chain — scoped Bash patterns do not match compound commands.
6. **Tool-missing never hard-fails.** If the runner's binary is absent, print the install hint, skip that test type, and continue. Report what was skipped. Never hard-fail the whole command for one missing tool.
7. **Commands route, they do not orchestrate.** This command names `frontend-developer:fe-test-generator` (and the owning framework agent on gate failure) so Claude routes the work; it does not call `Task`.
8. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Detect the framework and generate + register + verify tests for the current dir
/frontend-developer:generate-tests .

# Component tests for one component directory
/frontend-developer:generate-tests src/components/Checkout --type component

# An end-to-end flow test
/frontend-developer:generate-tests src/routes/checkout --type e2e

# Target untested branches surfaced by a coverage run
/frontend-developer:generate-tests . --coverage-gaps
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `path` | `.` | File, component, or directory to generate tests for. Detection is rooted at its enclosing project. |
| `--type unit\|component\|e2e` | auto | `unit` = pure functions/composables; `component` = render + interaction via Testing Library; `e2e` = browser flow via Playwright/Cypress. Auto: pick `component` for `.tsx`/`.vue`/`.svelte` UI, `unit` for plain `.ts`, prompt for `e2e` (it needs a route/flow). |
| `--coverage-gaps` | off | Run a coverage pass first (`vitest run --coverage`) and target generation at uncovered branches/lines. Requires a buildable, already-runnable existing suite to measure against. |

## Detection

Detection is the first and most important step. Scan top-down; the **in-use** framework always wins.

| Layer | Scan for (priority) | Resolved framework |
|-------|---------------------|--------------------|
| Unit/component runner | `vitest` in `package.json` / `vitest.config.*` | Vitest |
| Unit/component runner | `jest` in `package.json` / `jest.config.*` | Jest |
| Component renderer | `@testing-library/react` / `@testing-library/vue` / `@testing-library/svelte` / Angular `TestBed` | Testing Library (per framework) |
| E2E | `@playwright/test` / `playwright.config.*` | Playwright |
| E2E | `cypress` / `cypress.config.*` | Cypress |

Resolution rules:

- **In-use beats requested.** If the scan finds Vitest and the user implies Jest, use Vitest — do not mix runners.
- **Empty project.** No runner markers → use the matrix default from `skill: testing-principles`: Vitest + the framework's Testing Library for unit/component, Playwright for e2e. Scaffold the config and announce it.
- **UI framework drives the renderer.** React → `@testing-library/react`; Vue → `@testing-library/vue`; Svelte → `@testing-library/svelte`; Angular → `TestBed` + `@testing-library/angular`. Resolve the UI framework via `skill: language-detection`.

Keep this routing in sync with `skill: language-detection`, do not fork it.

## Workflow

### Phase 1: Detect (Bash + Read)

1. Confirm `path` exists. If not, emit the Error Handling "path not found" message and stop.
2. Resolve the UI framework and the test runner/renderer (Detection). Resolve `--type` (or auto-pick).
3. Verify the runner toolchain exists (`command -v npx`; the runner installs via the project). If missing, print the install hint (Graceful Degradation), skip that type, continue.
4. Identify the units under test: exported components + their props (`.tsx`/`.vue`/`.svelte`), public module functions/composables (`.ts`), or the route/flow under test (e2e). Read them so the generation brief carries real signatures and props.

### Phase 2: Coverage Baseline (Bash) — only with `--coverage-gaps`

1. Require an existing suite that already runs. If none, fall back to broad generation and note it.
2. Run `npx vitest run --coverage` (or `npx jest --coverage`), teeing to `.context/logs/`.
3. Parse uncovered lines/branches into a gap list (`{file, symbol, uncovered branch/line}`) to hand the generator, so it targets gaps rather than re-covering covered code.

### Phase 3: Generate (route to fe-test-generator)

Route per `--type` and framework. Pass the units under test, the resolved runner/renderer, and (if any) the coverage gap list. The agent must cover, per unit: **happy path + user interactions + edge cases + accessible-queries + failure modes** (loading/error/empty states, boundary props, async resolution/rejection).

- **component** → route to `frontend-developer:fe-test-generator`:
  "Generate {renderer} component tests for {framework} components in `{path}`: {component + props}. The project uses {runner} + {renderer} — do NOT introduce another runner/renderer. Cover: render with required/optional props; user interactions (`userEvent` click/type/keyboard); loading/error/empty states; accessible queries (`getByRole`/`getByLabelText`, not `getByTestId` unless unavoidable); async resolution and rejection. Follow `skill: fe-testing`. {coverage_gaps_block} Return the test files and any setup/config edits. Do not run the suite — I run the verification gate."
- **unit** → route to `frontend-developer:fe-test-generator` (same shape; renderer = none; focus on pure functions/composables, edge inputs, error paths).
- **e2e** → route to `frontend-developer:fe-test-generator`:
  "Generate {Playwright|Cypress} e2e tests for the flow in `{path}`. The project uses {framework} — do NOT introduce another. Cover the primary happy-path flow, one failure flow (validation/error), and one a11y assertion (`@axe-core/playwright` or role-based locators). Use stable locators (`getByRole`/`getByLabel`), web-first assertions, and `test.beforeEach` for setup. Follow `skill: fe-testing`. Return the spec files and any `playwright.config`/`cypress.config` edits. Do not run them."

If the generator returns tests in a framework other than the resolved one, reject that portion and re-brief — do not accept framework drift.

### Phase 4: Register (Write/Edit + Bash)

Wire the generated tests in so the runner discovers them. Registration is verified by *listing*, not by reading source.

| Framework | Registration | Discovery check |
|-----------|--------------|-----------------|
| Vitest | place as `*.test.ts(x)` / `*.spec.ts(x)` next to source or under `tests/`; ensure `vitest.config` `include` and `setupFiles` cover them | `npx vitest list` lists the new cases |
| Jest | place as `*.test.ts(x)`; ensure `jest.config` `testMatch`/`setupFilesAfterEach` | `npx jest --listTests` |
| Playwright | place as `tests/e2e/*.spec.ts`; ensure `playwright.config` `testDir` | `npx playwright test --list` |
| Cypress | place as `cypress/e2e/*.cy.ts`; ensure `cypress.config` `specPattern` | `npx cypress run --spec '...' --dry-run`-equivalent listing |

For empty-project scaffolding, also pin the dev dependency (`npm install -D vitest @testing-library/...` or `@playwright/test`) and the minimal config. Keep the pin in step with `skill: build-systems`.

### Phase 5: Verification Gate (Bash) — MANDATORY

Reuse `/frontend-developer:build-test`'s detect → run logic. Tee to `.context/logs/generate-tests-<timestamp>.log`.

1. **Type-check (TS):** `npx tsc --noEmit` over the touched files. A type error in a test is a gate FAIL (stage `compile`). Route back per Gate Failure.
2. **Discover:** run the runner's list command (Phase 4 column). If the new tests are NOT discovered → gate FAIL (registration defect). Fix registration and re-list.
3. **Run:**

   | Type | Run command |
   |------|-------------|
   | unit/component (Vitest) | `npx vitest run` |
   | unit/component (Jest) | `npx jest` |
   | e2e (Playwright) | `npx playwright test` (after `npx playwright install` if browsers are absent) |
   | e2e (Cypress) | `npx cypress run` |

   Capture `${PIPESTATUS[0]}`. **A suite that does not run is a FAILURE.** New tests that fail because they expose a real bug → report as a finding (the test is correct, the code is not); new tests that fail because they are wrong → route back to the generator.
4. Only when the suite **type-checks, is discovered, and runs** do you report success.

### Phase 6: Report (Bash)

Emit the Output Format summary. State pass/fail of the verification gate explicitly — never imply success without it.

## Gate Failure (route back, do not declare victory)

| Failure | Cause | Route to |
|---------|-------|----------|
| Test does not type-check | Bad import, wrong renderer API, prop-type mismatch | `frontend-developer:fe-test-generator` (fix the test) or the framework agent if the test exposed an API issue |
| Tests not discovered | Wrong path, missing `include`/`testDir`, missing setup file | Fix registration yourself (Phase 4), re-list |
| Test runs but a new test is wrong | Bad assertion / brittle locator / missing `await` | `frontend-developer:fe-test-generator` |
| Test runs and exposes a real bug | The component/flow is broken | Report as a finding; route the fix to the owning framework agent (`react`/`vue`/`svelte`/`angular`/`typescript`-developer) — do NOT weaken the test to make it pass |

Re-run the gate after each fix; report each cycle. Never silently auto-iterate.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint |
|--------------|--------------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) |
| `vitest` | `npm install -D vitest @vitest/coverage-v8` |
| `jest` | `npm install -D jest ts-jest @types/jest` |
| `@testing-library/*` | `npm install -D @testing-library/react @testing-library/user-event` (swap framework package) |
| `@playwright/test` | `npm install -D @playwright/test && npx playwright install` |
| `cypress` | `npm install -D cypress` |

If a runner is missing, print the hint, skip that test type, and continue with the others. With `--coverage-gaps` and no coverage tool, skip the baseline and fall back to broad generation with a note. Only when *every* targeted type is skipped does the command report "no runnable test runner available" with the aggregated hints — never a hard failure on a single missing tool.

## Output Format

```markdown
## Generate Tests Report

**Target:** {path}
**Framework:** {React | Vue | Svelte | Angular}
**Runner / renderer:** {Vitest + Testing Library | Jest | Playwright | Cypress} ({in-use | matrix default})
**Type:** {unit | component | e2e}
**Coverage mode:** {broad | --coverage-gaps targeting N gaps}
**Log:** .context/logs/generate-tests-{timestamp}.log

### Tests Generated ({count})
| File | Cases | Coverage focus |
|------|-------|----------------|
| Checkout.test.tsx | 6 | render, click, loading, error, empty, a11y role query |

### Registration
- {Placed as *.test.tsx + vitest include | tests/e2e + playwright testDir}
- Discovery check: {N tests now listed by `npx vitest list` / `npx playwright test --list`}

### Verification Gate
| Step | Result | Notes |
|------|--------|-------|
| Type-check | ✅ / ❌ | {0 errors, or first TS error} |
| Discover | ✅ / ❌ | {N new tests discovered} |
| Run | ✅ / ❌ | {N passed, M failed} |

**Gate:** PASS / FAIL ({failing step})

<!-- On gate failure only: -->
### Gate Failure
- **Stage:** {compile | discover | run}
- **Cause:** {one-line}
- **Routed to:** frontend-developer:{agent}
- **Status:** {patch applied + re-run PASS | awaiting fix | real bug surfaced — see finding}

<!-- When a new test exposed a real bug: -->
### Findings (tests correct, code under test failing)
- {file:line} — {what the test proved is broken} → fix routed to frontend-developer:{framework-agent}

<!-- On skipped types only: -->
### Skipped
- {type}: {missing tool} — install hint printed above.
```

## Error Handling

### Path not found
```
Error: Path not found: {path}
Suggestion: Pass a file or directory that exists, e.g. /frontend-developer:generate-tests src/components
```

### Framework conflict
```
Error: The requested runner/renderer conflicts with the one already in use ({detected}).
Mixing runners fragments the suite and the config.
Suggestion: Use {detected}, or migrate the whole suite first (out of scope for this command).
```

### --coverage-gaps with no runnable suite
```
Warning: --coverage-gaps needs an existing suite that already runs to measure.
None found — falling back to broad generation. Run generate-tests once, then re-run
with --coverage-gaps to target the remaining gaps.
```

## See Also

- `/frontend-developer:build-test` — the detect/build/test logic the verification gate reuses; run it first to confirm the project builds.
- `/frontend-developer:code-review` — review the code before adding tests; `--coverage-gaps` pairs well after a review.
- `/frontend-developer:a11y-audit` — deeper accessibility coverage than the role-based assertions generated here.
- `skill: fe-testing` — Vitest/Playwright/Testing Library patterns, framework detection, AAA/naming conventions.
- `skill: testing-principles` — test pyramid, framework matrix, coverage targets.
- `skill: language-detection` — canonical marker → framework → agent routing (keep the renderer table in sync).
