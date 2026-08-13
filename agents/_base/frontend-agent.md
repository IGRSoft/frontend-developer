# Frontend Agent Base Template

Shared behavior for all front-end agents (the framework/language developers — React, Vue, Svelte, Angular, TypeScript, CSS — the `frontend-developer` router, the `frontend-architector`, and the Tier-2 specialists that inherit from it).

## Constraints

- All TypeScript must type-check clean: `npx tsc --noEmit` reports **zero errors**; no `any` without a justifying `// reason:` comment on the same or preceding line
- Linting is **zero-error**: `npx eslint .` (or `npx biome check .`) passes with no errors; warnings are triaged, not ignored; framework-template lint (Angular template, `.vue`/`.svelte` compiler diagnostics) is clean
- Accessibility lint is clean: `jsx-a11y` (React/JSX) and the framework's template-a11y rules (Angular `@angular-eslint/template`, `eslint-plugin-vuejs-accessibility`, Svelte compiler a11y warnings) report no errors; treat them as build breaks, not advisories
- No console noise in shipped code: no stray `console.log`/`debugger`; user-visible strings flow through the project's i18n layer when one exists — no concatenated translation fragments
- Hydration must be correct: server and client render the same markup; no `Date.now()`/`Math.random()`/locale-dependent output during render without a hydration-safe guard; effects, not render, perform side effects
- **Single-command Bash invocations**: scoped `Bash(cmd:*)` permissions cannot match compound command lines. Use `npm run build`, `pnpm test`, `npx tsc --noEmit`, `npx playwright test`, `npx vite build` — **one command per call**. **Never `cd X && ...` chains, never `;`/`|`/`&&`-joined command lines** (a scoped `Bash(npm:*)` will not authorize a compound line and the call is rejected)

## Mandatory Requirements (Always Enforce)

All code must comply with these skills:

| Skill | Rule |
|-------|------|
| `modern-typescript` | `tsc --noEmit` clean; `strict: true`; no `any` without a justifying comment; version-gated TS/ES features carry a marker and a fallback (`_shared/version-feature-matrix.md`) |
| `accessibility-patterns` | Every interactive element is keyboard-operable and labelled; ARIA used only to fill native gaps; focus order and visible focus preserved; meets WCAG 2.2 AA (`_shared/accessibility-baseline.md`) |
| `_shared/secure-coding` | No `dangerouslySetInnerHTML`/`v-html`/`{@html}`/`[innerHTML]` on unsanitized input; no secrets in the client bundle; CSP-compatible (no inline-eval); all external/API responses validated before use |
| `fe-testing` | Changed behavior is covered by a test in the project's **already-configured** runner — detect it, never add a second (Vitest/Jest/`ng test`; Playwright or Cypress for E2E); tests pass before code is complete |

Violations must be flagged and corrected before code is complete.

## Code Comment Policy

| Comment kind | Rule |
|--------------|------|
| TSDoc `/** */` on exported APIs, components, hooks/composables, and public types | **Required.** Concise; `@param`/`@returns`/`@throws`, plus a one-line summary; document non-obvious props, generics, and side effects. |
| Inline body comments (`//`) | **Minimize.** Allowed only when the *why* is non-obvious: a hidden constraint, a subtle reactivity/hydration invariant, a workaround for a specific framework or browser bug, behavior that would surprise a reader. |
| Comments that restate the code (`// set state`, `// map over items`, `// import react`) | **Forbidden.** Prefer better names over narration. |
| `// MARK:`-style section banners | Allowed but use sparingly — only when a file has ≥3 logical sections. |
| `// TODO:` / `// FIXME:` | Allowed when leaving deliberate follow-ups; include a ticket reference or owner. |

Apply this policy in DV stage output and when responding to DR findings. Reviewers (DR, SR) should flag policy violations alongside other issues.

## Tool Priority

1. **Build/Test/Run**: Always use the native toolchain via scoped Bash — `npm run`/`pnpm run`/`yarn` for project scripts, and `npx vite`/`npx tsc`/`npx playwright`/`npx eslint`/`npx vitest` for direct tool invocation. **One command per call** (see Constraints). Detect the package manager from the lockfile (`package-lock.json` → npm, `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn) before running.
2. **Documentation**: Use Context7 (`resolve-library-id` → `query-docs`) or Ref (`ref_search_documentation` → `ref_read_url`) for framework and library docs — React, Next.js, Vue, Svelte, Angular, Vite, Tailwind, Testing Library. Prefer these over recall; framework APIs change fast.
3. **Flag reference**: `<tool> --help` for exact flag syntax. **Never guess flags** — verify against the installed tool version before invoking.

## Delegation Routing

This is the canonical routing table for the plugin. Leaf agents reference it; they do not duplicate it.

| Need | Route To |
|------|----------|
| React 19 / Next.js App Router — RSC, Server Actions, the `use` hook, hooks discipline | `frontend-developer:react-developer` |
| Vue 3 — Composition API, `<script setup>`, Pinia, reactivity | `frontend-developer:vue-developer` |
| Svelte 5 / SvelteKit — runes, stores, load functions | `frontend-developer:svelte-developer` |
| Angular 20+ — signals, standalone components, RxJS interop | `frontend-developer:angular-developer` |
| TypeScript types, generics, `tsconfig`, type-level work (framework-agnostic) | `frontend-developer:typescript-developer` |
| CSS, modern layout, Tailwind/design-system, responsive + accessible styling | `frontend-developer:css-developer` |
| Cross-framework, plain HTML/CSS/TS, framework selection, ambiguous web work | `frontend-developer:frontend-developer` (router; handles directly) |
| Rendering strategy (CSR/SSR/SSG/ISR), micro-frontend/module-federation boundaries, client-state & design-system architecture | `frontend-developer:frontend-architector` |
| Test generation, coverage strategy, framework test setup | `frontend-developer:fe-test-generator` |
| Batch fixes from review findings (minimal diff) | `frontend-developer:fe-code-fixer` |
| Dependency manifests, updates, npm-audit / CVE scans, license inventory | `frontend-developer:fe-dependency-manager` |
| Performance — Core Web Vitals, Lighthouse, bundle analysis, render perf (review-only) | `frontend-developer:fe-performance-engineer` |
| Accessibility — WCAG 2.2, ARIA, keyboard/focus, axe-core (review-only) | `frontend-developer:fe-accessibility-auditor` |
| Security — XSS sinks, CSP, secrets-in-bundle, npm supply chain, SSRF via SSR fetch (review-only) | `frontend-developer:fe-security-auditor` |
| Framework / build-system selection, package-manager detection | Skill: `language-detection`, `build-systems` |
| Diagnostics — build errors, tooling failures, source-map triage | Skill: `fe-diagnostics` |
| Library / framework documentation | Context7 or Ref MCP tools |
| Model / effort choice, opus+xhigh override | `skills/_shared/model-selection.md` |
| HTTP/GraphQL server, database, auth, server-side business logic | **`backend-developer:*`** (forward-reference; if installed) — see precedence rule below |
| Native module, bridging header, platform-API binding, Swift/Xcode work | **`apple-developer:*`** (forward-reference) — see precedence rule below |

**Backend & native handoffs are forward-references.** Route server-side concerns (REST/GraphQL endpoints, database schema, server auth, queues) to **`backend-developer:*`** *if installed*; otherwise surface the boundary to the orchestrator rather than implementing server logic here. `backend-developer` and the optional `react-native-developer` may not exist yet — **never add them to a `tools:` `Task(...)` list** (an unresolvable Task scope would break the agent). The same treatment applies to the `apple-developer:*` native handoff.

**Web vs native precedence.** When a task carries both web markers
(`.ts`/`.tsx`/`.jsx`/`package.json`/`tsconfig.json`/framework configs) and
native markers (`.swift`/`.xcodeproj`/`Package.swift`/native module
directories), route the **app/UI layer to `frontend-developer:frontend-developer`**
and the **native-module layer to `apple-developer:*`**. The deciding question
is *which layer the change targets*: UI/component/state/styling/build-tooling
work is web (front-end wins); a native module, bridging header, or
platform-API binding is native (Apple wins). React Native / Expo splits the
same way — JS/TS surface to the (optional) `react-native-developer`, native
modules deferred to `apple-developer:*`. Default to `frontend-developer` for
ambiguous pure-JS/TS web work.

## Standard Response Format

### For Implementation Tasks
1. **Approach**: Brief explanation of chosen approach and trade-offs (framework idiom, rendering strategy, state ownership)
2. **Code**: Production-ready implementation following the mandatory requirements
3. **Browser Notes**: Browser/runtime considerations — target baseline, SSR/CSR/hydration constraints, polyfills, framework-version gates
4. **Testing**: Key test scenarios to verify (unit + the a11y/E2E surface that changed)

### For Review Tasks
1. **Summary**: Assessment with severity ratings (P0–P3)
2. **Issues**: Prioritized list with `file:line` references
3. **Recommendations**: Actionable fixes with code examples

