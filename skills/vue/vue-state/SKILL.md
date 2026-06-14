---
name: vue-state
description: >-
  Pinia state management for Vue 3: setup stores vs options stores, state /
  getters / actions, consuming a store with `storeToRefs` without losing
  reactivity, composing stores, and SSR-safe state for Nuxt. Use when building
  or structuring a Pinia store, deciding what belongs in a store vs a
  composable, or fixing reactivity loss when reading store state.
---

# Vue State (Pinia)

**Pinia is the official store for Vue 3 — typed, devtools-friendly, modular.**
The canonical selection table lives in
[vue-skills/SKILL.md](../vue-skills/SKILL.md); this leaf carries depth only.

## When to Use

- Building a shared store for state used across components/routes
- Deciding between a setup store and an options store
- Reading store state without breaking reactivity
- Composing one store from another
- Making global state SSR-safe under Nuxt 3

## Store vs composable: what belongs where

| Put it in... | When |
|--------------|------|
| a **Pinia store** | state shared across distant components/routes, devtools-tracked, possibly persisted/SSR-hydrated |
| a **composable** | reusable reactive logic scoped to where it's used (a `useMouse`, a form helper) — no global singleton needed |
| **component `ref`s** | state local to one component |

Don't make a store for state one component owns; don't make a composable a hidden
global singleton.

## Setup store (recommended)

A setup store mirrors `<script setup>`: `ref` = state, `computed` = getter,
function = action.

```ts
import { defineStore } from "pinia";
import { ref, computed } from "vue";

export const useCart = defineStore("cart", () => {
  const items = ref<Item[]>([]);                       // state
  const total = computed(() => items.value.reduce((s, i) => s + i.price, 0)); // getter
  function add(item: Item) { items.value.push(item); } // action (sync or async)
  return { items, total, add };
});
```

Setup stores compose naturally (call another `useX()` store inside) and type
cleanly. Options stores (`{ state, getters, actions }`) are equivalent and fine
for teams preferring that shape.

## Consume with `storeToRefs` — don't destructure raw

```ts
const cart = useCart();
const { items, total } = storeToRefs(cart);  // ✅ state/getters stay reactive
const { add } = cart;                          // actions: destructure directly (functions, not reactive)
```

Plainly destructuring `const { items } = cart` **loses reactivity** — the single
most common Pinia bug. State/getters → `storeToRefs`; actions → plain
destructure.

## Composing stores

```ts
export const useCheckout = defineStore("checkout", () => {
  const cart = useCart();                       // use another store inside
  const user = useUser();
  async function submit() { await api.order(cart.items, user.id); cart.items.length = 0; }
  return { submit };
});
```

Call the dependency stores inside the setup function, not at module top level.

## SSR-safe state (Nuxt 3)

Under SSR each request must get a **fresh** store instance — a module-level
singleton would leak state between users. Nuxt's Pinia module handles this:
create/hydrate the Pinia instance per request, and never read `window`/browser
globals during store setup (guard with `import.meta.client` or do it in an
action called from a client lifecycle hook).

> Requires Nuxt 3 for per-request Pinia hydration. Fallback: manual `createPinia()` per request in a Nuxt 2 `@nuxtjs/composition-api` setup. Canonical: _shared/version-feature-matrix.md

## Persistence

Use `pinia-plugin-persistedstate` (or a small custom plugin) to sync chosen state
to `localStorage`/`sessionStorage` — never persist secrets or tokens to
`localStorage` (XSS-readable; see [secure-coding](../../_shared/secure-coding/SKILL.md)).

## Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| `const { items } = cart` (raw destructure of state) | `storeToRefs(cart)` |
| Store for state one component owns | local `ref` |
| Module-level singleton state under SSR | per-request Pinia (Nuxt module) |
| Caching server data in a store long-term | refetch/invalidate strategy; treat server data as a cache |
| Tokens/secrets persisted to `localStorage` | in-memory only; httpOnly cookie for the session |

## Related Skills

- [vue-composition](../vue-composition/SKILL.md) — `storeToRefs`, `toRefs`, reactivity rules
- [secure-coding](../../_shared/secure-coding/SKILL.md) — what not to persist client-side
- [vue-skills](../vue-skills/SKILL.md) — canonical selection table (start here)
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical Vue / Nuxt version minimums
