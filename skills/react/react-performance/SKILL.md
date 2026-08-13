---
name: react-performance
description: >-
  React render-performance diagnosis and fixes: finding wasted re-renders,
  applying `memo`/`useMemo`/`useCallback` (and when the React Compiler makes
  them unnecessary), streaming with Suspense, code-splitting with
  `lazy`/`Suspense`, and virtualizing long lists. Use when a component
  re-renders too often, a list janks while scrolling, the UI blocks on a heavy
  update, or you need to measure render cost.
---

# React Performance

**Measure first, then memoize the hot path — not everything.** The canonical
selection table lives in [react-skills/SKILL.md](../react-skills/SKILL.md); this
leaf carries depth only. For Core Web Vitals (LCP/INP/CLS) beyond render cost,
see [web-performance](../../quality/web-performance/SKILL.md).

## When to Use

- A component re-renders far more than its data changes
- A long list stutters while scrolling
- A heavy update blocks typing/clicking
- Initial JS bundle is too large (route/feature is rarely used)
- You need to confirm a "perf fix" actually helped

## Diagnose before you optimize

Use the React DevTools **Profiler** (flame graph + "why did this render") and the
Components panel's "Highlight updates". Don't memoize blind — confirm the
component is actually re-rendering and that the render is expensive.

| Symptom | Likely cause | Where to look |
|---------|--------------|---------------|
| Child re-renders when unrelated parent state changes | new object/array/function prop each render | Profiler → props that changed |
| Whole list re-renders on one item change | non-stable keys, or parent holds the list state | keys + state location |
| Typing/clicking feels laggy | a heavy synchronous update on every keystroke | `useTransition` / `useDeferredValue` |
| Big initial bundle | eagerly imported rarely-used route/feature | `React.lazy` + route-level split |

## Memoization: targeted, not reflexive

```tsx
// Stable callback identity so a memoized child doesn't re-render.
const onSelect = useCallback((id: string) => setSelected(id), []);

// Memoize an expensive derivation, not cheap ones.
const sorted = useMemo(() => expensiveSort(items), [items]);

// Memoize a component whose props are usually unchanged.
const Row = memo(function Row({ item }: { item: Item }) { /* ... */ });
```

**Rule:** memoize when a measurement shows wasted work. `useMemo`/`useCallback`
have their own cost; over-memoizing cheap components is net-negative.

> Requires the React Compiler for automatic memoization (1.0 stable, opt-in). With it enabled, remove most manual `useMemo`/`useCallback`/`memo`. Fallback: manual memoization. Canonical: _shared/version-feature-matrix.md

## Keep render-time work cheap

- Derive state during render or with `useMemo` — never sync derived state with `useEffect` + `setState` (extra render + bug surface).
- Pass stable props: hoist constant objects/arrays out of the component, or memoize them.
- Split state so a frequently-changing value doesn't sit in a component that renders an expensive subtree.
- Subscribe to narrow store slices (see [react-state](../react-state/SKILL.md)).

## Concurrent features for responsiveness (18+)

```tsx
const [isPending, startTransition] = useTransition();
function onChange(next: string) {
  setInput(next);                                  // urgent: keep the field responsive
  startTransition(() => setResults(filter(next))); // non-urgent: may be interrupted
}
```

`useDeferredValue(value)` is the prop-side equivalent for a value you receive.
Pre-18 fallback: manual debouncing.

## Suspense + streaming + code-splitting

```tsx
const Settings = lazy(() => import("./Settings"));   // route/feature-level split

<Suspense fallback={<Spinner />}>
  <Settings />        {/* JS for Settings loads on demand; UI streams in */}
</Suspense>
```

Split at route and large-feature boundaries; in the App Router, `loading.tsx`
gives Suspense streaming for free. Don't over-split tiny components — each chunk
has request overhead. See [bundling-optimization](../../tooling/bundling-optimization/SKILL.md).

## Virtualize long lists

Rendering thousands of DOM nodes is the usual scroll-jank cause. Render only the
visible window with `@tanstack/react-virtual` (or `react-window`).

```tsx
const rowVirtualizer = useVirtualizer({
  count: rows.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 40,
});
// render only rowVirtualizer.getVirtualItems()
```

Keep row components `memo`-stable and keys derived from item id, not index.
Preserve keyboard navigation and ARIA roles (`role="listbox"`/`option`) — see
[accessibility-patterns](../../quality/accessibility-patterns/SKILL.md).

## Verify the win

Re-profile after the change; compare commit counts / flame-graph durations. For
load-time work, check the bundle delta and Lighthouse — record both in the DV
Build Evidence (per [CORPFLOW.md](../../../CORPFLOW.md)).

## Related Skills

- [react-state](../react-state/SKILL.md) — state location and selector granularity drive re-render cost
- [modern-react](../modern-react/SKILL.md) — RSC moves work off the client; Compiler removes manual memo
- [web-performance](../../quality/web-performance/SKILL.md) — Core Web Vitals (LCP/INP/CLS) and Lighthouse budgets
- [bundling-optimization](../../tooling/bundling-optimization/SKILL.md) — code-splitting and bundle analysis
- [react-skills](../react-skills/SKILL.md) — canonical selection table (start here)
