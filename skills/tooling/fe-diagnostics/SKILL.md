---
name: fe-diagnostics
description: >-
  Front-end debugging workflows — reading source maps to map bundled stack
  traces to source, and diagnosing/fixing SSR hydration mismatches. Use when a
  stack trace points at minified/bundled code, when you see "hydration failed" /
  "text content did not match" errors, or when build-time vs runtime behavior
  diverges.
---

# Front-End Diagnostics

**Source maps and hydration-error triage.** For the domain selection table, see
the canonical [tooling-skills/SKILL.md](../tooling-skills/SKILL.md) — this leaf
carries depth and does not duplicate it.

## When to Use

Use this skill when:
- A production stack trace points at `bundle.js:1:90432` and you need the real source line
- You see a **hydration mismatch** warning/error in an SSR/SSG app
- Behavior differs between dev and a production build (env/minification/tree-shaking)
- A reported error has no obvious source location

## Source maps: bundled trace → source line

A source map (`*.js.map`) maps positions in built output back to original source.

```ts
// vite.config.ts / build config
build: { sourcemap: true }      // emit maps; 'hidden' = emit but don't reference in the bundle
```

Workflow:
1. **Reproduce with maps available.** In the browser devtools, an attached map
   resolves the trace automatically; ensure the `.map` is served (or load it in
   devtools → Sources → Add source map).
2. **For server/CI traces**, resolve programmatically with the `source-map`
   package, or `npx source-map-cli` against the line/column.
3. **Match the map to the exact build** — a stale `.map` from a different build
   resolves to the wrong line. Maps are keyed by content hash; regenerate per build.

**Shipping decision:** generating maps aids debugging, but publicly served maps
expose your original source. Choose deliberately: `sourcemap: 'hidden'` emits maps
for upload to an error tracker (Sentry, etc.) without referencing them in the
bundle, so the browser does not download them but your tracker can. Never embed
secrets in source that becomes a published map.

## Hydration errors: server HTML ≠ client render

SSR/SSG sends server-rendered HTML; the framework then **hydrates** it by attaching
event listeners and reconciling against what the client would render. A mismatch
("Hydration failed", "Text content did not match server-rendered HTML",
"server rendered HTML didn't match client") means the two renders disagreed.

### Top causes and fixes

| Cause | Why it mismatches | Fix |
|-------|-------------------|-----|
| `Date.now()` / `new Date()` / `Math.random()` in render | Server and client compute different values | Compute on the client in an effect, or pass a stable value as a prop/seed |
| `typeof window` / `localStorage` branching in render | Server has no `window`; renders the other branch | Render the server-safe branch, then update after mount (`useEffect`/`onMounted`) |
| Invalid HTML nesting (`<div>` inside `<p>`, `<p>` inside `<button>`) | Browser repairs the DOM; tree no longer matches | Fix the markup to be valid |
| Locale/timezone-formatted output | Server and client format differently | Format on the client, or pin locale/timezone |
| Browser-extension or third-party DOM mutation | Alters DOM before hydration | Often benign; suppress narrowly if confirmed external |
| Reading user-specific state during SSR | Server renders the wrong variant | Defer to client; render a neutral placeholder server-side |

### The pattern: defer client-only content

```tsx
// React: render nothing/placeholder on the server, real content after mount
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);
return mounted ? <ClientOnlyWidget /> : <Placeholder />;
```

> Framework escape hatches exist (React `suppressHydrationWarning`, `next/dynamic` `ssr:false`, Vue `<ClientOnly>`, Svelte `{#if browser}`). Use them narrowly and only after identifying the real cause — silencing the warning hides correctness bugs. Canonical: _shared/version-feature-matrix.md

**Rule:** a hydration warning is a correctness signal, not noise. Find why the two
renders diverge before suppressing. Suppression is per-node and last resort.

## Build-time vs runtime divergence

If something works in `dev` but breaks in the production build:
- **Minification / tree-shaking** removed code with side effects → mark
  `sideEffects` correctly (see [bundling-optimization](../bundling-optimization/SKILL.md)).
- **Env inlined at build** differs from what you expect at runtime (see
  [build-systems](../build-systems/SKILL.md) env section).
- **Strict-mode double-invoke** (React dev) masked or surfaced an effect bug.
- Reproduce with `npx vite build` + `npx vite preview` (or the framework's prod
  preview) — never debug prod issues against the dev server alone.

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| Debugging a minified trace by eye | attach/resolve the source map |
| Reusing a stale `.map` across builds | regenerate per build (content-hashed) |
| Suppressing every hydration warning | find the diverging render first |
| Non-deterministic values in SSR render | move to a post-mount effect |
| Only testing against the dev server | also test the production build + preview |

## Related Skills

- [tooling-skills/SKILL.md](../tooling-skills/SKILL.md) — canonical domain selection table
- [build-systems/SKILL.md](../build-systems/SKILL.md) — sourcemap config, env inlining
- [bundling-optimization/SKILL.md](../bundling-optimization/SKILL.md) — `sideEffects`, tree-shaking pitfalls
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — SSR/RSC framework version notes
