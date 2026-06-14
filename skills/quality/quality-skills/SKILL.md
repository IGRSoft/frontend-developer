---
name: quality-skills
description: >-
  Quality skills navigation for front-end — accessibility patterns (ARIA, focus,
  keyboard), web performance (Core Web Vitals budgets), and testing
  (Vitest/Playwright/Testing Library with framework detection). Use when
  auditing accessibility, setting or checking a performance budget, choosing a
  test framework, or defining the QA completion gate.
---

# Quality Skills

**Canonical selection table for front-end quality — a11y, performance, testing.**
This is *the* selection table for the `quality/` domain — every leaf below links
back here and does not duplicate it. These three areas together define the **QA
completion gate**: tests pass AND axe-clean AND the Lighthouse/Core-Web-Vitals
budget is met.

## Quality Gate Snapshot

| Dimension | Floor | Canonical source |
|-----------|-------|------------------|
| Accessibility | WCAG 2.2 AA; zero axe-detectable A/AA violations | [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md) |
| Performance | LCP < 2.5s, INP < 200ms, CLS < 0.1 | [web-performance](../web-performance/SKILL.md) |
| Testing | suite green; coverage thresholds met; framework-detected runner | [testing-principles](${CLAUDE_SKILL_DIR}/_shared/testing-principles.md) |

## Skill Selection Guide

| I need to... | Use this skill |
|--------------|----------------|
| Implement/audit ARIA, focus management, keyboard nav | [accessibility-patterns/SKILL.md](../accessibility-patterns/SKILL.md) |
| Set or check a Core Web Vitals budget; profile with Lighthouse | [web-performance/SKILL.md](../web-performance/SKILL.md) |
| Write tests with framework detection (Vitest/Playwright/Testing Library) | [fe-testing/SKILL.md](../fe-testing/SKILL.md) |
| Look up the WCAG success-criteria floors | [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md) |
| Look up the test pyramid / coverage policy | [testing-principles](${CLAUDE_SKILL_DIR}/_shared/testing-principles.md) |
| Map a finding to a severity (P0–P3) | [severity-matrix](${CLAUDE_SKILL_DIR}/_shared/severity-matrix.md) |

## Decision Tree

```
Quality task?
├── Accessibility → accessibility-patterns/SKILL.md
│   ├── numeric WCAG floors → accessibility-baseline (canonical)
│   └── CSS layout a11y (contrast, focus, motion) → styling/responsive-accessible-css
├── Performance → web-performance/SKILL.md
│   ├── budget targets (LCP/INP/CLS) → here (canonical for FE budgets)
│   └── bundle size driving the budget → tooling/bundling-optimization
├── Testing → fe-testing/SKILL.md
│   ├── pyramid / coverage policy → testing-principles (canonical)
│   └── framework detection first → fe-testing
└── Severity of a finding → _shared/severity-matrix
```

## Domain Constraints (quality delta)

These augment the inherited `_base/frontend-agent.md` Mandatory Requirements — they
do not restate or weaken them.

- **The a11y gate is a completion-blocker.** An axe-detectable or audit-confirmed
  WCAG A/AA failure is at least P1 (P0 when it blocks the core task). Review-only
  agents surface findings; fixes route to `frontend-developer:fe-code-fixer`.
- **Performance has a budget, not a vibe.** Ship against LCP < 2.5s / INP < 200ms /
  CLS < 0.1 measured at the 75th percentile; record lab numbers in DV Build Evidence.
- **Tests are framework-detected, never assumed.** Detect the project's runner from
  config/deps before writing or running tests — do not introduce a second framework.
- **Quality evidence is screenshot + report.** DV captures screens via the
  `web_adapter` path and attaches Lighthouse/axe as *supporting* rows — see
  [workflow-integration](${CLAUDE_SKILL_DIR}/_shared/workflow-integration/SKILL.md).

## File Overview

| File | Purpose |
|------|---------|
| [accessibility-patterns/SKILL.md](../accessibility-patterns/SKILL.md) | ARIA patterns, focus management, keyboard interaction, axe-core |
| [web-performance/SKILL.md](../web-performance/SKILL.md) | Core Web Vitals budgets, Lighthouse, field vs lab data |
| [fe-testing/SKILL.md](../fe-testing/SKILL.md) | Vitest/Playwright/Testing Library + framework-detection logic |

## Related Skills

- [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md) — canonical WCAG 2.2 floors
- [testing-principles](${CLAUDE_SKILL_DIR}/_shared/testing-principles.md) — canonical pyramid + coverage policy
- [severity-matrix](${CLAUDE_SKILL_DIR}/_shared/severity-matrix.md) — P0–P3 classification
- [responsive-accessible-css](${CLAUDE_SKILL_DIR}/styling/responsive-accessible-css/SKILL.md) — CSS side of a11y
- [bundling-optimization](${CLAUDE_SKILL_DIR}/tooling/bundling-optimization/SKILL.md) — bundle size → perf budget
