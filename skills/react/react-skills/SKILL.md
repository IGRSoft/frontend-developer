---
name: react-skills
description: >-
  React language and framework skills navigation for React 19 + Next.js 15:
  modern features (RSC, Server Actions, the `use` hook, React Compiler),
  client-state architecture (Redux Toolkit, Zustand, TanStack Query), and
  render performance (memoization, Suspense, virtualization). Use when writing
  or reviewing React/Next code, adopting a React 19 feature, choosing a state
  library, or fixing wasted renders. Canonical selection table for the react
  domain.
---

# React Skills

**Navigation and the canonical selection table for React 19 + Next.js 15**

This is the **canonical selection table** for the react domain. The three leaves
below carry topic depth and link back here — they never restate this table.

## Version Snapshot

| Version | Headline (one line) |
|---------|---------------------|
| 18 | Concurrent rendering, `useTransition`/`useDeferredValue`, Suspense for data, automatic batching |
| 19 | RSC, Server Actions (`"use server"`), `use(promise/context)`, `useActionState`/`useFormStatus`/`useOptimistic`, `ref` as prop, document metadata |
| 19-era | React Compiler (opt-in auto-memoization) replaces most manual `useMemo`/`useCallback` |

Framework feature landings shift between minor releases — verify against the
project's `react`/`next` versions and the canonical
[version-feature-matrix](../../_shared/version-feature-matrix.md).

> Requires React Server Components and Server Actions (React 19 + Next.js 15 App Router). Fallback: client components with route loaders / API routes on React 18 + Next 14. Canonical: _shared/version-feature-matrix.md

**Stack in one line:** React 19 + a framework (Next.js 15 App Router / Remix) —
server state via **TanStack Query** or RSC fetch, client state via **Zustand**
or **Redux Toolkit** only when truly shared, forms via Server Actions +
`useActionState`. TypeScript-first throughout ([ts-typing](../../typescript/ts-typing/SKILL.md)).

## Skill Selection Guide

| I need to... | Use this skill |
|--------------|----------------|
| Use RSC, Server Actions, `use`, React Compiler, or fix hook-rule violations | [modern-react/SKILL.md](../modern-react/SKILL.md) |
| Choose between Redux Toolkit, Zustand, and TanStack Query | [react-state/SKILL.md](../react-state/SKILL.md) |
| Separate server cache from client UI state | [react-state/SKILL.md](../react-state/SKILL.md) |
| Diagnose and fix wasted renders, add `memo`/Suspense/virtualization | [react-performance/SKILL.md](../react-performance/SKILL.md) |
| Confirm which React/Next version a feature needs | [version-feature-matrix](../../_shared/version-feature-matrix.md) (canonical) |

## Decision Tree

```
React task?
├── Which version has feature X? → version-feature-matrix (canonical)
├── Writing modern React 19 / Next 15 → modern-react/SKILL.md
│   ├── Server Component vs Client Component → modern-react (RSC boundary)
│   ├── form mutation → Server Action + useActionState → modern-react
│   └── hook-rule / dependency-array bug → modern-react (Rules of Hooks)
├── Where does this state live? → react-state/SKILL.md
│   ├── server data (cache, refetch, mutate) → TanStack Query / RSC
│   ├── shared global UI state → Zustand (light) / RTK (large, normalized)
│   └── purely local → useState / useReducer (no library)
└── Component re-renders too much / list janks → react-performance/SKILL.md
```

## Non-Negotiables (apply to every React task)

- **Rules of Hooks.** Hooks at the top level only, never in conditions/loops; the `eslint-plugin-react-hooks` rule (`exhaustive-deps`) must be clean. The base Constraints treat eslint zero-error as a completion gate.
- **Server/Client boundary is explicit.** A file is a Server Component by default in the App Router; `"use client"` opts into the client. Don't pass non-serializable props across the boundary.
- **Keys are stable and unique** — never the array index for reorderable lists.
- **Accessibility is mandatory** — `jsx-a11y` lint clean; see [accessibility-patterns](../../quality/accessibility-patterns/SKILL.md).

## File Overview

| File | Purpose |
|------|---------|
| [modern-react/SKILL.md](../modern-react/SKILL.md) | React 19 RSC/Actions/`use`/Compiler, hook rules with version gates |
| [react-state/SKILL.md](../react-state/SKILL.md) | Server vs client state; RTK / Zustand / TanStack Query selection |
| [react-performance/SKILL.md](../react-performance/SKILL.md) | Wasted-render diagnosis, memoization, Suspense, virtualization |

## Related Skills

- [typescript-skills](../../typescript/typescript-skills/SKILL.md) — React is TypeScript-first; props/state typing
- [modern-css](../../styling/modern-css/SKILL.md) / [tailwind-design-system](../../styling/tailwind-design-system/SKILL.md) — styling React components
- [fe-testing](../../quality/fe-testing/SKILL.md) — Testing Library + Vitest/Playwright for React
- [web-performance](../../quality/web-performance/SKILL.md) — Core Web Vitals beyond render cost
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical React / Next version minimums
