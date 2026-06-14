---
name: styling-skills
description: >-
  Styling skills navigation for modern web CSS — container queries, :has,
  cascade layers, subgrid, Tailwind design systems, and responsive/accessible
  CSS. Use when writing or reviewing CSS, choosing a layout primitive, building
  a design-token system, picking utility-first vs authored CSS, or ensuring a
  responsive layout meets accessibility floors.
---

# Styling Skills

**Canonical selection table and capability snapshot for modern web styling.**
This is *the* selection table for the `styling/` domain — every leaf below links
back here and does not duplicate it.

## Capability Snapshot

| Capability | One line |
|------------|----------|
| Container queries | Size components against their container, not the viewport (`@container`, `cqi` units) |
| `:has()` | Parent/relational selection without JS class toggling |
| Cascade layers (`@layer`) | Explicit, predictable specificity ordering across reset/base/components/utilities |
| Subgrid | Child grids that align to a parent's tracks |
| Native nesting (`&`) | Preprocessor-free nesting in plain CSS |
| Tailwind / utility-first | Token-driven utility classes; design system in config |

Browser support shifts; for any feature you ship, verify against
[caniuse.com](https://caniuse.com) / [Baseline](https://web.dev/baseline) and link the
canonical [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md).

> Requires container queries and `:has()` (Baseline 2023; verify older-browser targets). Fallback: viewport media queries and a JS-toggled wrapper class. Canonical: _shared/version-feature-matrix.md

**Approach in one line:** layout with fl/grid + container queries; isolate
specificity with `@layer`; drive spacing/color/type from design tokens (custom
properties or Tailwind config); progressively enhance newer features behind
`@supports`.

## Skill Selection Guide

| I need to... | Use this skill |
|--------------|----------------|
| Use container queries, `:has()`, cascade layers, subgrid, native nesting | [modern-css/SKILL.md](../modern-css/SKILL.md) |
| Build a Tailwind design system (tokens, theme, dark mode, component layers) | [tailwind-design-system/SKILL.md](../tailwind-design-system/SKILL.md) |
| Make a layout responsive AND accessible (fluid type, reduced motion, focus, contrast) | [responsive-accessible-css/SKILL.md](../responsive-accessible-css/SKILL.md) |
| Audit a component against WCAG / ARIA | [accessibility-patterns](${CLAUDE_SKILL_DIR}/quality/accessibility-patterns/SKILL.md) |
| Check WCAG contrast / target-size floors | [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md) |
| Reduce CSS bundle / unused styles | [bundling-optimization](${CLAUDE_SKILL_DIR}/tooling/bundling-optimization/SKILL.md) |

## Decision Tree

```
Styling task?
├── Which feature is safe to ship? → version-feature-matrix + caniuse/Baseline
├── Layout / selectors / specificity → modern-css/SKILL.md
│   ├── component-relative sizing → container queries
│   ├── parent/relational state → :has()
│   ├── predictable cascade → @layer
│   └── grid alignment across nesting → subgrid
├── Utility-first / token config → tailwind-design-system/SKILL.md
├── Fluid + accessible responsive → responsive-accessible-css/SKILL.md
│   ├── contrast / target size / focus → accessibility-baseline (canonical)
│   └── reduced motion / forced colors → responsive-accessible-css
└── a11y audit of the result → quality/accessibility-patterns
```

## Domain Constraints (styling delta)

These augment the inherited `_base/frontend-agent.md` Constraints — they do not
restate or weaken them.

- **Specificity is managed with `@layer`, not `!important`.** Reserve `!important`
  for genuine overrides of third-party styles you cannot edit, and document why.
  > Requires cascade layers (`@layer`, Baseline 2022). Fallback: disciplined source ordering and low-specificity selectors. Canonical: _shared/version-feature-matrix.md
- **Design tokens, not magic numbers.** Spacing, color, type scale, radius, and
  motion come from CSS custom properties or the Tailwind/theme config — never
  hard-coded hex/px scattered across files.
- **Progressive enhancement:** gate newer features behind `@supports` (or feature
  queries) with a working fallback, so the layout degrades rather than breaks.
- **Accessibility is part of styling, not a separate pass:** contrast ≥ 4.5:1
  (text) / 3:1 (UI), visible focus indicators, `prefers-reduced-motion`, and
  target size ≥ 24×24px are styling responsibilities. See
  [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md).
- **No layout shift from late-loading styles/fonts** — reserve space, use
  `font-display: swap` deliberately, size media. CLS is a styling concern too
  (see [web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md)).

## File Overview

| File | Purpose |
|------|---------|
| [modern-css/SKILL.md](../modern-css/SKILL.md) | Container queries, `:has()`, cascade layers, subgrid, native nesting |
| [tailwind-design-system/SKILL.md](../tailwind-design-system/SKILL.md) | Token-driven Tailwind config, theming, dark mode, component layers |
| [responsive-accessible-css/SKILL.md](../responsive-accessible-css/SKILL.md) | Fluid type, reduced motion, forced colors, accessible responsive patterns |

## Related Skills

- [accessibility-patterns](${CLAUDE_SKILL_DIR}/quality/accessibility-patterns/SKILL.md) — ARIA, focus, keyboard (the audit side)
- [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md) — WCAG 2.2 contrast/target-size floors
- [web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md) — CLS, render-blocking CSS, font loading
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — canonical modern-CSS Baseline anchors
