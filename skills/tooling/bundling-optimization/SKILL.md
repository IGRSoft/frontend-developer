---
name: bundling-optimization
description: >-
  Shrinking front-end bundles — route/component code-splitting with dynamic
  import(), tree-shaking (ESM + sideEffects), lazy-loading, and measuring with a
  bundle analyzer. Use when a bundle is too large, JS is blocking interactivity,
  dead code ships to users, or you need to tie bundle size to a performance
  budget.
---

# Bundling Optimization

**Code-splitting, tree-shaking, and lazy-loading to a budget.** For the domain
selection table, see the canonical [tooling-skills/SKILL.md](../tooling-skills/SKILL.md) —
this leaf carries depth and does not duplicate it. Budget targets are canonical in
[web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md).

## When to Use

Use this skill when:
- The initial JS bundle is too large / Time-to-Interactive is poor
- Dead or unused code is shipping to users
- A heavy feature (chart lib, editor, map) loads on first paint but is rarely used
- You need to measure where bytes go before changing config

## Measure first (always)

Never optimize blind. Generate a treemap and read it before touching config:

```
npx vite-bundle-visualizer        # Vite/Rollup
npx webpack-bundle-analyzer        # Webpack stats
```

Record the before/after byte delta in DV Build Evidence. Optimize the biggest,
least-used chunks first — a 200KB date library imported wholesale beats shaving
kilobytes off your own code.

## Code-splitting with dynamic import()

`import()` returns a promise and creates a separate chunk loaded on demand.

```ts
// Route-level (the highest-leverage split)
const Dashboard = lazy(() => import('./routes/Dashboard'));   // React
// Vue: () => import('./Dashboard.vue') in the route record
// Svelte/SvelteKit: route-based splitting is automatic
// Angular: @defer / loadComponent (see angular-signals)
```

| Split at… | When |
|-----------|------|
| **Route boundaries** | Almost always — users rarely visit every route on load |
| **Heavy feature components** | Editors, charts, maps, rich-text — load on interaction/viewport |
| **Below-the-fold widgets** | Defer until scrolled into view (`IntersectionObserver` / `@defer`) |
| **Conditional/admin code** | Split behind a capability/role check |

Pair with a loading placeholder (`<Suspense fallback>`, skeleton) so the split
does not cause layout shift (CLS).

## Tree-shaking: ship only what's used

Tree-shaking removes unused exports — but only works when the toolchain can prove
code is side-effect-free.

```jsonc
// package.json — tell the bundler your modules are pure
{ "sideEffects": false }                    // whole package is side-effect-free
{ "sideEffects": ["*.css", "./src/polyfills.ts"] }  // these have side effects; keep them
```

Rules that make tree-shaking actually work:
- **Use ESM imports/exports**, not CommonJS `require()` — CJS is not statically
  analyzable, so it ships whole.
- **Import named members**, not whole namespaces: `import { debounce } from 'lodash-es'`
  (or `lodash.debounce`), never `import _ from 'lodash'`.
- **Avoid import-for-side-effect** unless intended, and list those files in
  `sideEffects` so they are not shaken away.
- **Watch barrel files** (`index.ts` re-exporting everything) — they can defeat
  shaking if the bundler can't prune; import from the deep path when in doubt.

> Tree-shaking depends on ESM + correct `sideEffects` metadata. Fallback for CJS-only deps: import a sub-path build, swap to an ESM-published alternative, or accept the full cost. Canonical: _shared/version-feature-matrix.md

## Lazy-loading assets

- **Images:** `loading="lazy"` for below-the-fold; set explicit `width`/`height`
  (or `aspect-ratio`) to prevent CLS; serve responsive `srcset`; prefer AVIF/WebP.
- **Fonts:** subset, `font-display: swap`, preload only the critical face.
- **Third-party scripts:** load `async`/`defer`; gate non-essential ones behind
  interaction. Each third-party script is bundle weight you don't control.

## Chunking strategy

```ts
// Vite/Rollup: split stable vendor code so app changes don't bust its cache
build: { rollupOptions: { output: { manualChunks: { vendor: ['react', 'react-dom'] } } } }
```

Split by **stable cache boundaries** (framework vendor vs app code), not arbitrarily
— over-splitting creates request waterfalls and defeats compression. Measure the
network waterfall, not just total bytes.

## Tie it to a budget

Bundle work has a target, not "smaller forever." The budget lives in
[web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md): LCP < 2.5s,
INP < 200ms, CLS < 0.1. JS that blocks the main thread hurts INP and TTI; over-eager
splitting that adds round-trips hurts LCP. Optimize toward the budget, then stop.

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| Optimizing without an analyzer | measure the treemap first |
| `import _ from 'lodash'` | `import { debounce } from 'lodash-es'` |
| CommonJS deps blocking shaking | ESM build / sub-path import / swap dep |
| Loading a chart/editor on first paint | dynamic `import()` on interaction/viewport |
| Images with no dimensions (CLS) | explicit `width`/`height` or `aspect-ratio` |
| Over-splitting into many tiny chunks | split by stable vendor boundaries; measure waterfall |

## Related Skills

- [web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md) — Core Web Vitals budgets this work targets
- [build-systems/SKILL.md](../build-systems/SKILL.md) — where chunking/sourcemap config lives
- [tooling-skills/SKILL.md](../tooling-skills/SKILL.md) — canonical domain selection table
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — ESM/`exports` and tooling minimums
