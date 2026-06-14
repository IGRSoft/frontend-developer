---
name: modern-css
description: >-
  Modern CSS layout and selection with explicit Baseline gates — container
  queries (@container, cqi/cqw), :has() relational selection, cascade layers
  (@layer), subgrid, and native nesting. Use when building component-relative
  layouts, replacing JS class-toggling with :has(), taming specificity with
  @layer, aligning nested grids, or checking which feature is safe to ship.
---

# Modern CSS

**Baseline-gated layout and selection features with explicit fallbacks.** For the
domain selection table, see the canonical
[styling-skills/SKILL.md](../styling-skills/SKILL.md) — this leaf carries depth and
does not duplicate it.

## When to Use

Use this skill when:
- A component must size against its **container**, not the viewport
- You want **parent/relational** styling without toggling a class in JS
- Cascade specificity is fighting you and you need predictable ordering
- A nested grid must **align to its parent's tracks**
- You are checking whether a CSS feature is safe to ship to your target browsers

## Per-Feature Baseline Gate

Pick the feature only if your target browsers support it; otherwise use the
fallback. Canonical anchors live in
[version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md);
always confirm against [caniuse.com](https://caniuse.com) for your real audience.

| Feature | Baseline anchor | Pre-support fallback |
|---------|-----------------|----------------------|
| Container queries (`@container`, `cqi`/`cqw`) | Baseline 2023 (widely available) | viewport media queries; JS `ResizeObserver` |
| `:has()` relational selector | Baseline 2023 (widely available) | a wrapper class toggled in JS |
| Cascade layers (`@layer`) | Baseline 2022 (widely available) | source ordering + specificity discipline |
| Subgrid (`grid-template-*: subgrid`) | Baseline 2023 (widely available) | nested explicit grids with shared track sizes |
| Native nesting (`&`) | Baseline 2023 (widely available) | Sass / PostCSS nesting plugin |
| `@property` (typed custom properties) | Baseline 2024 | untyped `--var` + JS-driven transitions |
| `color-mix()` / relative color | `color-mix()` Baseline 2023; relative color Baseline 2024 *(verify)* | precomputed tokens / Sass color functions |

> Requires container queries and `:has()` (Baseline 2023, now widely available; verify legacy targets). Fallback: viewport media queries and a JS-toggled wrapper class. Canonical: _shared/version-feature-matrix.md

## Container Queries — size by container, not viewport

```css
.card-list { container-type: inline-size; container-name: cards; }

.card { display: grid; gap: .5rem; }

@container cards (min-width: 28rem) {     /* responds to the container width, reusable anywhere */
  .card { grid-template-columns: 8rem 1fr; }
}
```

`cqi`/`cqw` (container query inline/width) units size relative to the query
container. This makes a component **portable** — it adapts wherever it is placed,
not just at fixed viewport breakpoints.

> Requires container queries (`@container`, Baseline 2023, widely available). Fallback: viewport `@media` queries or a `ResizeObserver`-driven class. Canonical: _shared/version-feature-matrix.md

## `:has()` — relational selection without JS

```css
.field:has(input:invalid) { border-color: var(--danger); }   /* parent reacts to child state */
.card:has(> img) { padding-block-start: 0; }                  /* style based on what it contains */
label:has(+ input:focus) { color: var(--accent); }            /* style preceding sibling on focus */
```

Replaces a whole category of JS that toggled classes based on descendant state.
Behind feature detection:

```css
@supports selector(:has(*)) { /* enhanced rules */ }
```

> Requires `:has()` (Baseline 2023, widely available). Fallback: toggle a wrapper class in JS based on the same condition. Canonical: _shared/version-feature-matrix.md

## Cascade Layers — predictable specificity

```css
@layer reset, base, components, utilities;   /* declare order ONCE — later layers win */

@layer base       { a { color: var(--link); } }
@layer components { .btn { color: white; background: var(--accent); } }
@layer utilities  { .text-muted { color: var(--muted); } }   /* always beats components */
```

Layer order is decided by the `@layer` declaration, **not** selector specificity —
so a single-class utility in a later layer beats a more specific component selector
without `!important`. Third-party CSS can be wrapped: `@import "vendor.css" layer(vendor);`.

> Requires cascade layers (`@layer`, Baseline 2022). Fallback: disciplined source ordering and low-specificity selectors; avoid `!important`. Canonical: _shared/version-feature-matrix.md

## Subgrid — align nested grids to parent tracks

```css
.grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 1rem; }

.card { display: grid; grid-template-columns: subgrid; grid-column: span 3; }
/* card's children now align to the parent's 3 columns — e.g. ragged card content lines up */
```

> Requires subgrid (Baseline 2023, widely available). Fallback: nested explicit grids that repeat the parent's track sizes (kept in sync manually). Canonical: _shared/version-feature-matrix.md

## Native nesting (&)

```css
.card {
  padding: 1rem;
  & .title { font-weight: 600; }
  &:hover  { box-shadow: var(--shadow); }
  @media (min-width: 40rem) { & { padding: 2rem; } }
}
```

The `&` is required in many positions for correct parsing; prefer explicit `&` to
avoid ambiguity with type selectors.

> Requires native CSS nesting (Baseline 2023, widely available). Fallback: Sass or the PostCSS nesting plugin. Canonical: _shared/version-feature-matrix.md

## Feature-detect, then enhance

Gate every newer feature so the page degrades rather than breaks:

```css
.layout { /* solid fallback: media-query grid */ }
@supports (container-type: inline-size) {
  .layout { /* container-query enhancement */ }
}
```

Prefer `@supports` / `CSS.supports(...)` over browser/version sniffing — see usage
rule 1 in [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md).

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| `!important` to win specificity | put the rule in a later `@layer` |
| JS class toggling for descendant state | `:has()` (behind `@supports`) |
| Fixed viewport breakpoints for reusable components | container queries |
| Magic px/hex scattered in files | design tokens (custom properties) |
| Shipping a new feature with no fallback | wrap in `@supports` with a working base |

## Related Skills

- [styling-skills/SKILL.md](../styling-skills/SKILL.md) — canonical domain selection table
- [responsive-accessible-css/SKILL.md](../responsive-accessible-css/SKILL.md) — fluid type, reduced motion, accessible responsive
- [tailwind-design-system/SKILL.md](../tailwind-design-system/SKILL.md) — token config and utility layers
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — canonical modern-CSS Baseline anchors
