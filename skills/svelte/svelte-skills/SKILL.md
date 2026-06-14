---
name: svelte-skills
description: >-
  Svelte 5 and SvelteKit skills navigation: runes reactivity (`$state`,
  `$derived`, `$effect`, `$props`, `$bindable`), snippets and event attributes,
  and SvelteKit application architecture (routing, `load` functions, form
  actions, server vs universal code). Use when writing or reviewing Svelte 5
  components, migrating from Svelte 4, or building a SvelteKit app. Canonical
  selection table for the svelte domain.
---

# Svelte Skills

**Navigation and the canonical selection table for Svelte 5 + SvelteKit**

This is the **canonical selection table** for the svelte domain. The two leaves
below carry topic depth and link back here — they never restate this table.

## Version Snapshot

| Version | Headline (one line) |
|---------|---------------------|
| 4 | `let` + `$:` reactive statements, `export let` props, `<slot>`, `on:click` directives |
| 5 | Runes (`$state`/`$derived`/`$effect`/`$props`/`$bindable`), snippets (`{#snippet}`/`{@render}`), `onclick` event attributes, fine-grained signals-based reactivity |

Feature landings shift between minor releases — verify against the project's
`svelte`/`@sveltejs/kit` versions and the canonical
[version-feature-matrix](../../_shared/version-feature-matrix.md).

> Requires Svelte 5 runes (`$state`/`$derived`/`$effect`). Fallback: Svelte 4 `$:` reactive statements and `export let` props. Canonical: _shared/version-feature-matrix.md

**Stack in one line:** Svelte 5 runes for component reactivity, **SvelteKit** for
routing/SSR/data-loading/form-actions, Vite under the hood. TypeScript-first
([ts-typing](../../typescript/ts-typing/SKILL.md)).

## Skill Selection Guide

| I need to... | Use this skill |
|--------------|----------------|
| Write Svelte 5 runes, migrate from `$:`/`export let`, use snippets | [svelte-runes/SKILL.md](../svelte-runes/SKILL.md) |
| Declare props with `$props`, two-way with `$bindable` | [svelte-runes/SKILL.md](../svelte-runes/SKILL.md) |
| Fix an `$effect` loop or a lost-reactivity bug | [svelte-runes/SKILL.md](../svelte-runes/SKILL.md) |
| Build routes, `load` functions, layouts | [sveltekit/SKILL.md](../sveltekit/SKILL.md) |
| Handle forms with progressive-enhancement actions | [sveltekit/SKILL.md](../sveltekit/SKILL.md) |
| Separate server-only from universal code, read env safely | [sveltekit/SKILL.md](../sveltekit/SKILL.md) |
| Confirm which Svelte/Kit version a feature needs | [version-feature-matrix](../../_shared/version-feature-matrix.md) (canonical) |

## Decision Tree

```
Svelte task?
├── Which version has feature X? → version-feature-matrix (canonical)
├── Component reactivity / props / events → svelte-runes/SKILL.md
│   ├── state/derived/effect → svelte-runes (runes)
│   ├── reusable markup → snippets {#snippet}/{@render} → svelte-runes
│   └── migrating Svelte 4 → svelte-runes (migration table)
└── App-level routing / data / forms → sveltekit/SKILL.md
    ├── page data → load function (server vs universal) → sveltekit
    ├── mutation → form action + use:enhance → sveltekit
    └── secrets/env → $env modules → sveltekit
```

## Non-Negotiables (apply to every Svelte task)

- **Runes for new code** (Svelte 5). Don't mix `$:` reactive statements with runes in the same component.
- **`$effect` is a last resort** — prefer `$derived` for computed values; effects are for synchronizing with external systems, and must not set state they depend on (infinite loop).
- **Server secrets never reach the client** — use `$env/static/private` / `$env/dynamic/private` only in server code (see [sveltekit](../sveltekit/SKILL.md)).
- **Template a11y lint clean** (Svelte's built-in a11y warnings are errors here); semantic elements over `<div onclick>`.

## File Overview

| File | Purpose |
|------|---------|
| [svelte-runes/SKILL.md](../svelte-runes/SKILL.md) | Runes, snippets, event attributes, Svelte 4 → 5 migration |
| [sveltekit/SKILL.md](../sveltekit/SKILL.md) | Routing, `load`, form actions, server/universal split, env |

## Related Skills

- [typescript-skills](../../typescript/typescript-skills/SKILL.md) — typed `$props`, `load` return types
- [modern-css](../../styling/modern-css/SKILL.md) — scoped styles and modern CSS in `.svelte` files
- [fe-testing](../../quality/fe-testing/SKILL.md) — Vitest + Testing Library / Playwright for Svelte
- [secure-coding](../../_shared/secure-coding/SKILL.md) — `{@html}` XSS, env-secret boundaries
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical Svelte / SvelteKit version minimums
