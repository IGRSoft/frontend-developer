---
name: responsive-accessible-css
description: >-
  Responsive layouts that are accessible by construction — fluid type with
  clamp(), intrinsic responsive grids, prefers-reduced-motion, forced-colors,
  visible focus, target size, and zoom/reflow. Use when building responsive CSS,
  honoring user motion/contrast preferences, ensuring layouts survive 200% zoom,
  or reviewing CSS for WCAG 2.2 layout criteria.
---

# Responsive & Accessible CSS

**Responsive layout that clears accessibility floors by construction.** For the
domain selection table, see the canonical
[styling-skills/SKILL.md](../styling-skills/SKILL.md) — this leaf carries depth and
does not duplicate it. WCAG success criteria themselves are canonical in
[accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md);
this skill is the CSS *how-to* for the layout-related ones.

## When to Use

Use this skill when:
- Building responsive layouts (fluid type, intrinsic grids, breakpoints)
- Honoring user preferences: reduced motion, forced/high contrast, color scheme
- Ensuring a layout survives 200% zoom and 320px reflow without horizontal scroll
- Implementing visible focus and adequate target size
- Reviewing CSS against WCAG 2.2 layout-related criteria

## Fluid type and space with clamp()

```css
:root {
  /* min, preferred (viewport-scaled), max — no media query needed */
  --step-0: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
  --step-1: clamp(1.25rem, 1.1rem + 0.8vw, 1.75rem);
}
h2 { font-size: var(--step-1); }
```

`clamp()` scales smoothly between a floor and ceiling — fewer breakpoints, no
sudden jumps. **Always set the min in `rem`** so text still respects the user's
font-size and survives 200% zoom (WCAG 1.4.4 Resize Text). Never lock font size in
`px` only.

## Intrinsic responsive layout (no/few breakpoints)

```css
/* wrap-when-needed grid: each card is ≥ 16rem, fills the row, wraps automatically */
.grid { display: grid; gap: 1rem; grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr)); }

/* fluid sidebar that collapses below the content's needs */
.with-sidebar { display: flex; flex-wrap: wrap; gap: 1rem; }
.with-sidebar > .sidebar { flex: 1 1 16rem; }
.with-sidebar > .main    { flex: 999 1 60%; }
```

Prefer **intrinsic** patterns (`auto-fit`/`minmax`, `flex-wrap`) over a ladder of
`@media` breakpoints — they reflow at any width and satisfy WCAG 1.4.10 Reflow
(no horizontal scroll at 320px-equivalent). Add container queries for
component-level adaptation (see [modern-css](../modern-css/SKILL.md)).

## Honor user preferences

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;   /* near-instant; don't fully remove (some rely on end events) */
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

@media (forced-colors: active) {
  /* respect the user's color set; don't override system colors. Use system color keywords. */
  .btn { border: 1px solid ButtonText; }
  /* ensure focus is visible via forced-color-adjust where needed */
}
```

`prefers-reduced-motion` is **mandatory** for non-essential motion (WCAG 2.3.3 /
2.2.2). `forced-colors` (Windows High Contrast) must not be fought — set borders
with system color keywords so controls stay distinguishable. `prefers-color-scheme`
drives default theming (see [tailwind-design-system](../tailwind-design-system/SKILL.md)).

> Reduced-motion / forced-colors / color-scheme media features are broadly supported (Baseline). Fallback: none needed — they are progressive (unsupported browsers simply get the default rules). Canonical: _shared/version-feature-matrix.md

## Visible focus and target size

```css
:focus-visible {
  outline: 2px solid var(--focus, AccentColor);
  outline-offset: 2px;
}
.icon-button { min-block-size: 24px; min-inline-size: 24px; }   /* WCAG 2.5.8 Target Size (Minimum) */
```

- **Never remove focus outlines without a replacement.** Use `:focus-visible` so
  the indicator shows for keyboard users without bothering mouse users; the
  indicator must clear 3:1 contrast (WCAG 1.4.11 / 2.4.13 Focus Appearance).
- **Interactive targets ≥ 24×24 CSS px** (WCAG 2.2 2.5.8), with adequate spacing —
  pad small icon buttons rather than relying on the glyph size.

## Contrast and content scaling

- Text contrast ≥ **4.5:1** (normal), **3:1** (large ≥ 24px or 18.66px-bold); UI
  component / graphic contrast ≥ **3:1** (WCAG 1.4.3 / 1.4.11). Validate token
  pairs; do not eyeball.
- Layout must survive **200% zoom** and **400% / 320px reflow** without clipping or
  horizontal scroll — avoid fixed-px widths/heights on text containers; use
  `min()`/`max()`/`clamp()` and logical properties.

The numeric floors are canonical in
[accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md).

## Logical properties for internationalization

Prefer logical properties (`margin-inline`, `padding-block`, `inset-inline-start`)
over physical (`margin-left`) so layouts mirror correctly in RTL languages without
duplicate stylesheets.

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| `font-size` in `px` only | `rem`-based `clamp()` floor (survives zoom) |
| `outline: none` with no replacement | `:focus-visible` with a contrasting outline |
| Animations ignoring user preference | wrap non-essential motion in `prefers-reduced-motion` |
| Fixed-width text containers | `max-inline-size` in `ch`/`rem`; reflow-safe |
| 16px icon-only tap target | pad to ≥ 24×24 CSS px |
| Physical margins breaking RTL | logical properties (`margin-inline`) |

## Related Skills

- [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md) — canonical WCAG 2.2 floors (contrast, reflow, target size)
- [accessibility-patterns](${CLAUDE_SKILL_DIR}/quality/accessibility-patterns/SKILL.md) — ARIA, focus management, keyboard (behavior side)
- [modern-css/SKILL.md](../modern-css/SKILL.md) — container queries for component-level responsiveness
- [styling-skills/SKILL.md](../styling-skills/SKILL.md) — canonical domain selection table
- [web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md) — CLS from late styles/fonts
