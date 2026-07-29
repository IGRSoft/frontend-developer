---
description: Select a frontend architecture — rendering strategy, state management, component boundaries, framework, and repo shape
argument-hint: <feature or project description> [--scope feature|route|app] [--framework react|vue|svelte|angular]
allowed-tools: Read, Write, Glob, Grep
estimated-cost:
  min-tokens: 3500
  max-tokens: 18000
  model-distribution:
    haiku: 10%
    sonnet: 55%
    opus: 35%
---

# Architecture Selection
<!-- Updated: July 2026 -->

Choose the frontend architecture for a new feature, route tree, or whole app: **where each route renders** (CSR / SSR / SSG / ISR / islands / streaming), **which state tier owns each piece of data** (local / server-cache / global store / URL), **where the component boundaries fall**, **which framework fits**, and **whether the code lives in one app or a monorepo**. Produces a written decision record with a concrete directory layout, not a menu of options.

[Extended thinking: These five decisions are coupled, and picking them independently is how teams end up with a Next.js App Router app that fetches everything client-side into Redux, or a Vite SPA that needs SEO. Rendering choice constrains the data-fetching seam; the data-fetching seam determines whether a global store is needed at all; the store choice leaks into component boundaries; and the repo shape only matters once there is more than one deployable. So this command captures constraints once, routes the whole decision to a single architect that reasons across all five axes, then makes the choice concrete as a file tree the user can scaffold. The dominant failure mode is over-selection — reaching for SSR, a global store, and a monorepo for a three-screen app. The guardrail is explicit: pick the smallest structure that satisfies a stated constraint, and never add a runtime dependency the constraints do not demand.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Capture constraints before recommending.** Fill the Constraint Table from the argument plus a read of `package.json`, `tsconfig.json`, and the build config. Ask the user for any **critical** unknown (SEO requirement, auth model, data freshness) instead of assuming it. Do NOT recommend from the task title alone.
2. **Decide all five axes, or say why one does not apply.** Rendering, state, component boundaries, framework, and repo shape. A recommendation missing an axis is incomplete. Where an axis is already fixed by the existing codebase, record it as "inherited" with the evidence.
3. **Smallest viable structure wins.** Every escalation — SSR over SSG, a global store over server-cache, a monorepo over one app, a new framework — MUST name the specific constraint that forces it. No constraint, no escalation.
4. **No new runtime dependency without a stated trade-off.** Introducing Redux Toolkit, NgRx, a meta-framework, or module federation requires an explicit cost line (bundle, build, onboarding) that the user can reject.
5. **Rendering choices are per route group, not per app.** State the mode for each route group with its data-freshness reason (per-request / per-build / revalidate-N) and a fallback if the framework version does not support it.
6. **Version-gate every modern feature.** RSC, streaming SSR, ISR, `use`, signals, runes — cite the minimum version and a fallback per `skill: version-feature-matrix`. Never assert a feature is available without checking the project's installed versions.
7. **Commands route, they do not orchestrate.** This command names `frontend-developer:frontend-architector` as the owner so Claude routes the work; it does not call `Task` itself.
8. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Select the architecture for a new app from a description
/frontend-developer:arch-select "B2B analytics dashboard, auth-gated, no SEO, heavy charts"

# Select for a single feature inside an existing codebase
/frontend-developer:arch-select "multi-step checkout with saved drafts" --scope feature

# Select for a route tree, forcing the framework
/frontend-developer:arch-select "public product catalog + PDP" --scope route --framework react

# Whole-app decision including repo shape
/frontend-developer:arch-select "marketing site + docs + app shell, three teams" --scope app
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `description` | required | Feature or project scope in prose. Include SEO, auth, and data-freshness needs if you know them. |
| `--scope feature\|route\|app` | inferred | `feature` = one component tree (rendering/framework usually inherited); `route` = a route group; `app` = greenfield or whole-shell decision including repo shape. |
| `--framework react\|vue\|svelte\|angular` | auto | Fix the framework instead of selecting it. Use when the org has already standardized. |

## Step 1: Capture Constraints

Gather from the argument and a read of the repo. Use `skill: language-detection` for the marker → framework mapping; do not fork its logic.

| Constraint | How to determine | Why it matters |
|-----------|------------------|----------------|
| **Existing framework** | `package.json` deps (`next`, `nuxt`, `@sveltejs/kit`, `@angular/core`, `react`, `vue`, `svelte`) + build config | Fixes the framework axis and constrains rendering options |
| **SEO / shareability** | Ask. Public-indexed pages, OG previews, or auth-only? | The single strongest driver of CSR vs SSR/SSG |
| **Data freshness** | Ask. Per-request, per-build, revalidate-every-N, or realtime? | Selects SSR vs SSG vs ISR vs client polling/websocket |
| **Auth model** | Scan for auth middleware/route guards | Personalized HTML forces SSR or client-render behind a shell |
| **State shape** | Server-owned vs client-owned; cross-route persistence? | Selects the state tier (see Step 3) |
| **Interactivity density** | Whole page vs isolated widgets | Islands / partial hydration only pays off for sparse interactivity |
| **Team familiarity** | Existing patterns in the codebase as proxy | A pattern the team cannot maintain is the wrong pattern |
| **Dependency tolerance** | Existing dep count, bundle budget in CI | Gates Redux/NgRx/federation escalations |
| **Deployables & teams** | One app or several? Independent deploy cadence? | The only legitimate driver of monorepo / micro-frontend |
| **Target versions** | `package.json` versions vs `skill: version-feature-matrix` | Determines whether RSC/ISR/signals/runes are even available |

If a **critical** constraint (SEO, data freshness, auth) is unknown, ask before recommending. Non-critical gaps get a stated assumption in the output.

## Step 2: Rendering Strategy

Pick **per route group**. Most real apps mix modes.

| Strategy | Choose when | Trade-off | Version gate |
|----------|-------------|-----------|--------------|
| **CSR** (client render) | Auth-gated app shell, dashboards, no SEO | Slow first paint, large JS, blank-until-hydrate | none — any SPA build |
| **SSR** (per request) | Personalized **and** SEO-critical, fresh-per-request data | Server cost per request, TTFB, hydration-mismatch risk | meta-framework required (Next/Nuxt/SvelteKit/Angular SSR) |
| **SSG** (build time) | Marketing, docs, blogs — identical for all users | Stale until rebuild; unusable for per-user data | any meta-framework |
| **ISR / on-demand revalidate** | Large mostly-static catalogs that change occasionally | Cache-invalidation complexity; first-after-revalidate latency | Next.js / Nuxt route rules; check version |
| **Islands / partial hydration** | Mostly-static pages with a few interactive widgets | Cross-island coordination is awkward | Astro / Qwik / SvelteKit-with-care |
| **Streaming SSR (+ RSC)** | Big pages where above-the-fold must not wait on slow data | RSC boundary discipline, Suspense fallbacks, serialization cost | React 19 + Next App Router; fallback = SSR + skeleton |

For each route group record: mode, data-freshness reason, and the fallback mode if the project's framework version cannot support the first choice (`skill: version-feature-matrix`). Flag any server-rendered branch that reads `window`/`document`/`localStorage` or a non-deterministic value (`Date.now()`, `Math.random()`) without a client-only guard — that is the recurring hydration failure.

## Step 3: State Management

Escalate only when the cheaper tier genuinely cannot express the requirement.

| Tier | Mechanism | Choose when |
|------|-----------|-------------|
| **Local** | `useState` / `ref` / `$state` / `signal` | One component and its direct children |
| **Lifted / context** | context, provide-inject, Angular DI | A small subtree shares it; no cross-route persistence |
| **URL** | route + search params | Shareable, bookmarkable, back-button-correct (filters, tabs, pagination) |
| **Server-cache** | TanStack Query, RTK Query, framework loaders (`load`, route loaders, resolvers) | The server owns the data — cache, revalidate, mutate |
| **Global client store** | Zustand, Pinia, Svelte stores, NgRx / NgRx SignalStore, Redux Toolkit | Genuinely client-owned cross-route state (session UI, theme, multi-step drafts) |

Library selection inside the global tier, once the tier is justified:

| Library | Fits | Cost |
|---------|------|------|
| **Zustand** | React app wanting a tiny store, no boilerplate, selector-based reads | Little structure — conventions are on you |
| **Redux Toolkit** | Large React app needing devtools, middleware, strict action audit trail | Boilerplate + bundle; overkill under ~a dozen slices |
| **TanStack Query** | Any framework, server-owned data (this is a *server-cache*, not a global store) | Cache-key discipline required |
| **Pinia** | Vue/Nuxt client state; SSR-safe stores | Vue-only |
| **Svelte stores / runes** | Svelte client state; `$state` in runes mode covers most of it | Cross-component sharing needs deliberate module placement |
| **NgRx (Store or SignalStore)** | Angular apps with complex cross-feature state and effects | Heaviest ceremony; SignalStore is the lighter modern default |

**The dominant anti-pattern is server data copied into a global client store** — it desyncs and reimplements cache logic badly. Server data belongs in the server-cache tier. See `skill: react-state` (React), `skill: vue-state` (Pinia), `skill: svelte-runes`, `skill: angular-signals`.

## Step 4: Component Architecture & Boundaries

| Boundary | Decide |
|----------|--------|
| **Route vs component** | What is a route (owns data fetching + layout) vs a leaf component (props in, events out) |
| **Server vs client** | Where `'use client'` / island directives / hydration boundaries fall; keep them as low in the tree as possible |
| **Data-fetching seam** | Fetch at the route/loader level; leaf components take data as props. No fetch calls scattered through leaves |
| **Feature slices vs layer folders** | `features/<name>/{ui,model,api}` for app code; `components/` layer folders only for the shared design system |
| **Design-system surface** | Tokens → accessible primitives → composites. Exported component + prop + token names are a semver-gated public API (`skill: accessibility-patterns`, `skill: modern-css`) |
| **Test seams** | Which boundaries get component tests vs which get an E2E path (`skill: fe-testing`, `skill: testing-principles`) |

## Step 5: Framework Selection

Skip when `--framework` is set or a framework is already in `package.json` (record it as inherited).

| Framework | Fits | Against |
|-----------|------|---------|
| **React + Next.js** | SEO + personalization, RSC/streaming, largest hiring pool | RSC mental model, framework churn |
| **React + Vite (SPA)** | Auth-gated app shells, no SEO, fast dev loop | No SSR story without adding one later |
| **Vue + Nuxt** | Full-stack app with a gentler reactivity model; strong SSR defaults | Smaller ecosystem than React |
| **Svelte + SvelteKit** | Smallest bundles, least ceremony, runes reactivity | Smallest ecosystem; fewer off-the-shelf components |
| **Angular** | Large enterprise app, opinionated DI/tooling, long-lived teams | Heaviest ceremony; standalone+signals is the modern baseline |

Do NOT recommend a framework change for an existing codebase unless the constraints make the current one unworkable — say so explicitly, with the migration cost.

## Step 6: Repo Shape — Monorepo vs Single App

Default is **one app**. Escalate only on a real boundary:

| Signal | Shape |
|--------|-------|
| One deployable, one team | Single app. Stop here. |
| Shared design system or utils across 2+ deployables | Monorepo with workspaces (pnpm/npm/Yarn) + a published-internally package |
| Independent deploy cadence per team, or mixed framework versions during a migration | Monorepo **and** consider module federation |
| "The codebase feels big" | Not a reason. Use feature slices inside one app. |

Module federation is a deployment decision, not a code-organization one. If chosen, pin shared singletons (`singleton`, `requiredVersion`) — a mismatch means duplicate React and broken context — and require an error boundary per remote so a failed remote load degrades instead of white-screening. See `skill: build-systems` and `skill: bundling-optimization` for workspace and chunking impact.

## Step 7: Route the Decision

Route the captured constraints to the architect, which reasons across all five axes at once:

Route to `frontend-developer:frontend-architector` (`subagent_type="frontend-developer:frontend-architector"`, model `opus`):

"Select the frontend architecture for: {description}. Scope: {feature|route|app}. Constraints:\n```\n{constraint table from Step 1}\n```\nDecide **all five axes**: (1) rendering strategy per route group with a data-freshness reason and a version-gated fallback; (2) the state tier per state concern, and the library only if the global tier is justified; (3) component boundaries — route vs leaf, server/client split, data-fetching seam, design-system surface, test seams; (4) framework (or record the inherited one with evidence); (5) repo shape — single app unless a deployment or team boundary forces a monorepo. Apply the guardrail: pick the smallest structure that satisfies a stated constraint, and name the constraint that forces every escalation. Do not add a runtime dependency without an explicit trade-off line. Then produce a concrete directory/route tree with project-specific names, the key boundaries, and a testing strategy. If the user named a specific architecture, validate fit and report fit/mismatch with the closest alternative. Verify version-gated features against the project's installed versions."

Write the returned decision to `.context/arch-selection.md` using the Output Format below. Print the summary inline as well — do not make the user open the file to see the recommendation.

## Output Format

```markdown
## Architecture Selection: {feature or project}

**Scope:** {feature | route | app}
**Fit:** {fit | mismatch — with the closest alternative}

### Decision Summary
| Axis | Decision | Forcing constraint |
|------|----------|--------------------|
| Rendering | {per route group} | {SEO / freshness / auth} |
| State | {tier(s) + library if any} | {ownership / persistence} |
| Components | {slice or layer layout} | {reuse / team} |
| Framework | {name — selected or inherited} | {evidence} |
| Repo shape | {single app | monorepo | + federation} | {deployables / teams} |

### Rendering per Route Group
| Route group | Mode | Data freshness | Version gate | Fallback |
|-------------|------|----------------|--------------|----------|
| {/marketing/*} | SSG | per build | — | — |
| {/app/*} | CSR | client fetch | — | — |

### State Ownership
| State concern | Tier | Mechanism | Why not the cheaper tier |
|---------------|------|-----------|--------------------------|

### Structure
{directory / route tree with real file names}

### Key Boundaries
- **Hydration / client boundary:** {where 'use client' or islands start}
- **Data-fetching seam:** {loader / route / server component}
- **Design-system surface:** {exported components, props, tokens}
- **Federation singletons:** {pinned deps — or "n/a"}

### Testing Strategy
- {what gets component tests}
- {what gets an E2E path}
- {how server data is stubbed}

### Trade-offs Accepted
- {dependency or complexity added, and what it buys}

### Assumptions
- {any non-critical constraint that was assumed, so the user can correct it}

### Next Steps
1. {scaffold step — e.g. /frontend-developer:gen-component for the first primitive}
2. {verify the shell builds — /frontend-developer:build-test .}
3. {review once implemented — /frontend-developer:arch-review src/}
```

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect on selection |
|--------------|--------------|---------------------|
| `node` / `npm` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | version gates come from `package.json` text only; note reduced confidence |
| `npm ls` / installed `node_modules` | `npm install` in the project root | resolved versions unavailable — fall back to declared ranges in `package.json` |
| no `package.json` at all (greenfield) | not an error — proceed | framework axis is a genuine choice rather than inherited; say so |

A missing tool never blocks the recommendation: print the hint, record the reduced confidence in **Assumptions**, and continue. Selection is a reasoning task — it degrades in confidence, never into failure.

## Error Handling

### No description given
```
Error: /frontend-developer:arch-select needs a feature or project description.
Suggestion: /frontend-developer:arch-select "auth-gated analytics dashboard, no SEO"
```

### Critical constraint unknown
Not an error. Ask one focused question (SEO? data freshness? auth?) before recommending, rather than guessing — a rendering choice made on a guessed SEO requirement is the most expensive mistake this command can make.

### User named an architecture that does not fit
Not an error. Report `mismatch`, state the specific constraint it violates, and give the closest-fit alternative with the trade-off — do not silently substitute your own pick.

## See Also

- `skill: language-detection` — marker → framework → agent routing used by Step 1.
- `skill: version-feature-matrix` — canonical minimum versions and fallbacks for RSC, ISR, signals, runes.
- `skill: react-state` / `skill: vue-state` / `skill: svelte-runes` / `skill: angular-signals` — per-framework state selection detail.
- `skill: bundling-optimization` — route-splitting and chunking impact of the chosen structure.
- `skill: fe-testing` — test seams for the boundaries selected in Step 4.
- `/frontend-developer:arch-review` — verify an implementation against the strategy chosen here.
- `/frontend-developer:gen-component` — scaffold the first primitives of the recommended structure.
- `/frontend-developer:build-test` — confirm the scaffolded shell builds and tests green.
- `/frontend-developer:analyze-tech-debt` — when the selection surfaces an existing structure that must be paid down first.
