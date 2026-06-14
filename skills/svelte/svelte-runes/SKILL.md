---
name: svelte-runes
description: >-
  Svelte 5 runes reactivity: `$state` and deep reactivity, `$derived`/`$derived.by`,
  `$effect` (and why to avoid it), `$props` with defaults and rest, `$bindable`
  two-way props, snippets (`{#snippet}`/`{@render}`), `onclick` event attributes,
  and the Svelte 4 → 5 migration map. Use when writing Svelte 5 components,
  migrating from `$:`/`export let`/`<slot>`, or debugging an effect loop or lost
  reactivity.
---

# Svelte 5 Runes

**Runes make reactivity explicit and signals-based.** The canonical selection
table lives in [svelte-skills/SKILL.md](../svelte-skills/SKILL.md); this leaf
carries depth only.

## When to Use

- Writing reactive state, derived values, or effects in Svelte 5
- Declaring component props and two-way bindings
- Replacing `<slot>` content with snippets
- Migrating a Svelte 4 component to runes
- Debugging an infinite `$effect` or a value that won't update

## The five core runes

```svelte
<script lang="ts">
  let count = $state(0);                       // reactive state (deeply reactive for objects/arrays)
  let doubled = $derived(count * 2);           // recomputed when deps change
  let { label = "Add", onclick }: Props = $props(); // props with defaults
  $effect(() => { document.title = `Count: ${count}`; }); // sync with the outside world
</script>

<button {onclick}>{label}: {count}</button>
```

> Requires Svelte 5 runes (`$state`/`$derived`/`$effect`/`$props`). Fallback: Svelte 4 `let` + `$:` reactive statements + `export let` on Svelte 4. Canonical: _shared/version-feature-matrix.md

## `$derived` over `$effect`

Prefer `$derived` (or `$derived.by(() => {...})` for multi-statement logic) for
any **computed value**. Reach for `$effect` only to synchronize with something
*outside* Svelte's reactivity (DOM measurement, third-party libs, subscriptions).

```svelte
let full = $derived(`${first} ${last}`);          // ✅ derived value
$effect(() => { const id = setInterval(tick, 1000); return () => clearInterval(id); }); // ✅ external + cleanup
```

**Effect rules:** never set state the effect reads (infinite loop); return a
teardown function for timers/listeners/subscriptions; keep effects small.

## Props with `$props` and two-way `$bindable`

```svelte
<script lang="ts">
  interface Props { value: string; required?: boolean; }
  let { value = $bindable(""), required = false, ...rest }: Props = $props();
</script>
<input bind:value {required} {...rest} />
```

`$bindable` opts a prop into `bind:`; `...rest` spreads remaining attributes.
Type props with an `interface` for IDE + `svelte-check` safety.

## Snippets replace slots

```svelte
{#snippet row(item)}
  <li>{item.name}</li>
{/snippet}

<ul>{#each items as item}{@render row(item)}{/each}</ul>
```

Snippets are reusable, parameterized markup — pass them as props for what slots
used to do. `{@render children?.()}` renders default content.

> Requires Svelte 5 snippets (`{#snippet}`/`{@render}`). Fallback: `<slot>` / named slots on Svelte 4. Canonical: _shared/version-feature-matrix.md

## Event attributes, not directives

Svelte 5 uses plain attributes: `onclick={handler}` instead of
`on:click={handler}`. Event modifiers (`|preventDefault`) are gone — call
`e.preventDefault()` in the handler, or use a small wrapper.

> Requires Svelte 5 event attributes (`onclick={…}`). Fallback: `on:click` directive + `|preventDefault` modifiers on Svelte 4. Canonical: _shared/version-feature-matrix.md

## Svelte 4 → 5 migration map

| Svelte 4 | Svelte 5 |
|----------|----------|
| `let count = 0` (top-level, reactive) | `let count = $state(0)` |
| `$: doubled = count * 2` | `let doubled = $derived(count * 2)` |
| `$: { sideEffect() }` | `$effect(() => { sideEffect(); })` |
| `export let value` | `let { value } = $props()` |
| `export let value` + `bind:value` from parent | `let { value = $bindable() } = $props()` |
| `<slot>` / `<slot name="x">` | snippets + `{@render children()}` |
| `on:click={fn}` / `on:click\|preventDefault` | `onclick={fn}` / handle in `fn` |
| `createEventDispatcher()` | callback props (`onsomething`) |

Run the official migration script (`npx sv migrate svelte-5`) for a first pass,
then review effects by hand — `$:` blocks that did side effects become `$effect`,
those that computed become `$derived`.

## State for objects/arrays

`$state({...})` / `$state([...])` is **deeply reactive** — mutating
`obj.field` or `arr.push(x)` triggers updates. Use `$state.raw(x)` to opt out of
deep proxying for large immutable data, and `$state.snapshot(x)` to get a plain
(non-proxy) copy for serialization or passing to external libs.

## Diagnostics

| Symptom | Cause | Fix |
|---------|-------|-----|
| "effect_update_depth_exceeded" | `$effect` sets state it also reads | move the computation to `$derived`; don't self-trigger |
| Value won't update in template | plain `let` instead of `$state`, or pre-runes file | wrap in `$state`; ensure runes mode |
| Mixing `$:` and runes errors | can't mix legacy + runes in one component | convert the whole component to runes |
| `bind:` to a prop fails | prop not `$bindable` | wrap default in `$bindable(...)` |
| Passing `$state` proxy to a library breaks it | library got a Proxy | pass `$state.snapshot(value)` |

## Related Skills

- [sveltekit](../sveltekit/SKILL.md) — app routing, `load` data flowing into runes
- [typescript-skills](../../typescript/typescript-skills/SKILL.md) — typed `$props` interfaces
- [svelte-skills](../svelte-skills/SKILL.md) — canonical selection table (start here)
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical Svelte version minimums
