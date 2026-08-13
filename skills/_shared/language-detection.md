---
name: language-detection
description: Shared marker-to-framework-to-agent routing table for frontend-developer commands and the router agent, plus the web/native precedence pointer. Reference when deciding which front-end agent owns a file, directory, or repository, or when a task carries both web and native markers.
---

# Framework Detection & Agent Routing

Single source of truth for the marker → framework → agent mapping used by
`frontend-developer` (router), `corpflow:developer`, and every command that scopes
work per framework. Keep command-local detection logic in sync with this file —
do not fork the table.

## Detection Priority Order

Evaluate top-down; the first matching tier wins. Within a tier, apply the
tie-breaking rules below.

| Priority | Signal | Why it ranks here |
|----------|--------|-------------------|
| 1 | Explicit user statement ("this is a Vue app", `--framework` flag) | User intent overrides inference |
| 2 | Framework config files (`next.config.*`, `nuxt.config.*`, `svelte.config.*`, `angular.json`, `vite.config.*`) | Declares the framework authoritatively |
| 3 | `package.json` `dependencies` (the framework package itself: `react`, `vue`, `svelte`, `@angular/core`) | Pins the ecosystem |
| 4 | Component-file extension census (`.tsx`/`.jsx`/`.vue`/`.svelte`) | Reflects actual code volume |
| 5 | `.ts`/`.js` only, no framework marker | Plain TypeScript/JavaScript → typescript-developer |

## Marker → Framework → Agent Table

| Marker(s) | Framework | Agent |
|-----------|-----------|-------|
| `next.config.{js,ts,mjs}`, `app/` dir + `react` dep, `.tsx`/`.jsx` with `react` | React / Next.js | `frontend-developer:react-developer` |
| `react` in `package.json`, `.jsx`/`.tsx` (Vite/CRA/Remix) | React | `frontend-developer:react-developer` |
| `nuxt.config.{js,ts}`, `vue` dep, `.vue` files | Vue / Nuxt | `frontend-developer:vue-developer` |
| `svelte.config.js`, `svelte`/`@sveltejs/kit` dep, `.svelte` files | Svelte / SvelteKit | `frontend-developer:svelte-developer` |
| `angular.json`, `@angular/core` dep, `.component.ts` | Angular | `frontend-developer:angular-developer` |
| `.ts`/`.tsx` (type-level, build config, no framework UI) | TypeScript | `frontend-developer:typescript-developer` |
| `.css`/`.scss`/`.pcss`, `tailwind.config.*`, `postcss.config.*`, styling-only change | CSS / styling | `frontend-developer:css-developer` |
| `.html` template, plain DOM, no framework dep | Vanilla web | `frontend-developer:frontend-developer` (router; handles directly or splits) |
| Mixed framework markers (e.g. `next.config` + `.vue`), monorepo | Multi-framework | `frontend-developer:frontend-developer` (router; per-package routing) |

File-level extension map (for per-file routing inside a mixed repo):

| Extension / marker | Agent |
|--------------------|-------|
| `.tsx`, `.jsx` (with `react`) | `react-developer` |
| `.vue` | `vue-developer` |
| `.svelte` | `svelte-developer` |
| `.component.ts`, `.component.html` (Angular) | `angular-developer` |
| `.ts`, `.mts`, `.cts` (no framework UI) | `typescript-developer` |
| `.css`, `.scss`, `.sass`, `.less`, `.pcss` | `css-developer` |
| `tsconfig.json` | `typescript-developer` (config); routed with the owning framework if app-scoped |

## Tie-Breaking Rules

1. **Config file beats a stray dependency.** `next.config.ts` makes it a Next/React project even if a `vue` package lingers in `devDependencies` for a tool. Route by the framework whose config file is present and whose app code dominates.
2. **`.ts`/`.tsx` inside a framework app routes to that framework, not typescript-developer.** A `.ts` store/composable in a Vue app is Vue work; a `.tsx` component in a Next app is React work. `typescript-developer` owns *framework-agnostic* TS: shared type libraries, build/config TS, codegen, and plain Node/browser TS with no UI framework.
3. **Styling-only changes route to css-developer regardless of framework.** A change confined to `.css`/`.scss`/`tailwind.config`/`<style>` blocks is css-developer's, even inside a React/Vue/Svelte/Angular app. A change that touches both component logic and styles routes to the framework agent, which consults css-developer.
4. **`tsconfig.json` is not a framework marker by itself.** Classify by the framework config and component files; a `tsconfig.json` edit scoped to compiler options is typescript-developer, but an app-wide path/alias change rides with the owning framework agent.
5. **Monorepo → router splits per package.** A workspace with a Next app, a Vue admin, and a shared TS lib routes each package to its agent; `frontend-developer` (router) coordinates cross-package changes (shared design tokens, a workspace-wide dependency bump).
6. **Dependency presence beats file count for new/empty trees.** A freshly scaffolded app with `@angular/core` in `package.json` but few components is Angular; do not wait for an extension census.
7. **Still ambiguous → router.** When two frameworks have comparable volume with no dominant config, dispatch `frontend-developer:frontend-developer` and let it split the work.

## Census Snippet

When config files are absent or ambiguous, count tracked component sources (never
`node_modules/`, `dist/`, `.next/`, `build/`, `.svelte-kit/`, vendored dirs):

```bash
git ls-files | grep -E '\.(tsx|jsx|vue|svelte|component\.ts)$' \
  | sed -E 's/.*\.(tsx|jsx|vue|svelte)$/\1/; s/.*\.component\.ts$/angular/' \
  | sort | uniq -c | sort -rn
```

Route to the dominant framework's agent if it holds >70% of component files;
otherwise use the router.

## Web vs Native Precedence (D6)

When a task carries **both** web markers (`.ts`/`.tsx`/`.jsx`/`package.json`/
`tsconfig.json`/framework configs) **and** native markers (`.swift`/`.xcodeproj`/
`Package.swift`/native module directories), route the **app/UI layer to
`frontend-developer:frontend-developer`** and the **native-module layer to
`apple-developer:*`** (forward-reference; if installed). The deciding question is
*which layer the change targets*: UI/component/state/styling/build-tooling work is
web (front-end wins); a native module, bridging header, or platform-API binding is
native (Apple wins). React Native / Expo splits the same way — JS/TS surface to the
(optional) `react-native-developer`, native modules deferred to `apple-developer:*`.
Default to `frontend-developer` for ambiguous pure-JS/TS web work.

Backend boundaries (HTTP/GraphQL server, DB, auth) route to `backend-developer:*`
(forward-reference; if installed) — surface the boundary to the orchestrator when
that plugin is absent. Neither `apple-developer:*` nor `backend-developer:*` ever
appears in a `tools: Task(...)` list; they are documented handoffs only.

The full precedence paragraph and the developer.md companion patch live in
`docs/companion-patch-developer.md`.

## Related Skills

- `workflow-integration/SKILL.md` — how the routed agent participates in DV (screenshot evidence)
- `model-selection.md` — model/effort to pass with the routed `Task()` call
- `version-feature-matrix.md` — framework/feature floors once the framework is known
- `severity-matrix.md` — the P0–P3 scale a routed review uses
