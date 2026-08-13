# Skills Index

Root index for all frontend-developer skills (TypeScript, React, Vue, Svelte,
Angular, Styling, Tooling, Quality, and shared cross-cutting patterns).
**31 SKILL.md across 8 domains, plus shared references.** Start at
[`SKILL.md`](SKILL.md) for the routing entry point.

## Domains

| Directory | Canonical entry | Skills | Description |
|-----------|-----------------|--------|-------------|
| [_shared/](_shared/_index.md) | [`_index.md`](_shared/_index.md) | 8 + refs | Cross-cutting: workflow integration, secure coding, versions, routing, severity, testing, accessibility baseline |
| [typescript/](typescript/typescript-skills/SKILL.md) | [`typescript-skills/SKILL.md`](typescript/typescript-skills/SKILL.md) | 1 + 3 leaves | TS 5.x features, strict typing, tsconfig discipline, ES2024 surface |
| [react/](react/react-skills/SKILL.md) | [`react-skills/SKILL.md`](react/react-skills/SKILL.md) | 1 + 3 leaves | React 19 RSC/Actions/`use`, hooks, client state, render performance |
| [vue/](vue/vue-skills/SKILL.md) | [`vue-skills/SKILL.md`](vue/vue-skills/SKILL.md) | 1 + 2 leaves | Vue 3.5 Composition API, `<script setup>`, Pinia state |
| [svelte/](svelte/svelte-skills/SKILL.md) | [`svelte-skills/SKILL.md`](svelte/svelte-skills/SKILL.md) | 1 + 2 leaves | Svelte 5 runes, SvelteKit routing/load/actions |
| [angular/](angular/angular-skills/SKILL.md) | [`angular-skills/SKILL.md`](angular/angular-skills/SKILL.md) | 1 + 2 leaves | Angular 20+ signals, standalone components, control flow, RxJS interop |
| [styling/](styling/styling-skills/SKILL.md) | [`styling-skills/SKILL.md`](styling/styling-skills/SKILL.md) | 1 + 3 leaves | Modern CSS, Tailwind design systems, responsive + accessible CSS |
| [tooling/](tooling/tooling-skills/SKILL.md) | [`tooling-skills/SKILL.md`](tooling/tooling-skills/SKILL.md) | 1 + 3 leaves | Vite/build systems, diagnostics, bundle optimization |
| [quality/](quality/quality-skills/SKILL.md) | [`quality-skills/SKILL.md`](quality/quality-skills/SKILL.md) | 1 + 3 leaves | Testing, accessibility patterns, Core Web Vitals performance |

## All Skills

### _shared

| Skill | Path | Description |
|-------|------|-------------|
| **secure-coding** | [`_shared/secure-coding/SKILL.md`](_shared/secure-coding/SKILL.md) | Web security rules and bug-class defenses — XSS sinks, CSP, CSRF, clickjacking, secrets-in-bundle, `dangerouslySetInnerHTML`, npm supply chain |
| **workflow-integration** | [`_shared/workflow-integration/SKILL.md`](_shared/workflow-integration/SKILL.md) | Integrating with corpflow; DV screenshot gate via web_adapter, Lighthouse/axe supporting evidence |
| version-feature-matrix | [`_shared/version-feature-matrix.md`](_shared/version-feature-matrix.md) | Framework/language versions → minimum versions and fallbacks (canonical lookup) |
| language-detection | [`_shared/language-detection.md`](_shared/language-detection.md) | Marker → framework → agent routing table, detection priority, tie-breaking |
| model-selection | [`_shared/model-selection.md`](_shared/model-selection.md) | Per-agent model/effort/maxTurns assignments and opus+xhigh override paths |
| severity-matrix | [`_shared/severity-matrix.md`](_shared/severity-matrix.md) | Severity levels, P0-P3 review priorities, coverage requirements |
| testing-principles | [`_shared/testing-principles.md`](_shared/testing-principles.md) | Test pyramid, per-framework matrix, quality gates, anti-patterns |
| accessibility-baseline | [`_shared/accessibility-baseline.md`](_shared/accessibility-baseline.md) | WCAG 2.2 success-criteria floors every UI must meet |

### typescript

| Skill | Path | Description |
|-------|------|-------------|
| **typescript** (canonical) | [`typescript/typescript-skills/SKILL.md`](typescript/typescript-skills/SKILL.md) | TypeScript skills navigation and the canonical selection table for TS 5.x |
| **modern-typescript** | [`typescript/modern-typescript/SKILL.md`](typescript/modern-typescript/SKILL.md) | TS 5.x feature gates: `using`/`await using`, `const` type params, `satisfies`, standard decorators, `NoInfer` |
| **ts-typing** | [`typescript/ts-typing/SKILL.md`](typescript/ts-typing/SKILL.md) | Eliminating `any`, generics, discriminated unions, narrowing, utility-type patterns |
| **ts-config** | [`typescript/ts-config/SKILL.md`](typescript/ts-config/SKILL.md) | Strict-mode tsconfig flags, module resolution, project references, build vs emit |

### react

| Skill | Path | Description |
|-------|------|-------------|
| **react** (canonical) | [`react/react-skills/SKILL.md`](react/react-skills/SKILL.md) | React skills navigation and the canonical selection table for React 19 |
| **modern-react** | [`react/modern-react/SKILL.md`](react/modern-react/SKILL.md) | React 19 RSC, Server Actions, `use`, `useActionState`/`useOptimistic`, React Compiler, hook rules |
| **react-state** | [`react/react-state/SKILL.md`](react/react-state/SKILL.md) | Server vs client state; Redux Toolkit, Zustand, TanStack Query selection |
| **react-performance** | [`react/react-performance/SKILL.md`](react/react-performance/SKILL.md) | Wasted-render diagnosis, `memo`/`useMemo`/`useCallback`, Suspense, list virtualization |

### vue

| Skill | Path | Description |
|-------|------|-------------|
| **vue** (canonical) | [`vue/vue-skills/SKILL.md`](vue/vue-skills/SKILL.md) | Vue skills navigation and the canonical selection table for Vue 3.5 |
| **vue-composition** | [`vue/vue-composition/SKILL.md`](vue/vue-composition/SKILL.md) | `<script setup>`, refs vs reactive, composables, `defineModel`, reactivity rules |
| **vue-state** | [`vue/vue-state/SKILL.md`](vue/vue-state/SKILL.md) | Pinia stores: setup vs options stores, getters, actions, SSR-safe state |

### svelte

| Skill | Path | Description |
|-------|------|-------------|
| **svelte** (canonical) | [`svelte/svelte-skills/SKILL.md`](svelte/svelte-skills/SKILL.md) | Svelte skills navigation and the canonical selection table for Svelte 5 |
| **svelte-runes** | [`svelte/svelte-runes/SKILL.md`](svelte/svelte-runes/SKILL.md) | `$state`/`$derived`/`$effect`/`$props`/`$bindable`, snippets, event attributes |
| **sveltekit** | [`svelte/sveltekit/SKILL.md`](svelte/sveltekit/SKILL.md) | Routing, `load` functions, form actions, server vs universal code, env modules |

### angular

| Skill | Path | Description |
|-------|------|-------------|
| **angular** (canonical) | [`angular/angular-skills/SKILL.md`](angular/angular-skills/SKILL.md) | Angular skills navigation and the canonical selection table for Angular 18+ |
| **angular-signals** | [`angular/angular-signals/SKILL.md`](angular/angular-signals/SKILL.md) | `signal`/`computed`/`effect`, `input`/`output`/`model`, standalone, `@if`/`@for`/`@defer` |
| **angular-rxjs** | [`angular/angular-rxjs/SKILL.md`](angular/angular-rxjs/SKILL.md) | RxJS-signal bridge, `toSignal`/`toObservable`, when streams still win |

### styling

| Skill | Path | Description |
|-------|------|-------------|
| **styling** (canonical) | [`styling/styling-skills/SKILL.md`](styling/styling-skills/SKILL.md) | Styling skills navigation and the canonical selection table |
| **modern-css** | [`styling/modern-css/SKILL.md`](styling/modern-css/SKILL.md) | Container queries, `:has()`, cascade layers, subgrid, native nesting, `@property` |
| **tailwind-design-system** | [`styling/tailwind-design-system/SKILL.md`](styling/tailwind-design-system/SKILL.md) | Token-driven Tailwind, theming, component classes vs utilities |
| **responsive-accessible-css** | [`styling/responsive-accessible-css/SKILL.md`](styling/responsive-accessible-css/SKILL.md) | Fluid layouts, reduced-motion, focus visibility, color-contrast |

### tooling

| Skill | Path | Description |
|-------|------|-------------|
| **tooling** (canonical) | [`tooling/tooling-skills/SKILL.md`](tooling/tooling-skills/SKILL.md) | Tooling skills navigation; routes "won't build", "won't bundle", "too big" symptoms |
| **build-systems** | [`tooling/build-systems/SKILL.md`](tooling/build-systems/SKILL.md) | Vite config, dev/build pipelines, framework CLIs, env handling |
| **fe-diagnostics** | [`tooling/fe-diagnostics/SKILL.md`](tooling/fe-diagnostics/SKILL.md) | Symptom → fix routing: build errors, hydration mismatch, HMR, source maps |
| **bundling-optimization** | [`tooling/bundling-optimization/SKILL.md`](tooling/bundling-optimization/SKILL.md) | Code-splitting, tree-shaking, bundle analysis, lazy loading |

### quality

| Skill | Path | Description |
|-------|------|-------------|
| **quality** (canonical) | [`quality/quality-skills/SKILL.md`](quality/quality-skills/SKILL.md) | Quality skills navigation: testing, accessibility, performance |
| **fe-testing** | [`quality/fe-testing/SKILL.md`](quality/fe-testing/SKILL.md) | Vitest, Playwright, Testing Library, framework detection, test pyramid |
| **accessibility-patterns** | [`quality/accessibility-patterns/SKILL.md`](quality/accessibility-patterns/SKILL.md) | ARIA patterns, focus management, keyboard nav, axe-core triage |
| **web-performance** | [`quality/web-performance/SKILL.md`](quality/web-performance/SKILL.md) | Core Web Vitals budgets (LCP/INP/CLS), Lighthouse, runtime profiling |

## Child Indexes

| Index Path | Contents |
|------------|----------|
| [`_shared/_index.md`](_shared/_index.md) | Shared skills: workflow, secure coding, versions, routing, severity, testing, accessibility baseline |
