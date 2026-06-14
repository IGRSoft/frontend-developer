---
name: version-feature-matrix
description: Canonical lookup mapping front-end framework and language versions (React 19, Next.js 15 App Router, Vue 3.5, Nuxt 3, Svelte 5 runes, SvelteKit, Angular 18+, TypeScript 5.x, ES2024, modern CSS, Vite 5/6) to the feature that needs them and the pre-version fallback. Reference before asserting any feature is available or pinning a version.
---

# Version & Feature Matrix (Front-End)

**Canonical version source for this plugin.** Every version-specific claim in any
skill, agent, or command links here rather than restating minimums — one table to
update. Each row is `feature → min version → fallback`. Browser support and
framework feature landings shift between minor releases; for entries marked
*(verify)*, confirm against the project's actual `package.json`, the framework's
release notes, and [caniuse.com](https://caniuse.com) / [Baseline](https://web.dev/baseline)
before relying on a feature in production.

The doctrine every version-gated claim carries, verbatim:

> Requires `<feature>` (`<framework>` `<version>`+). Fallback: `<pre-version approach>`. Canonical: _shared/version-feature-matrix.md

## React

| Feature | Min version | Fallback (pre-version) |
|---------|-------------|------------------------|
| React Server Components (RSC) | React 19 + a framework (Next.js 15 App Router / similar) | Client components + route loaders; data-fetch in `useEffect` or a loader |
| Server Actions (`"use server"`, `<form action={fn}>`) | React 19 (+ Next.js 15) | API route + `fetch` from the client |
| `use(promise)` / `use(context)` hook | React 19 | `useContext` + a suspense-less data lib (React Query); thread promises manually |
| `useActionState` / `useFormStatus` | React 19 | `useState` + manual pending/error tracking |
| `useOptimistic` | React 19 | local optimistic state in `useState`, reconcile on response |
| `ref` as a prop (no `forwardRef`) | React 19 | `forwardRef(...)` |
| Document metadata (`<title>`/`<meta>` in components) | React 19 | `next/head`, `react-helmet`, framework head API |
| React Compiler (auto-memoization) | React 19-era, opt-in | manual `useMemo`/`useCallback`/`React.memo` |
| Concurrent features (`useTransition`, `useDeferredValue`, `<Suspense>`) | React 18+ | synchronous renders; manual debouncing |

> Requires React Server Components and Server Actions (React 19 + Next.js 15 App Router). Fallback: client components with route loaders / API routes. Canonical: _shared/version-feature-matrix.md

## Next.js

| Feature | Min version | Fallback (pre-version) |
|---------|-------------|------------------------|
| App Router (`app/`, layouts, RSC by default) | Next.js 13.4 stable; 15 is the current baseline | Pages Router (`pages/`, `getServerSideProps`/`getStaticProps`) |
| Async request APIs (`cookies()`/`headers()`/`params` are Promises) | Next.js 15 *(verify)* | synchronous access in Next 14 |
| `after()` (post-response work) | Next.js 15 *(verify)* | run the work inline before responding |
| Partial Prerendering (PPR) | Next.js 15 experimental *(verify)* | full SSR or full static, no per-route mix |
| Turbopack dev (`next dev --turbopack`) | Next.js 15 stable for dev *(verify)* | webpack dev server |
| Route Handlers (`app/**/route.ts`) | App Router | `pages/api/*` API routes |

> Requires Next.js 15 App Router with async request APIs. Fallback: Pages Router with `getServerSideProps` on Next 14. Canonical: _shared/version-feature-matrix.md

## Vue

| Feature | Min version | Fallback (pre-version) |
|---------|-------------|------------------------|
| `<script setup>` + Composition API | Vue 3.0 | Options API; `setup()` function |
| Reactive props destructure (compile-time) | Vue 3.5 *(verify)* | `toRefs(props)` / access `props.x` directly |
| `useId()`, `useTemplateRef()` | Vue 3.5 *(verify)* | manual `ref` + generated ids |
| `defineModel()` (two-way binding macro) | Vue 3.4+ | `modelValue` prop + `update:modelValue` emit |
| Generic components (`<script setup generic="T">`) | Vue 3.3+ | non-generic component; cast at call site |
| Suspense (`<Suspense>`) | Vue 3.x experimental *(verify)* | manual loading state |

> Requires Vue 3.5 reactive props destructure / `useId`. Fallback: `toRefs(props)` and manual ids on Vue 3.4. Canonical: _shared/version-feature-matrix.md

## Nuxt

| Feature | Min version | Fallback (pre-version) |
|---------|-------------|------------------------|
| Nuxt 3 (Vue 3, Nitro server, `app/` + `server/`) | Nuxt 3.x | Nuxt 2 (Vue 2, `@nuxtjs/composition-api`) |
| Server routes (`server/api/*`), `runtimeConfig` split | Nuxt 3.x | `serverMiddleware` (Nuxt 2) |
| Hybrid rendering / route rules (`routeRules`) | Nuxt 3.x | per-page `ssr`/`generate` config |
| `useFetch`/`useAsyncData` with `$fetch` | Nuxt 3.x | `asyncData`/`fetch` hooks (Nuxt 2) |

> Requires Nuxt 3 route rules / Nitro server routes. Fallback: Nuxt 2 `serverMiddleware` and per-page config. Canonical: _shared/version-feature-matrix.md

## Svelte / SvelteKit

| Feature | Min version | Fallback (pre-version) |
|---------|-------------|------------------------|
| Runes (`$state`, `$derived`, `$effect`, `$props`, `$bindable`) | Svelte 5 | Svelte 4 reactive `let` + `$:` reactive statements + `export let` props |
| Snippets (`{#snippet}` / `{@render}`) | Svelte 5 | slots (`<slot>`) |
| Event attributes (`onclick={…}` instead of `on:click`) | Svelte 5 | `on:click` directive (Svelte 4) |
| `$effect.pre` / fine-grained reactivity | Svelte 5 | `beforeUpdate`/`afterUpdate` lifecycle (Svelte 4) |
| SvelteKit form actions, `load` functions | SvelteKit 1.x/2.x | manual endpoints + client fetch |
| SvelteKit `$env/static/private` etc. | SvelteKit 1.x+ | `process.env` in server hooks |

> Requires Svelte 5 runes (`$state`/`$derived`/`$effect`). Fallback: Svelte 4 `$:` reactive statements and `export let` props. Canonical: _shared/version-feature-matrix.md

## Angular

| Feature | Min version | Fallback (pre-version) |
|---------|-------------|------------------------|
| Signals (`signal()`, `computed()`, `effect()`) | Angular 16 (stable 17+); `input()`/`output()`/`model()` signals 17.1–18 *(verify)* | RxJS `BehaviorSubject` + `async` pipe; `@Input()`/`@Output()` decorators |
| Standalone components/directives/pipes (no NgModule) | Angular 15+ (default in new apps 17+) | declare in an `NgModule` |
| New control flow (`@if`/`@for`/`@switch`) | Angular 17 | `*ngIf`/`*ngFor`/`*ngSwitch` structural directives |
| Deferred loading (`@defer`) | Angular 17 | manual lazy-load / `loadChildren` |
| `provideHttpClient()` functional providers | Angular 15+ | `HttpClientModule` import |
| Zoneless change detection | Angular 18 experimental *(verify)* | Zone.js change detection (default) |

> Requires Angular 18+ signals and standalone components. Fallback: Angular 15 RxJS `BehaviorSubject` + `async` pipe and NgModule declarations. Canonical: _shared/version-feature-matrix.md

## TypeScript

| Feature | Min version | Fallback (pre-version) |
|---------|-------------|------------------------|
| `const` type parameters (`<const T>`) | TS 5.0 | manual `as const` at call sites |
| `using`/`await using` (explicit resource management) | TS 5.2 | manual `try/finally` cleanup |
| `satisfies` operator | TS 4.9 | explicit type annotation + widening care |
| Decorators (ES standard) | TS 5.0 | experimental decorators (`experimentalDecorators`) |
| `NoInfer<T>` utility | TS 5.4 | hand-rolled inference-blocking wrapper |
| `${configDir}` in tsconfig, `module: "preserve"` | TS 5.5 *(verify)* | relative paths; `module: "esnext"` |
| Isolated declarations (`--isolatedDeclarations`) | TS 5.5 *(verify)* | full type-checker `.d.ts` emit |

> Requires TypeScript 5.2 `using` / 5.0 `const` type params. Fallback: `try/finally` cleanup and `as const` on TS 4.9. Canonical: _shared/version-feature-matrix.md

## JavaScript (ECMAScript)

| Feature | Min standard | Fallback (pre-version) |
|---------|--------------|------------------------|
| `Array.prototype.group` / `Object.groupBy` / `Map.groupBy` | ES2024 *(verify engine support)* | reduce into a `Map`/record manually |
| `Promise.withResolvers()` | ES2024 *(verify)* | construct a promise + capture `resolve`/`reject` in the executor |
| `Array.fromAsync()` | ES2024 *(verify)* | `for await` accumulation |
| Well-formed `Unicode` (`isWellFormed`/`toWellFormed`) | ES2024 *(verify)* | manual surrogate validation |
| Array `findLast`/`findLastIndex`, `toSorted`/`toReversed`/`with` (non-mutating) | ES2023 | reverse-then-find; copy-then-sort |
| Top-level `await` | ES2022 (ESM) | async IIFE wrapper |
| `Array.prototype.at`, `Object.hasOwn`, error `cause` | ES2022 | index math; `hasOwnProperty`; manual error chaining |

> Requires `Object.groupBy` / `Promise.withResolvers` (ES2024; verify runtime support). Fallback: manual `Map` grouping and an explicit promise executor. Canonical: _shared/version-feature-matrix.md

## Modern CSS

| Feature | Baseline anchor | Fallback (pre-support) |
|---------|-----------------|------------------------|
| Container queries (`@container`, `cqw`/`cqi` units) | Baseline 2023 (verify older targets) | media queries against the viewport; JS `ResizeObserver` |
| `:has()` relational selector | Baseline 2023 *(verify Firefox/older)* | a wrapper class toggled in JS |
| Cascade layers (`@layer`) | Baseline 2022 | careful source ordering + selector specificity discipline |
| Subgrid (`grid-template-*: subgrid`) | Baseline 2023 *(verify)* | nested explicit grids with shared track sizes |
| `:is()` / `:where()` | Baseline 2021 | enumerate selectors; `:where()` ≈ low-specificity grouping |
| Nesting (native CSS `&`) | Baseline 2023 *(verify)* | a preprocessor (Sass/PostCSS nesting plugin) |
| `color-mix()`, relative color syntax | Baseline 2023 / newer *(verify)* | precomputed color tokens / Sass color functions |
| `text-wrap: balance`/`pretty` | newer *(verify)* | accept default wrapping (progressive enhancement) |
| `@property` (typed custom properties) | Baseline 2024 *(verify)* | untyped `--var` + JS-driven transitions |

> Requires container queries and `:has()` (Baseline 2023; verify older-browser targets). Fallback: viewport media queries and a JS-toggled wrapper class. Canonical: _shared/version-feature-matrix.md

## Build Tooling

| Tool / feature | Min version | Fallback (pre-version) |
|----------------|-------------|------------------------|
| Vite 5 (Rollup 4, Node 18+) | Vite 5.x | Vite 4 (Rollup 3) |
| Vite 6 (Environment API, Node 18+) | Vite 6.x *(verify)* | Vite 5 single-environment dev server |
| Vitest (Vite-native test runner) | tracks the installed Vite major | Jest with `ts-jest`/babel |
| esbuild / SWC transpile | bundled with Vite/Next | Babel |
| `package.json` `exports` conditions / ESM-first | Node 18+ ecosystems | `main`/`module` fields, CJS interop |

> Requires Vite 6 Environment API. Fallback: Vite 5 single-environment dev/build. Canonical: _shared/version-feature-matrix.md

## Usage Rules

1. **Feature-detect before version-test** at runtime where possible — `CSS.supports('selector(:has(*))')`, `'withResolvers' in Promise`, `typeof window.trustedTypes !== 'undefined'` — rather than sniffing a framework version alone.
2. **Every skill claim that names a version links here** — do not restate minimum versions elsewhere; one table to update.
3. **Pin tool versions in the lockfile, not prose** — the rows above are floors for *features*; the project's `package.json`/lockfile is the source of truth for what is actually installed.
4. **Hedge volatile minutiae** — where this table says *(verify)*, the feature landed across several releases or has uneven browser support; confirm against the project and caniuse/Baseline before relying on it.

## Related Skills

- `language-detection.md` — routing to the right framework agent before version questions arise
- `model-selection.md` — model/effort to pass with the routed `Task()` call
- `accessibility-baseline.md` — WCAG floors (orthogonal to framework versions)
- `secure-coding/SKILL.md` — browser-security-feature floors (Trusted Types, CSP level)
