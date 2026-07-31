---
name: skills
description: >-
  Comprehensive front-end development skills for TypeScript, React, Vue,
  Svelte, Angular, modern CSS/styling, build tooling, and quality (testing,
  accessibility, performance). Use when writing React 19 / Next.js 15, Vue 3.5,
  Svelte 5 runes, or Angular 18+ code, authoring TypeScript, styling with modern
  CSS or Tailwind, configuring Vite/bundlers, writing Vitest/Playwright tests,
  auditing WCAG 2.2 accessibility, or tuning Core Web Vitals.
---

# Skills Index

## Overview

This collection provides guidance for front-end web development across the four
major component frameworks — React, Vue, Svelte, Angular — plus the TypeScript
language, modern CSS/styling, the build tooling that bundles and ships them, and
the quality disciplines (testing, accessibility, performance) every UI must pass.
The emphasis is on **version specificity**: every framework feature carries a
version marker and a fallback path, so guidance stays correct whether you target
React 19 or 18, Vue 3.5 or 3.4, Svelte 5 runes or Svelte 4, Angular 20+ signals
or NgModule-era code, ES2024 or ES2022. When a framework or browser-support claim
matters, verify it against the project's `package.json` and the canonical
[`_shared/version-feature-matrix.md`](_shared/version-feature-matrix.md) rather
than trusting memory.

## Quick Navigation

| Domain | Entry (canonical) | Skills | Focus |
|--------|-------------------|--------|-------|
| [TypeScript](#typescript) | [`typescript/typescript-skills/SKILL.md`](typescript/typescript-skills/SKILL.md) | 1 + 3 leaves | TS 5.x features, strict typing, tsconfig discipline, ES2024 |
| [React](#react) | [`react/react-skills/SKILL.md`](react/react-skills/SKILL.md) | 1 + 3 leaves | React 19 RSC/Actions/`use`, hooks, state, render performance |
| [Vue](#vue) | [`vue/vue-skills/SKILL.md`](vue/vue-skills/SKILL.md) | 1 + 2 leaves | Vue 3.5 Composition API, `<script setup>`, Pinia state |
| [Svelte](#svelte) | [`svelte/svelte-skills/SKILL.md`](svelte/svelte-skills/SKILL.md) | 1 + 2 leaves | Svelte 5 runes, SvelteKit routing/load/actions |
| [Angular](#angular) | [`angular/angular-skills/SKILL.md`](angular/angular-skills/SKILL.md) | 1 + 2 leaves | Angular 20+ signals, standalone, control flow, RxJS interop |
| [Styling](#styling) | [`styling/styling-skills/SKILL.md`](styling/styling-skills/SKILL.md) | 1 + 3 leaves | Modern CSS, Tailwind design systems, responsive + accessible CSS |
| [Tooling](#tooling) | [`tooling/tooling-skills/SKILL.md`](tooling/tooling-skills/SKILL.md) | 1 + 3 leaves | Vite/build systems, diagnostics, bundle optimization |
| [Quality](#quality) | [`quality/quality-skills/SKILL.md`](quality/quality-skills/SKILL.md) | 1 + 3 leaves | Testing, accessibility patterns, Core Web Vitals |
| [Shared](#shared) | [`_shared/_index.md`](_shared/_index.md) | 8 + references | Workflow integration, secure coding, versions, routing, severity |

**Total: 31 SKILL.md across 8 domains, plus shared references.**

## I need help with...

| Task | Go to |
|------|-------|
| Adopting a TS 5.x feature (`using`, `const` type params, `satisfies`) | [typescript/modern-typescript/SKILL.md](typescript/modern-typescript/SKILL.md) |
| Eliminating `any`, writing generics, discriminated unions | [typescript/ts-typing/SKILL.md](typescript/ts-typing/SKILL.md) |
| Turning on strict `tsconfig.json` flags | [typescript/ts-config/SKILL.md](typescript/ts-config/SKILL.md) |
| Using React 19 RSC, Server Actions, the `use` hook, React Compiler | [react/modern-react/SKILL.md](react/modern-react/SKILL.md) |
| Choosing Zustand vs Redux Toolkit vs TanStack Query | [react/react-state/SKILL.md](react/react-state/SKILL.md) |
| Fixing wasted React renders, memo, Suspense, virtualization | [react/react-performance/SKILL.md](react/react-performance/SKILL.md) |
| Writing Vue 3 `<script setup>` + Composition API | [vue/vue-composition/SKILL.md](vue/vue-composition/SKILL.md) |
| Structuring Pinia stores | [vue/vue-state/SKILL.md](vue/vue-state/SKILL.md) |
| Migrating to or writing Svelte 5 runes | [svelte/svelte-runes/SKILL.md](svelte/svelte-runes/SKILL.md) |
| Building SvelteKit routes, `load`, form actions | [svelte/sveltekit/SKILL.md](svelte/sveltekit/SKILL.md) |
| Adopting Angular 20+ signals + standalone components | [angular/angular-signals/SKILL.md](angular/angular-signals/SKILL.md) |
| Bridging RxJS and signals | [angular/angular-rxjs/SKILL.md](angular/angular-rxjs/SKILL.md) |
| Using container queries, `:has()`, cascade layers, subgrid | [styling/modern-css/SKILL.md](styling/modern-css/SKILL.md) |
| Building a Tailwind design-token system | [styling/tailwind-design-system/SKILL.md](styling/tailwind-design-system/SKILL.md) |
| Writing responsive + accessible CSS | [styling/responsive-accessible-css/SKILL.md](styling/responsive-accessible-css/SKILL.md) |
| Configuring Vite, choosing a bundler | [tooling/build-systems/SKILL.md](tooling/build-systems/SKILL.md) |
| Routing a build/runtime symptom to a fix | [tooling/fe-diagnostics/SKILL.md](tooling/fe-diagnostics/SKILL.md) |
| Shrinking a bundle, code-splitting, tree-shaking | [tooling/bundling-optimization/SKILL.md](tooling/bundling-optimization/SKILL.md) |
| Writing Vitest / Playwright / Testing Library tests | [quality/fe-testing/SKILL.md](quality/fe-testing/SKILL.md) |
| Auditing WCAG 2.2 / ARIA / keyboard / focus | [quality/accessibility-patterns/SKILL.md](quality/accessibility-patterns/SKILL.md) |
| Hitting Core Web Vitals budgets (LCP/INP/CLS) | [quality/web-performance/SKILL.md](quality/web-performance/SKILL.md) |
| Reviewing XSS sinks, CSP, secrets-in-bundle | [_shared/secure-coding/SKILL.md](_shared/secure-coding/SKILL.md) |
| Participating in an company-workflow workflow stage | [_shared/workflow-integration/SKILL.md](_shared/workflow-integration/SKILL.md) |
| Confirming a feature is available on a framework version | [_shared/version-feature-matrix.md](_shared/version-feature-matrix.md) |
| Routing a file or repo to the right agent | [_shared/language-detection.md](_shared/language-detection.md) |

Each domain's leaf skills are enumerated in its canonical `*-skills/SKILL.md`
(linked above) and in the full nav tree [`_index.md`](_index.md) — this router
holds only the task→leaf map and quick-nav above so it stays a router, not a
duplicate of the canonical selection tables it points to.

---

## Cross-Framework Version Snapshot

Current targets at a glance — the canonical lookup with minimum versions and
fallback rows is [`_shared/version-feature-matrix.md`](_shared/version-feature-matrix.md).
Framework feature landings and browser support shift between minor releases;
verify against the project's `package.json` before relying on a feature.

| Framework / language | Current target |
|----------------------|----------------|
| React | 19 (+ Next.js 15 App Router) |
| Vue | 3.5 (+ Nuxt 3) |
| Svelte | 5 (+ SvelteKit) |
| Angular | 20+ |
| TypeScript | 6.x |
| ECMAScript | ES2024 (verify runtime) |
| CSS | Baseline 2023-24 |

Headline features per version live in the matrix — this router does not restate them.

## Conventions

- **Version markers everywhere.** Every version-specific claim names a framework or version and links to the matrix; uneven-support items are hedged with "verify against the project / caniuse" instead of asserting a hard floor.
- **Lint-clean is the bar.** `tsc --noEmit` clean, eslint/biome zero-error, `jsx-a11y` / framework-template a11y lint clean.
- **Accessibility-first.** Every interactive component meets the WCAG 2.2 floor in [`_shared/accessibility-baseline.md`](_shared/accessibility-baseline.md) before it ships.

## Related Documentation

- [`_shared/version-feature-matrix.md`](_shared/version-feature-matrix.md) — canonical version/feature lookup
- [`_shared/language-detection.md`](_shared/language-detection.md) — file/repo → agent routing
- [`_shared/workflow-integration/SKILL.md`](_shared/workflow-integration/SKILL.md) — company-workflow workflow integration
- [`_index.md`](_index.md) — full navigation index of every skill directory
