---
name: vue-composition
description: >-
  Vue 3 Composition API with `<script setup>`: refs vs reactive, computed and
  watch, building reusable composables, typed `defineProps`/`defineEmits`, the
  `defineModel` two-way macro (3.4+), reactive props destructure and `useId`
  (3.5+), and the reactivity rules that prevent lost-reactivity bugs. Use when
  writing Vue single-file components, extracting a composable, choosing refs vs
  reactive, or debugging broken reactivity.
---

# Vue Composition API (`<script setup>`)

**Composition API patterns and the reactivity rules behind them.** The canonical
selection table lives in [vue-skills/SKILL.md](../vue-skills/SKILL.md); this leaf
carries depth only.

## When to Use

- Writing a Vue 3 single-file component with `<script setup>`
- Choosing `ref` vs `reactive` for a piece of state
- Extracting shared logic into a composable (`useX`)
- Wiring a two-way bound prop with `defineModel`
- Fixing a "value updates but template doesn't" reactivity bug

## `ref` vs `reactive`

```ts
import { ref, reactive, computed } from "vue";

const count = ref(0);                       // primitives → ref; access via .value in script
const user = reactive({ name: "", age: 0 }); // objects → reactive; no .value
const doubled = computed(() => count.value * 2);
```

**Rule:** default to `ref` (works for any value, survives destructuring via
`toRef`); use `reactive` for an object you treat as a cohesive unit. **Never
destructure a `reactive` object** — it severs reactivity. In templates, refs
auto-unwrap (no `.value`).

## The lost-reactivity trap

```ts
const state = reactive({ q: "" });
let { q } = state;          // ❌ q is now a plain string snapshot — not reactive
const { q: qRef } = toRefs(state);  // ✅ keeps reactivity
```

Same with stores — use `storeToRefs(store)`, not plain destructuring (see
[vue-state](../vue-state/SKILL.md)).

## Typed props and emits

```vue
<script setup lang="ts">
const props = defineProps<{ id: string; count?: number }>();
const emit = defineEmits<{ change: [value: number]; close: [] }>();
</script>
```

> Requires Vue 3.5 reactive props destructure (`const { count } = defineProps(...)` stays reactive). Fallback: access `props.count` directly or `toRefs(props)` on Vue 3.4. Canonical: _shared/version-feature-matrix.md

## `defineModel`: two-way binding (3.4+)

```vue
<script setup lang="ts">
const model = defineModel<string>();   // replaces modelValue prop + update:modelValue emit
</script>
<template><input :value="model" @input="model = $event.target.value" /></template>
```

> Requires `defineModel()` (Vue 3.4+). Fallback: declare a `modelValue` prop and emit `update:modelValue` on Vue 3.3. Canonical: _shared/version-feature-matrix.md

## Composables: extract reusable reactive logic

A composable is a function starting with `use` that returns reactive state and
methods. It is the Vue analog of a custom hook.

```ts
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from "vue";
export function useMouse() {
  const x = ref(0), y = ref(0);
  const update = (e: MouseEvent) => { x.value = e.clientX; y.value = e.clientY; };
  onMounted(() => window.addEventListener("mousemove", update));
  onUnmounted(() => window.removeEventListener("mousemove", update));  // always clean up
  return { x, y };
}
```

Rules: return `ref`s (not a `reactive` object, so callers can destructure);
register lifecycle hooks synchronously at the top of the composable; clean up
listeners/timers in `onUnmounted`.

## `watch` / `watchEffect`

```ts
watch(() => props.id, (id) => load(id), { immediate: true }); // explicit source
watchEffect(() => console.log(count.value));                   // auto-tracks reads
```

Use `watch` when you need the old value or an explicit dependency; `watchEffect`
when you want to react to whatever it reads. Watch a getter (`() => x.value`),
not a bare ref, when watching a derived value.

## `useTemplateRef` / `useId` (3.5+)

```vue
<script setup lang="ts">
import { useTemplateRef, useId } from "vue";
const inputEl = useTemplateRef<HTMLInputElement>("field");  // 3.5+
const id = useId();                                          // SSR-stable unique id
</script>
<template><label :for="id">Name</label><input :id ref="field" /></template>
```

> Requires `useTemplateRef`/`useId` (Vue 3.5+). Fallback: a matching `ref` name + a manually generated id on Vue 3.4. Canonical: _shared/version-feature-matrix.md

## Diagnostics

| Symptom | Cause | Fix |
|---------|-------|-----|
| Value changes but template doesn't update | destructured a `reactive`/store | use `toRefs`/`storeToRefs` |
| `ref` is `undefined` in `onMounted` | template ref name mismatch | match the `ref="x"` name; or `useTemplateRef("x")` |
| `watch` never fires | watching a bare value, not a getter | watch `() => source` |
| Memory grows / duplicate listeners | composable didn't clean up | add `onUnmounted` teardown |
| Props seem stale after destructure | pre-3.5 destructure of `defineProps` | access `props.x` or upgrade to 3.5 |

## Related Skills

- [vue-state](../vue-state/SKILL.md) — Pinia and `storeToRefs` for shared state
- [typescript-skills](../../typescript/typescript-skills/SKILL.md) — typed props/emits/composables
- [vue-skills](../vue-skills/SKILL.md) — canonical selection table (start here)
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical Vue version minimums
