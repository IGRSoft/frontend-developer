---
name: severity-matrix
description: Reusable severity and priority definitions (P0–P3) for frontend-developer review, with web-specific examples — XSS, broken build, accessibility violations, hydration errors, Core Web Vitals regressions, and style drift. Use when classifying review findings or setting remediation priority.
---

# Severity Matrix Reference (Front-End)

Shared definitions for severity levels, the P0–P3 review-finding scale, and
effort/impact assessments used by every frontend-developer agent's Review
response format and by the review-only auditors.

## Severity Levels

| Level | Description | Response Time | Web Examples |
|-------|-------------|---------------|--------------|
| Critical | Security, broken build, broken core flow | Immediate | XSS sink on user data, secret shipped in the bundle, `tsc`/build failure, app crashes on load |
| High | Major functionality, accessibility, hydration | Within sprint | WCAG-blocking a11y violation, hydration mismatch, missing CSP, broken keyboard navigation, INP/LCP over budget |
| Medium | Quality, minor performance, maintainability | Quarterly | `any` without justification, prop-drilling that should be context, unmemoized hot list, missing loading/error state |
| Low | Style, polish | Opportunistic | Class-name drift, formatting, naming, missing TSDoc on an exported component |

## Review Finding Priorities (P0–P3)

Used by the Review response format of all frontend-developer agents and by the
≤500-token findings summaries the review-only auditors (`fe-accessibility-auditor`,
`fe-performance-engineer`, `fe-security-auditor`) hand back. Each finding cites
`file:line`.

| Priority | Definition | Web Examples |
|----------|------------|--------------|
| P0 | Must fix before merge — correctness/security/build broken | XSS via `dangerouslySetInnerHTML`/`v-html`/`{@html}` on untrusted data; secret read in a client component; `eval`/`new Function`; `tsc --noEmit` or production build fails; React Hook called conditionally (rules-of-hooks violation); app throws on first render |
| P1 | Fix in this change — defect likely to bite | Hydration mismatch (SSR ≠ client markup); WCAG 2.2 Level A/AA violation (missing label, contrast < 4.5:1, focus not visible); missing/weak CSP; unhandled promise rejection; effect with a wrong/missing dependency array; `javascript:` href; missing `key` on a list; INP > 200ms / LCP > 2.5s / CLS > 0.1 over budget |
| P2 | Should fix — quality/maintainability | `any` without a justifying comment; unnecessary client component (should be a Server Component); unmemoized expensive computation in a hot path; duplicated component logic; missing loading/error/empty state; prop drilling that warrants context/store; non-semantic markup (`div` button) |
| P3 | Nice to have — style | Formatting/lint-autofixable drift, naming, comment polish, class-name ordering, missing TSDoc on a non-public component (auto-fixable via eslint/biome/prettier/stylelint) |

## Priority Matrix (impact × effort)

| Priority | Impact | Effort | Action |
|----------|--------|--------|--------|
| P0 | Critical | Any | Immediate remediation |
| P1 | High | Low | Do first (quick wins) |
| P2 | High | High | Plan and schedule |
| P3 | Medium | Low | Batch together |
| P4 | Low | High | Deprioritize or skip |

## Effort/Impact Quadrant

```
High Impact ┌──────────────┬──────────────┐
            │   SCHEDULE   │  DO FIRST    │
            │  (P2: Plan)  │ (P1: Quick)  │
            ├──────────────┼──────────────┤
            │    AVOID     │  FILL-INS    │
            │ (P4: Defer)  │ (P3: Batch)  │
Low Impact  └──────────────┴──────────────┘
             High Effort    Low Effort
```

## Code Smell Indicators

| Smell | Thresholds | Impact |
|-------|------------|--------|
| Large component | >250 lines / >1 responsibility | Hard to understand/test; split it |
| Deep prop drilling | >3 levels passing the same prop | Use context/store |
| `useEffect` overuse | effect for derived state or event logic | Derive in render / handle in the event |
| Inline anonymous objects/functions in JSX hot paths | per-render allocation in a large list | Hoist / memoize |
| Nesting depth (JSX or selectors) | >4 levels | Reduced readability |
| Props per component | >8 | Group into an object / split component |
| Bundle weight of one route chunk | >150KB gzipped *(verify budget)* | Code-split / lazy-load |
| `any` / `@ts-ignore` density | any unjustified use | Type it or justify inline |
| Duplicated component/style code | >5% | Extract a shared component/utility |

## Accessibility Gate (a11y findings map here)

`fe-accessibility-auditor` findings use the same P0–P3 scale; the gate maps WCAG
conformance levels to priority:

| WCAG level | Default priority | Example |
|------------|------------------|---------|
| Level A failure | P1 (P0 if it blocks the core task) | No keyboard operability; missing form label; no alt text on a content image |
| Level AA failure | P1 | Contrast < 4.5:1 (normal text); focus not visible; target size below 24×24 (2.2) |
| Level AAA / best-practice | P2–P3 | Enhanced contrast; redundant landmark; nicer focus order |

See `accessibility-baseline.md` for the full WCAG 2.2 success-criteria baseline
and the a11y-gate definition.

## Coverage Requirements

| Scope | Minimum | Target |
|-------|---------|--------|
| Critical UI flows (auth, checkout, forms with validation) | 90% | 95%+ |
| Components with logic (state, conditionals, effects) | 75% | 80%+ |
| Presentational components / utilities | 60% | 70%+ |
| Generated / config / pure-style code | N/A | N/A |

## Usage

Reference this file in commands and agents using:
```markdown
See: skills/_shared/severity-matrix.md for severity definitions
```

## Related Skills

- `accessibility-baseline.md` — WCAG 2.2 baseline and the a11y-gate definition (P1 findings)
- `secure-coding/SKILL.md` — the security findings that map to P0/P1
- `testing-principles.md` — coverage thresholds and the QA gate
- `workflow-integration/SKILL.md` — DR/QA gate definitions for worktask runs
