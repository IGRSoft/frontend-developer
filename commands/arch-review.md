---
description: Review a frontend codebase's architecture — layering, state ownership, data-fetching seams, coupling, bundle split
argument-hint: [scope: file/dir/PR#/branch — default: repo root] [--strategy csr|ssr|ssg|isr|islands] [--deep]
allowed-tools: Read, Glob, Grep, Bash
estimated-cost:
  min-tokens: 4000
  max-tokens: 24000
  model-distribution:
    haiku: 10%
    sonnet: 55%
    opus: 35%
---

# Architecture Review
<!-- Updated: July 2026 -->

Review an existing frontend codebase against the architecture it is *supposed* to have: does each route actually render the way the strategy says, does state live in the tier that owns it, are data fetches at the seam or scattered through leaves, are components coupled through anything other than props and events, and does the route/bundle structure match the split the app needs. Produces severity-rated findings (P0-P3) with `file:line` evidence and a concrete fix per finding — read-only, no edits.

[Extended thinking: Architecture drift is invisible in a per-file code review. Each individual `useEffect(fetch)` in a leaf component is defensible; two hundred of them mean the app has no data-fetching seam. A single `'use client'` looks harmless; one at the root of the tree means the whole RSC boundary collapsed and the SSR strategy is decorative. So this command works top-down: detect the architecture that is actually implemented (from config, route layout, and directive placement), compare it against the intended strategy (from `.context/arch-selection.md`, the `--strategy` flag, or an inferred baseline), and only then look for the specific violations that gap predicts. Findings must carry `file:line` evidence — an architecture claim without a file reference is an opinion. The guardrail against over-reaction is explicit: a boundary violation gets the smallest fix that restores the boundary, never a wholesale rewrite recommendation, unless the mismatch is systemic and named as such.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Detect before judging.** Establish the *implemented* architecture from evidence (build config, route tree, `'use client'` / island directive placement, store files, loader usage) before comparing it to any intended strategy. Report detection confidence (high/medium/low) and the evidence.
2. **Read-only.** This command MUST NOT write or edit source. It produces findings and fixes as text. Remediation is a separate, explicit step (`/frontend-developer:fix-refactor`, `/frontend-developer:fix-modernize`).
3. **Every finding carries `file:line`.** A finding without a concrete location is speculation — drop it. Aggregate findings ("47 leaf components fetch directly") cite at least three representative locations plus the count.
4. **Compare against a stated intent.** Resolve the intended strategy from `.context/arch-selection.md`, then `--strategy`, then inference from the framework's own conventions. Print which source was used. Do NOT invent an intent to justify findings.
5. **Smallest fix that restores the boundary.** Recommend the minimal change per violation. A wholesale architecture switch is only proposed when the mismatch is systemic, and then it is labelled **Deep Refactor** with named risk points and a phased path — never as a casual suggestion.
6. **Severity is from the shared matrix.** Rank P0-P3 per `skill: severity-matrix`; do not invent local severity semantics.
7. **No manufactured findings.** If the architecture is sound, say so plainly. Do NOT pad the report with P3 preferences.
8. **Commands route, they do not orchestrate.** This command names `frontend-developer:frontend-architector` as the owner so Claude routes the work; it does not call `Task` itself.
9. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Review the whole repo against its recorded/inferred strategy
/frontend-developer:arch-review

# Review one area
/frontend-developer:arch-review src/features/checkout

# Review the architecture impact of a PR or branch
/frontend-developer:arch-review 142
/frontend-developer:arch-review feature/ssr-migration

# Assert the intended strategy explicitly
/frontend-developer:arch-review src/ --strategy ssr

# Add migration planning when the mismatch is systemic
/frontend-developer:arch-review --deep
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `scope` | repo root | File, directory, PR number, or branch. See Scope Resolution. |
| `--strategy csr\|ssr\|ssg\|isr\|islands` | resolved | Assert the intended rendering strategy instead of resolving it from `.context/arch-selection.md` or inference. |
| `--deep` | off | Add a Deep Refactor section: phased migration path, coexistence strategy, and risk points. Use only when the mismatch is systemic. |

## Scope Resolution

Resolve the file set **once**, top-down — first applicable rule wins:

1. **Explicit path** — a file or directory: review it plus its route/config context (the nearest `package.json`, build config, and route root above it).
2. **PR number** (bare integer) — `gh pr diff <N> --name-only`. If `gh` is unavailable, print the install hint and fall back to rule 3.
3. **Branch name** — `git diff --name-only $(git merge-base HEAD <branch>)..<branch>`.
4. **No argument** (default) — the repo root: all source under the detected app directories.

Exclude `node_modules/`, `dist/`, `build/`, `.next/`, `.svelte-kit/`, `.nuxt/`, `coverage/`. Print the resolved scope and file count before reviewing. For PR and branch scopes, still read the surrounding architecture context — an architecture review of a diff without its context is meaningless.

## Step 1: Detect the Implemented Architecture

Use `skill: language-detection` for the framework, then read the structural signals:

| Signal | Implies |
|--------|---------|
| `next.config` + `app/` dir + `'use server'`/`'use client'` markers | Next.js App Router — RSC/streaming |
| `next.config` + `pages/` + `getServerSideProps`/`getStaticProps` | Next.js Pages Router — SSR/SSG |
| `nuxt.config` + `server/`, route rules | Nuxt SSR/ISR |
| `svelte.config` + `+page.server.ts` / `+page.ts` loaders | SvelteKit SSR or prerender |
| `angular.json` + `@angular/ssr` / `provideServerRendering` | Angular SSR/hydration |
| `vite.config` + single `index.html` + client router only | CSR SPA shell |
| `astro.config` + `client:load`/`client:visible` | Islands / partial hydration |
| `ModuleFederationPlugin` / `@module-federation/*` / `remoteEntry.js` | Module federation |
| `pnpm-workspace.yaml` / `workspaces` in `package.json` | Monorepo |
| Store files (`store.ts`, `*.slice.ts`, Pinia `defineStore`, NgRx `createFeature`) | Global client-state tier in use |
| `@tanstack/react-query`, `rtk-query`, framework loaders | Server-cache tier in use |

Record: implemented rendering mode(s), state tiers present, repo shape, framework versions, and confidence with evidence. A **low-confidence** detection is reported as such rather than guessed into certainty.

## Step 2: Resolve the Intended Architecture

Precedence, first hit wins — print which one was used:

1. `--strategy` flag.
2. `.context/arch-selection.md` (written by `/frontend-developer:arch-select`) — read the Decision Summary table.
3. Inference from the framework's own conventions (e.g. a Next.js App Router project is *intended* to render on the server and use RSC boundaries; a Vite SPA is intended to be CSR).

If intent cannot be resolved at all, review against framework conventions only and label the report **"intent inferred — findings are convention-based."**

## Step 3: Review Dimensions

Check each dimension against the intended architecture. These are the frontend architecture failure classes worth a finding.

### 3.1 Layering

- Leaf/presentational components importing from `api/`, `services/`, or a store directly instead of receiving props.
- Route/page modules containing business logic that belongs in a model/domain layer.
- Cross-feature imports that reach into another feature's internals (`features/a/model/internal.ts` imported from `features/b/`) instead of its public entry.
- Shared design-system components importing app-specific code — the dependency arrow must point one way only.

### 3.2 State Ownership

- **Server data mirrored into a global client store** (the flagship anti-pattern): a store slice populated by a fetch, plus manual refetch/invalidate glue. Fix: move to the server-cache tier.
- Global store used for state one subtree needs — should be local or context.
- Shareable state (filters, tabs, pagination, sort) held in a store instead of the URL — breaks deep links and the back button.
- Derived state stored instead of computed (`useEffect` writing state that a `useMemo`/`computed`/`$derived` should produce).
- Duplicate sources of truth for the same entity across two stores or a store plus a query cache.
- See `skill: react-state`, `skill: vue-state`, `skill: svelte-runes`, `skill: angular-signals`.

### 3.3 Data-Fetching Boundaries

- Fetches scattered through leaf components instead of at the route/loader seam (cite representative locations plus the count).
- Request waterfalls: a child fetch that cannot start until a parent's fetch resolves, where both could be issued at the seam.
- Server-side data fetched again on the client after hydration (double fetch).
- Missing loading/error states at the seam — a route that can render an unhandled rejection.
- No cancellation/abort on a fetch tied to a component that can unmount mid-flight.

### 3.4 Rendering & Hydration Boundaries

- `'use client'` (or an island directive) placed so high in the tree that the server-rendering strategy is effectively cancelled — cite the file and the subtree size it forces client-side.
- Server-rendered branches reading `window`/`document`/`localStorage`/`navigator` or non-deterministic values (`Date.now()`, `Math.random()`, locale-dependent formatting) without a client-only guard — hydration mismatch.
- Secrets or server-only modules imported across the client boundary (cross-check `skill: secure-coding`).
- A route whose rendering mode contradicts its data freshness (SSG on per-user data; SSR on content identical for every user).
- Version-gated features used without a fallback (`skill: version-feature-matrix`).

### 3.5 Component Coupling

- Prop-drilling more than ~3 levels for state a context or store should own.
- Components coupled through module-level mutable state or a shared singleton instead of explicit inputs.
- Parents reaching into child internals (`ref` reaching, DOM queries across component boundaries, `ViewChild` used to drive a child's logic).
- God components: one file owning fetching, transformation, layout, and interaction for a whole route.
- Circular imports between feature modules.

### 3.6 Route & Bundle Structure

- No route-level code splitting: one entry chunk carrying the whole app (`skill: bundling-optimization`).
- Heavy dependencies (charting, editors, date/i18n bundles, PDF) in the shared entry chunk instead of a lazy route boundary.
- Route tree that does not match the navigation model (deep nesting with no shared layout reuse, or a flat tree duplicating layout).
- Barrel files (`index.ts` re-exporting everything) defeating tree-shaking across feature boundaries.
- In a monorepo: a package importing across a workspace boundary it does not declare as a dependency; federation remotes with unpinned shared singletons.

## Step 4: Route the Analysis

Route to `frontend-developer:frontend-architector` (`subagent_type="frontend-developer:frontend-architector"`, model `opus`):

"Read-only architecture review of `{scope}` ({N} files, framework: {framework} {version}). Implemented architecture detected as: {detection + evidence + confidence}. Intended architecture: {intent, and which source it came from}. Review the six dimensions — layering, state ownership, data-fetching boundaries, rendering/hydration boundaries, component coupling, and route/bundle structure — against that intent. Every finding MUST carry `file:line` evidence; aggregate findings cite at least three representative locations plus a count. Rank each finding P0-P3 per `skill: severity-matrix` and give the **smallest** concrete fix that restores the boundary. Do NOT recommend a wholesale architecture switch unless the mismatch is systemic — if it is, label it Deep Refactor with phased steps and named risk points{, and produce that migration plan (--deep was passed)}. Do NOT edit any file. If the architecture is sound, say so plainly rather than manufacturing P3 items."

## Severity

Ranked per `skill: severity-matrix`. Architecture-specific reading:

| Severity | Meaning here |
|----------|--------------|
| **P0** | The intended architecture is functionally cancelled, or the violation ships a defect: hydration mismatch on a live route, secrets across the client boundary, whole app client-rendered under an SSR strategy. |
| **P1** | A boundary is broken in a way that blocks testing or scaling: no data-fetching seam, server data in a global store, no route splitting on a large app. |
| **P2** | Convention violation with real maintenance cost: prop-drilling, cross-feature reach-in, derived-state-in-effect, barrel-file tree-shaking loss. |
| **P3** | Structural preference with no current defect: folder naming, slice granularity. |

## Output Format

```markdown
## Architecture Review

**Scope:** {paths / PR# / branch} ({N} files)
**Framework:** {name} {version}
**Implemented:** {detected architecture} (confidence: {high|medium|low})
**Intended:** {strategy} (source: {arch-selection.md | --strategy | inferred})
**Verdict:** PASS / WARN / FAIL

### Summary
{One or two sentences. If sound: "The implementation matches the intended architecture — no boundary violations found." Otherwise the headline gap.}

| Priority | Count |
|----------|-------|
| P0 | {n} |
| P1 | {n} |
| P2 | {n} |
| P3 | {n} |

### Detection Evidence
| Signal | Location | Implies |
|--------|----------|---------|

### P0 — Architecture Broken
| File:Line | Dimension | Issue | Smallest fix |
|-----------|-----------|-------|--------------|

### P1 — Boundary Violations
{same table shape}

### P2 — Convention Violations
{same table shape}

### P3 — Preferences
{same table shape}

### Dimension Scorecard
| Dimension | Status | Note |
|-----------|--------|------|
| Layering | ✅ / ⚠️ / ❌ | |
| State ownership | ✅ / ⚠️ / ❌ | |
| Data-fetching seams | ✅ / ⚠️ / ❌ | |
| Rendering / hydration | ✅ / ⚠️ / ❌ | |
| Component coupling | ✅ / ⚠️ / ❌ | |
| Route / bundle structure | ✅ / ⚠️ / ❌ | |

### Top 3 Recommendations
1. {highest-leverage fix}
2. {next}
3. {next}

<!-- --deep only, and only when the mismatch is systemic: -->
### Deep Refactor Plan
- **Current → Target:** {transition map}
- **Phases:** {ordered, each independently buildable and testable}
- **Coexistence:** {how old and new interoperate during the transition}
- **Risk points:** {hydration mismatch, singleton drift, design-system contract breaks, bundle regressions}

<!-- When a tool was unavailable: -->
### Reduced-Depth Notes
- {tool} unavailable — {dimension} reviewed statically only. Install: {hint}.
```

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect on review |
|--------------|--------------|------------------|
| `node` / `npm` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | version gates read from `package.json` text only; note reduced confidence |
| `gh` (PR scope) | `brew install gh` then `gh auth login` | fall back to a branch diff against the default branch |
| `git` | install git | explicit-path scope only; PR/branch scopes unavailable |
| bundle analyzer (`npx vite-bundle-visualizer`, `npx source-map-explorer`) | `npm install -D rollup-plugin-visualizer` | route/bundle dimension reviewed from imports and route config only — note reduced depth |
| `npx knip` / `npx madge` (cycle + dead-export detection) | `npm install -D knip` (or `madge`) | circular-import and barrel findings come from manual import tracing — note reduced depth |

Tool-missing is always a skip-with-note, never a hard failure: print the hint, record it under **Reduced-Depth Notes**, and continue. The command reports "cannot review" only when the resolved scope contains no web sources.

## Error Handling

### No web sources in scope
```
Note: No reviewable web sources in the resolved scope.
Resolved scope: {scope}
Suggestion: pass an explicit path, e.g. /frontend-developer:arch-review src/
```

### Intent cannot be resolved
Not an error. Review against framework conventions, and label the report "intent inferred — findings are convention-based." Suggest running `/frontend-developer:arch-select` to record an explicit intent for future reviews.

### Detection confidence is low
Not an error. Report the low confidence, list the ambiguous signals, and scope findings to what the evidence actually supports. Do not upgrade a guess into a P0.

### Architecture is sound
Report PASS with the dimension scorecard and no findings. Per Rule 7, do not backfill P3 items.

## See Also

- `skill: language-detection` — framework detection used by Step 1.
- `skill: severity-matrix` — P0-P3 definitions used for ranking.
- `skill: version-feature-matrix` — version gates and fallbacks checked in dimension 3.4.
- `skill: bundling-optimization` — route-splitting and chunking rules behind dimension 3.6.
- `skill: secure-coding` — client-boundary secret leakage cross-checked in dimension 3.4.
- `/frontend-developer:arch-select` — record the intended architecture this review compares against.
- `/frontend-developer:review-code` — per-file correctness review; this command is the structural layer above it.
- `/frontend-developer:fix-refactor` — apply the structural fixes this review recommends.
- `/frontend-developer:analyze-tech-debt` — quantify and prioritize the debt these findings represent.
- `/frontend-developer:build-test` — confirm the project still builds and tests green after any fix lands.
