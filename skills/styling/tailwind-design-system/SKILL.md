---
name: tailwind-design-system
description: >-
  Building a token-driven design system with Tailwind — theme/token config,
  CSS-variable theming for light/dark, component vs utility layering, and
  avoiding utility-soup with extracted patterns. Use when setting up Tailwind
  tokens, implementing dark mode, deciding utility-first vs @apply/component
  classes, or reviewing a Tailwind codebase for design-system consistency.
---

# Tailwind Design System

**Token-driven utility-first styling.** For the domain selection table, see the
canonical [styling-skills/SKILL.md](../styling-skills/SKILL.md) — this leaf carries
depth and does not duplicate it.

## When to Use

Use this skill when:
- Standing up a Tailwind design system: tokens, theme, spacing/type scale
- Implementing light/dark (or multi-) theming
- Deciding between inline utilities, extracted component classes, and `@apply`
- Reviewing a Tailwind codebase for consistency (one source of truth for tokens)

## Tokens are the single source of truth

A design system in Tailwind lives in **the theme/token config**, not scattered
arbitrary values. Drive everything (color, spacing, radius, type, shadow) from
named tokens so a change is one edit.

```js
// tailwind.config — tokens, not magic values
export default {
  theme: {
    extend: {
      colors: {
        // bind to CSS variables so themes can swap them at runtime
        bg:      'rgb(var(--color-bg) / <alpha-value>)',
        fg:      'rgb(var(--color-fg) / <alpha-value>)',
        accent:  'rgb(var(--color-accent) / <alpha-value>)',
      },
      borderRadius: { card: 'var(--radius-card)' },
      spacing: { gutter: 'var(--space-gutter)' },
    },
  },
};
```

**Rule:** prefer `text-fg`, `bg-bg`, `rounded-card` over arbitrary values like
`text-[#1a1a1a]` or `rounded-[11px]`. Arbitrary values are an escape hatch, not the
norm — each one is a token that escaped the system. Tailwind v4 moves token config
out of `tailwind.config.js` and into CSS via the `@theme` directive; either way the
tokens stay centralized.

> Requires the CSS-first `@theme` token config (Tailwind v4+). Fallback: the JS `tailwind.config.js` `theme.extend` block (Tailwind v3). Canonical: _shared/version-feature-matrix.md

## Theming with CSS variables (light / dark)

Bind Tailwind colors to CSS custom properties, then swap the variables — utilities
never change, only the variable values do:

```css
:root            { --color-bg: 255 255 255; --color-fg: 17 17 17; --color-accent: 37 99 235; }
:root.dark,
[data-theme=dark] { --color-bg: 17 17 17; --color-fg: 240 240 240; --color-accent: 96 165 250; }

@media (prefers-color-scheme: dark) {
  :root:not([data-theme=light]) { /* same dark token values */ }
}
```

```html
<div class="bg-bg text-fg">…</div>   <!-- adapts to whichever theme is active -->
```

This respects `prefers-color-scheme` by default while allowing an explicit override
via `data-theme` / a `.dark` class (Tailwind's `darkMode: 'selector'`). The
contrast of every token pair must still clear the WCAG floor — see
[accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md).

## Utility-first vs component classes — when to extract

Stay inline until repetition or readability forces extraction:

| Situation | Approach |
|-----------|----------|
| One-off layout | inline utilities |
| Same cluster repeated 3+ times | extract a **component** (React/Vue/Svelte) — let the framework dedupe, not CSS |
| Framework-less / shared widget | a component class with `@apply` of tokens (sparingly) |
| Conditional variants | a variants helper (`cva`/`tailwind-variants`), not string concatenation |

**Prefer extracting a UI component over `@apply`.** `@apply` re-introduces the
indirection Tailwind removes; reserve it for genuinely framework-agnostic shared
classes. Never build long `@apply` chains that recreate a CSS framework.

## Variants without string soup

```ts
import { cva } from 'class-variance-authority';

const button = cva('inline-flex items-center rounded-card font-medium', {
  variants: {
    intent: { primary: 'bg-accent text-white', ghost: 'bg-transparent text-fg' },
    size:   { sm: 'h-8 px-3 text-sm', md: 'h-10 px-4' },
  },
  defaultVariants: { intent: 'primary', size: 'md' },
});
// button({ intent: 'ghost', size: 'sm' })
```

Centralizes variant logic and keeps class strings out of conditionals.

## Consistency review checklist

- Arbitrary values (`-[…]`) are rare and justified — most are missing tokens.
- Color/spacing/radius/type come from the theme, not literals.
- Dark mode swaps variables, not duplicated utility sets.
- Repeated clusters are components or `cva` variants, not copy-paste.
- Content config covers all template globs (else classes get purged in prod).
- Focus-visible utilities present on interactive elements (a11y).

## Build / purge note

Tailwind's content scanning tree-shakes unused utilities at build time — ensure
the `content` globs cover every file that emits classes, or production strips
classes you actually use. Dynamic class names must be safelisted or fully spelled
out (never build class names by string concatenation of runtime values). See
[bundling-optimization](${CLAUDE_SKILL_DIR}/tooling/bundling-optimization/SKILL.md).

## Related Skills

- [styling-skills/SKILL.md](../styling-skills/SKILL.md) — canonical domain selection table
- [modern-css/SKILL.md](../modern-css/SKILL.md) — `@layer` (Tailwind uses cascade layers internally), container queries
- [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md) — token contrast and focus floors
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — CSS-variable theming Baseline notes
