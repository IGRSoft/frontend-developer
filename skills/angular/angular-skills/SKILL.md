---
name: angular-skills
description: >-
  Angular skills navigation for Angular 18+ — signals reactivity, standalone
  components and the new control flow, RxJS interop, and the modern functional
  provider/router APIs. Use when writing or reviewing Angular, choosing between
  signals and RxJS, migrating off NgModules, or deciding which Angular feature
  needs which version.
---

# Angular Skills

**Canonical selection table and version snapshot for Angular 18+ development.**
This is *the* selection table for the `angular/` domain — every leaf below links
back here and does not duplicate it.

## Version Snapshot

| Version | Headline (one line) |
|---------|---------------------|
| 16 | Signals (developer preview), standalone APIs stabilizing, `DestroyRef`, `takeUntilDestroyed` |
| 17 | New built-in control flow (`@if`/`@for`/`@switch`), deferred loading (`@defer`), standalone the default for new apps, renamed build/serve to esbuild/Vite |
| 17.1–18 | Signal `input()`/`output()`/`model()`, `signal()`-based queries, signals stabilizing |
| 18 | Zoneless change detection (experimental), built-in control flow stable, material 3 stable |

Feature landings shift across point releases; for anything you pin in CI, verify
against the project's `package.json` and the Angular release notes, and link the
canonical [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md).

> Requires Angular 18+ signals and standalone components. Fallback: Angular 15 RxJS `BehaviorSubject` + `async` pipe and NgModule declarations. Canonical: _shared/version-feature-matrix.md

**Stack in one line:** standalone components (no NgModule) + signals for synchronous
state + RxJS for async streams + `provideHttpClient()`/`provideRouter()` functional
providers + the new control flow. Pin the Angular version in the lockfile, never in prose.

## Skill Selection Guide

| I need to... | Use this skill |
|--------------|----------------|
| Use signals (`signal`/`computed`/`effect`), signal inputs/outputs, standalone components, new control flow | [angular-signals/SKILL.md](../angular-signals/SKILL.md) |
| Bridge RxJS and signals (`toSignal`/`toObservable`), manage subscriptions, choose streams vs signals | [angular-rxjs/SKILL.md](../angular-rxjs/SKILL.md) |
| Add type annotations / strict template typing | [modern-typescript](${CLAUDE_SKILL_DIR}/typescript/typescript-skills/SKILL.md) |
| Audit a component for WCAG / ARIA | [accessibility-patterns](${CLAUDE_SKILL_DIR}/quality/accessibility-patterns/SKILL.md) |
| Write component / e2e tests | [fe-testing](${CLAUDE_SKILL_DIR}/quality/fe-testing/SKILL.md) |
| Optimize change detection / bundle | [web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md) · [bundling-optimization](${CLAUDE_SKILL_DIR}/tooling/bundling-optimization/SKILL.md) |

## Decision Tree

```
Angular task?
├── Which version has feature X? → version-feature-matrix (canonical)
├── Synchronous component/UI state → angular-signals/SKILL.md
│   ├── signal() / computed() / effect()
│   ├── input()/output()/model() signal-based I/O
│   └── standalone component + @if/@for/@switch control flow
├── Async stream (HTTP, events, debounce, combine) → angular-rxjs/SKILL.md
│   ├── toSignal()/toObservable() interop
│   └── subscription lifecycle (takeUntilDestroyed)
├── Type/template strictness → typescript/ (modern-typescript)
├── a11y / testing / perf → quality/ domain
└── Migrating off NgModules → /frontend-developer:code-modernize
```

## Domain Constraints (Angular delta)

These augment the inherited `_base/frontend-agent.md` Constraints — they do not
restate or weaken them.

- **Signals for synchronous state; RxJS for asynchronous streams.** Do not wrap a
  plain value in a `BehaviorSubject` when a `signal` suffices, and do not poll a
  signal where an observable stream models the data.
- **Standalone-first.** New components/directives/pipes are `standalone: true`
  (the default from v17). Do not introduce a new `NgModule` in greenfield code.
  > Requires standalone components (Angular 15+, default 17+). Fallback: declare in an `NgModule`. Canonical: _shared/version-feature-matrix.md
- **`OnPush` (or zoneless) change detection** for new components; never mutate
  inputs in place — produce new references so change detection fires.
- **No manual `subscribe()` without teardown.** Prefer the `async` pipe or
  `toSignal()`; when you must subscribe imperatively, use `takeUntilDestroyed()`.
- **New control flow** (`@if`/`@for`/`@switch`) over the legacy `*ngIf`/`*ngFor`
  structural directives in v17+ code; `@for` requires a `track` expression.
  > Requires new control flow (Angular 17+). Fallback: `*ngIf`/`*ngFor`/`*ngSwitch`. Canonical: _shared/version-feature-matrix.md

## File Overview

| File | Purpose |
|------|---------|
| [angular-signals/SKILL.md](../angular-signals/SKILL.md) | Signals reactivity, signal I/O, standalone components, new control flow |
| [angular-rxjs/SKILL.md](../angular-rxjs/SKILL.md) | RxJS interop with signals, subscription lifecycle, stream operators |

## Related Skills

- [typescript-skills](${CLAUDE_SKILL_DIR}/typescript/typescript-skills/SKILL.md) — strict typing for templates and DI
- [fe-testing](${CLAUDE_SKILL_DIR}/quality/fe-testing/SKILL.md) — TestBed, component harnesses, Playwright
- [accessibility-patterns](${CLAUDE_SKILL_DIR}/quality/accessibility-patterns/SKILL.md) — CDK a11y, focus management
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — canonical Angular version minimums
- [secure-coding](${CLAUDE_SKILL_DIR}/_shared/secure-coding/SKILL.md) — `DomSanitizer`, `bypassSecurityTrust*` hazards
