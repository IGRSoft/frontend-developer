---
name: vue-skills
description: >-
  Vue 3.5 language and framework skills navigation: the Composition API with
  `<script setup>` (refs vs reactive, composables, `defineModel`, reactivity
  rules) and Pinia state management (setup vs options stores, getters, actions,
  SSR-safe state). Use when writing or reviewing Vue 3 single-file components,
  building composables, choosing refs vs reactive, or structuring Pinia stores.
  Canonical selection table for the vue domain.
---

# Vue Skills

**Navigation and the canonical selection table for Vue 3.5**

This is the **canonical selection table** for the vue domain. The two leaves
below carry topic depth and link back here — they never restate this table.

## Version Snapshot

| Version | Headline (one line) |
|---------|---------------------|
| 3.0 | Composition API, `<script setup>`, Proxy-based reactivity, fragments/teleport |
| 3.4 | `defineModel()` two-way binding macro; faster parser |
| 3.5 | reactive props destructure (compile-time), `useId()`, `useTemplateRef()`, lower memory |

Feature landings shift between minor releases — verify against the project's
`vue` version and the canonical
[version-feature-matrix](../../_shared/version-feature-matrix.md).

> Requires Vue 3.5 reactive props destructure / `useId`. Fallback: `toRefs(props)` and manual ids on Vue 3.4. Canonical: _shared/version-feature-matrix.md

**Stack in one line:** Vue 3.5 + `<script setup>` Composition API (Options API
only for legacy), **Pinia** for shared state, Vite/Nuxt 3 for the build, Vue
Router for routing. TypeScript-first ([ts-typing](../../typescript/ts-typing/SKILL.md)).

## Skill Selection Guide

| I need to... | Use this skill |
|--------------|----------------|
| Write `<script setup>`, choose refs vs reactive, build a composable | [vue-composition/SKILL.md](../vue-composition/SKILL.md) |
| Use `defineModel`, `defineProps`/`defineEmits`, `useTemplateRef` | [vue-composition/SKILL.md](../vue-composition/SKILL.md) |
| Fix a reactivity-loss bug (destructured ref, broken `watch`) | [vue-composition/SKILL.md](../vue-composition/SKILL.md) |
| Structure a Pinia store, choose setup vs options store | [vue-state/SKILL.md](../vue-state/SKILL.md) |
| Make global state SSR-safe (Nuxt) | [vue-state/SKILL.md](../vue-state/SKILL.md) |
| Confirm which Vue version a feature needs | [version-feature-matrix](../../_shared/version-feature-matrix.md) (canonical) |

## Decision Tree

```
Vue task?
├── Which version has feature X? → version-feature-matrix (canonical)
├── Component logic / reactivity → vue-composition/SKILL.md
│   ├── ref vs reactive → vue-composition (primitives → ref)
│   ├── two-way prop → defineModel() → vue-composition (3.4+)
│   └── reactivity lost after destructure → vue-composition (toRefs/storeToRefs)
└── Shared / cross-component state → vue-state/SKILL.md
    ├── setup store (recommended) → vue-state
    └── SSR hydration (Nuxt) → vue-state
```

## Non-Negotiables (apply to every Vue task)

- **Composition API + `<script setup>`** for new code; Options API only when maintaining legacy.
- **Don't break reactivity.** Destructuring a `reactive` object or a store loses reactivity — use `toRefs`/`storeToRefs`. Primitives belong in `ref`.
- **Template a11y lint clean** (`eslint-plugin-vuejs-accessibility`); semantic elements over `<div @click>`.
- **Typed props/emits** via `defineProps<T>()`/`defineEmits<T>()` generics.

## File Overview

| File | Purpose |
|------|---------|
| [vue-composition/SKILL.md](../vue-composition/SKILL.md) | `<script setup>`, reactivity, composables, macros with version gates |
| [vue-state/SKILL.md](../vue-state/SKILL.md) | Pinia stores: setup vs options, getters, actions, SSR safety |

## Related Skills

- [typescript-skills](../../typescript/typescript-skills/SKILL.md) — typed `defineProps`/`defineEmits`, composable signatures
- [modern-css](../../styling/modern-css/SKILL.md) — scoped styles and modern CSS in SFCs
- [fe-testing](../../quality/fe-testing/SKILL.md) — Vitest + Vue Test Utils / Testing Library
- [accessibility-patterns](../../quality/accessibility-patterns/SKILL.md) — ARIA and keyboard support in templates
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical Vue / Nuxt version minimums
