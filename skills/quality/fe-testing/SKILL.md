---
name: fe-testing
description: >-
  Front-end testing with framework detection — detect the project's runner first
  (Vitest/Jest, Testing Library, Playwright/Cypress), then write user-centric
  unit/component/e2e tests at the right pyramid layer. Use when generating or
  reviewing tests, choosing a runner, writing accessible component tests, or
  defining the testing completion gate.
---

# Front-End Testing

**Framework-detection-first testing across the pyramid.** For the domain selection
table, see the canonical [quality-skills/SKILL.md](../quality-skills/SKILL.md). The
pyramid, coverage policy, and detection rule are canonical in
[testing-principles](${CLAUDE_SKILL_DIR}/_shared/testing-principles.md) — this leaf
is the **how-to** and does not duplicate them.

## When to Use

Use this skill when:
- Generating or reviewing front-end tests for a project
- Choosing a test runner (do not introduce a second one)
- Writing component tests that assert user-visible behavior, not internals
- Adding e2e coverage for a critical journey
- Defining/checking the testing completion gate

## Rule 1: Detect the framework — never assume (binding)

Before writing or running a single test, **detect the project's existing stack**
and match it. Introducing a second test framework is a defect.

| Look for | Runner / library in use |
|----------|--------------------------|
| `vitest` in deps, `vitest.config.*`, `import.meta.vitest` | **Vitest** (unit/component) |
| `jest` in deps, `jest.config.*`, `babel-jest`/`ts-jest` | **Jest** (unit/component) |
| `@testing-library/{react,vue,svelte,angular}` | **Testing Library** (component layer) |
| `@playwright/test`, `playwright.config.*` | **Playwright** (e2e) |
| `cypress` in deps, `cypress.config.*` | **Cypress** (e2e) |
| `vite` present, no test runner | default to **Vitest** (Vite-native) |

> Vitest tracks the installed Vite major and shares its config/transform. Fallback when there is no Vite toolchain: Jest with `ts-jest`/`babel-jest`. Canonical: _shared/version-feature-matrix.md

Detection order: read `package.json` deps + scripts, then config files, then an
existing `*.test.*`/`*.spec.*` for the established conventions. Mirror them.

## Pyramid layers (what to test where)

The canonical pyramid and weighting live in
[testing-principles](${CLAUDE_SKILL_DIR}/_shared/testing-principles.md); the
front-end shape is unit-heavy at the base with a substantial **component** middle.

| Layer | Tool | Test… |
|-------|------|-------|
| Unit | Vitest / Jest | pure functions, hooks, stores, reducers, utils |
| Component | Testing Library (+ Vitest/Jest) | rendered output + user interaction, a11y |
| E2E | Playwright / Cypress | critical journeys across real routes/network |

## Component tests: assert behavior, not implementation

Testing Library's core principle — **query the DOM the way a user (or assistive
tech) does**, by role/label/text, not by test-ids or component internals.

```ts
import { render, screen } from '@testing-library/react';   // or /vue, /svelte, /angular
import userEvent from '@testing-library/user-event';
import { axe } from 'vitest-axe';

test('submits the form and shows confirmation', async () => {
  const user = userEvent.setup();
  render(<SignupForm />);

  await user.type(screen.getByLabelText(/email/i), 'a@b.com');   // by accessible label
  await user.click(screen.getByRole('button', { name: /sign up/i })); // by role + name

  expect(await screen.findByText(/check your inbox/i)).toBeInTheDocument();
  expect(await axe(document.body)).toHaveNoViolations();          // a11y assertion in the test
});
```

- **Query priority:** `getByRole` > `getByLabelText` > `getByText` > … `getByTestId`
  (last resort). Role/label queries double as accessibility checks — if you can't
  query by role, the component probably isn't accessible (see
  [accessibility-patterns](../accessibility-patterns/SKILL.md)).
- **`userEvent` over `fireEvent`** — it simulates real user interaction (focus,
  key sequences) more faithfully.
- **Avoid asserting on state/props/instance internals** — they break on refactor
  without catching real regressions.

## E2E with Playwright

```ts
import { test, expect } from '@playwright/test';

test('checkout happy path', async ({ page }) => {
  await page.goto('/cart');
  await page.getByRole('button', { name: /checkout/i }).click();
  await page.getByLabel(/card number/i).fill('4242 4242 4242 4242');
  await page.getByRole('button', { name: /pay/i }).click();
  await expect(page.getByText(/order confirmed/i)).toBeVisible();   // auto-waits
});
```

- Reserve e2e for **critical journeys** (auth, checkout, core CRUD) — they're slow
  and flaky in bulk; the pyramid keeps them few.
- Playwright auto-waits and retries assertions; **avoid hard `waitForTimeout`** —
  wait on a condition/role instead. Use role/label locators for resilience.
- Run against a production-like build for trustworthy results.

## Coverage and the gate

- **Coverage thresholds** are configured in the runner (`vitest --coverage`,
  Jest `coverageThreshold`). The policy (per-file vs global, thresholds) is
  canonical in [testing-principles](${CLAUDE_SKILL_DIR}/_shared/testing-principles.md).
- **The testing gate passes** when: the suite is green, configured coverage
  thresholds are met, and `fe-test-generator` filled detected gaps. Record the test
  transcript path under `.context/logs/` in DV Build Evidence.
- Run tests as single scoped commands: `npx vitest run`, `npx playwright test` —
  never `&&`-chained.

## Mocking discipline

- **Network:** prefer MSW (Mock Service Worker) to intercept at the network layer
  over mocking `fetch`/the data lib — tests exercise more real code.
- **Mock at boundaries** (network, time, randomness), not your own modules.
- Use fake timers (`vi.useFakeTimers()`) for debounce/throttle; restore after.

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| Adding Jest to a Vitest project (or vice versa) | detect and match the existing runner |
| `getByTestId` everywhere | `getByRole`/`getByLabelText` (also tests a11y) |
| Asserting on component state/props | assert user-visible output |
| `waitForTimeout(1000)` in e2e | wait on a role/condition |
| Mocking your own modules | mock at network/time boundaries (MSW) |
| Only e2e, no unit/component | rebalance to the pyramid |

## Related Skills

- [testing-principles](${CLAUDE_SKILL_DIR}/_shared/testing-principles.md) — canonical pyramid, coverage policy, detection rule
- [accessibility-patterns](../accessibility-patterns/SKILL.md) — role/label queries double as a11y checks
- [quality-skills/SKILL.md](../quality-skills/SKILL.md) — canonical domain selection table
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — Vitest/Vite version coupling
