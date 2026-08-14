---
name: testing-principles
description: Front-end test pyramid, framework matrix (Vitest/Jest unit, Testing Library component, Playwright e2e), framework-detection-first rule, coverage thresholds, and testing best practices. Use when generating or reviewing front-end tests, choosing a test runner, or setting a QA gate.
---

# Testing Principles Reference (Front-End)

Shared testing patterns, framework selection, coverage requirements, and quality
gates for web work. Used by `fe-test-generator`, the framework agents, and the
QA gate.

## Test Pyramid

```
         /\
        /  \      E2E Tests (5-10%)
       /────\     Real browser, critical user journeys (Playwright / Cypress)
      /      \
     /────────\   Component / Integration Tests (20-30%)
    /          \  Rendered components + user interaction (Testing Library)
   /────────────\ Unit Tests (60-70%)
  /              \ Pure functions, hooks, stores, utilities (Vitest / Jest)
```

Front-end weighting note: the **component layer** carries more of the load than in
back-end pyramids — a UI's behavior lives in rendered output and interaction, so a
healthy front-end suite is unit-heavy at the base but invests substantially in
Testing-Library component tests over brittle, slow full-browser e2e.

## Framework-Detection-First Rule (binding)

**Detect the project's existing test framework before writing a single test.
Never introduce a second framework into a project that already has one.** A repo
on Jest stays on Jest; a Vite project on Vitest stays on Vitest; a Playwright e2e
suite is not "improved" by adding Cypress. Adding a parallel runner doubles config,
splits coverage, and fragments CI.

Detection signals (top-down, first match wins):

| Signal | Framework in use |
|--------|------------------|
| `vitest` in `package.json` / `vitest.config.*` / `test` block in `vite.config.*` | Vitest |
| `jest` in `package.json` / `jest.config.*` / `jest` key in `package.json` | Jest |
| `@playwright/test` / `playwright.config.*` | Playwright (e2e) |
| `cypress` / `cypress.config.*` | Cypress (e2e) |
| `@testing-library/*` dep | Testing Library (component layer — pairs with Vitest or Jest) |
| `vitest-axe` / `jest-axe` / `@axe-core/playwright` | accessibility assertions in the existing runner |
| none of the above | Vitest for a Vite app, Jest for a CRA/legacy app; Playwright for new e2e |

When no runner exists, default to the one that matches the build tool (Vitest with
Vite, Jest with webpack/Babel/CRA) and Playwright for e2e — but state the choice
and confirm before scaffolding config.

## Framework Matrix

| Layer | Runner / library | Accessibility add-on | Focused run |
|-------|------------------|----------------------|-------------|
| Unit (functions, hooks, stores) | Vitest (Vite) / Jest (webpack) | — | `vitest -t <name>` / `jest -t <name>` |
| Component (render + interact) | Testing Library (`@testing-library/react`/`vue`/`svelte`/`angular`) on Vitest/Jest | `vitest-axe` / `jest-axe` | same `-t` filter |
| E2E (real browser, journeys) | Playwright (`@playwright/test`) / Cypress | `@axe-core/playwright` | `playwright test -g <title>` |
| Visual / snapshot | Playwright screenshots / Storybook + a vis-diff tool | — | per-story |

> Requires Vitest (Vite-native test runner; tracks the installed Vite major). Fallback: Jest with `ts-jest`/babel for non-Vite projects. Canonical: _shared/version-feature-matrix.md

Registration is part of test generation: the file must match the runner's
discovery glob (`*.test.ts(x)`/`*.spec.ts(x)`, Playwright's `testDir`). A test in a
file the runner does not pick up does not exist.

## Testing-Library Doctrine (component layer)

The single most important front-end testing principle: **test behavior the user
observes, not implementation details.**

- Query by **accessible role/name/label** (`getByRole('button', { name: /save/i })`), then by text, then by `data-testid` as a last resort. Never query by class name or component internals.
- Interact via `userEvent` (real keyboard/pointer sequences), not by calling handlers directly or firing synthetic low-level events.
- Assert on rendered output, accessible state (`aria-*`, `disabled`, focus), and side effects the user can perceive — not on state variables, render counts, or props.
- This makes a11y testable for free: if you can't query an element by its accessible role/name, neither can a screen-reader user — that's a finding, not a test workaround.

## Coverage Targets

| Scope | Minimum | Target | Notes |
|-------|---------|--------|-------|
| Critical UI flows | 90% | 95%+ | Auth, checkout, forms with validation, payment |
| Components with logic | 75% | 80%+ | State, conditionals, effects, data fetching |
| Presentational components / hooks / utils | 60% | 70%+ | Shared building blocks |
| Generated / config / pure-style code | N/A | N/A | Excluded from coverage |

## Test Types

| Type | Purpose | Scope | Speed |
|------|---------|-------|-------|
| Unit | Verify isolated logic | Pure function / hook / store | <50ms |
| Component | Verify render + interaction | One component + its subtree | <500ms |
| Integration | Verify component interactions / data flow | Multiple components, mocked network (MSW) | <2s |
| E2E | Verify user journeys | Real browser, real routing | <30s |
| Accessibility | Catch WCAG violations | Component + page (axe) | fast |
| Visual | Catch unintended visual change | Rendered component/page snapshot | fast |

Accessibility assertions are a first-class test type — run `axe` on rendered
components in the component layer and on key pages in e2e; an axe violation is a
QA-gate blocker (see `accessibility-baseline.md`).

## Testing Best Practices

### Naming Convention

```
<component/unit> <scenario> <expected>
"SaveButton disabled while submitting shows spinner"
"useCart adding a duplicate item increments quantity"
```

### AAA Pattern

```tsx
test("LoginForm invalid email shows an error", async () => {
  // Arrange
  const user = userEvent.setup();
  render(<LoginForm />);

  // Act
  await user.type(screen.getByLabelText(/email/i), "not-an-email");
  await user.click(screen.getByRole("button", { name: /log in/i }));

  // Assert — observable, accessible output
  expect(screen.getByRole("alert")).toHaveTextContent(/valid email/i);
});
```

### Mock the Boundary, Not Your Modules

- Mock the **network** with MSW (request interception), not by stubbing your own fetch wrapper — MSW exercises the real serialization path.
- Mock the **clock** (`vi.useFakeTimers()`) for time-dependent UI, never `sleep`.
- Do not mock your own components/hooks to "make a test pass" — that tests the mock, not the code.

### Test Independence

- Each test renders fresh — no shared mutable state, no ordering assumptions. Testing Library auto-cleans the DOM between tests; verify cleanup is configured.
- Deterministic: seed any random source; freeze the clock; no real network, no real `localStorage` bleed (clear it in `beforeEach`).
- Hermetic: stub network via MSW, timers via fake timers, storage via a fresh jsdom per file.

## Quality Gates

| Gate | Threshold | Action on Failure |
|------|-----------|-------------------|
| Coverage | >80% new code | Block merge |
| Unit + component + integration tests | All pass | Block merge |
| `axe` on changed components/pages | 0 violations | Block merge (QA + a11y gate) |
| Lighthouse performance budget | within budget | Block merge (QA gate) |
| Flaky tests | <1% flakiness | Investigation (quarantine, fix root cause) |
| E2E suite duration | reasonable per CI budget | Optimization / parallelization |

## Anti-Patterns to Avoid

- Querying by class name, `id`, or component internals instead of accessible role/text.
- Asserting on state/props/render counts instead of observable behavior.
- Firing low-level synthetic events instead of `userEvent` sequences.
- Excessive mocking — mock the boundary (network/clock/storage), not your own modules.
- Snapshot tests over entire large trees (brittle; obscure the actual assertion) — prefer targeted assertions.
- Flaky tests from real network, real timers, animation timing, or shared global state.
- Introducing a second test framework into a project that already has one (violates the framework-detection-first rule).
- Testing third-party library internals or the framework itself.

## Related Skills

- `accessibility-baseline.md` — the WCAG baseline and a11y-gate the axe assertions enforce
- `severity-matrix.md` — coverage requirements and the P0–P3 finding scale
- `version-feature-matrix.md` — Vitest/Playwright version floors
- `CORPFLOW.md` — the QA gate definition for worktask runs
