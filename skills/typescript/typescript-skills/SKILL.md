---
name: typescript-skills
description: >-
  TypeScript language skills navigation for TS 5.x: modern features (`using`,
  `const` type parameters, `satisfies`, standard decorators), strict static
  typing (no `any`, generics, discriminated unions, narrowing), and disciplined
  `tsconfig.json` configuration. Use when writing or reviewing TypeScript,
  adopting a TS 5.x feature, eliminating `any`, tightening type safety, or
  turning on strict compiler flags. Canonical selection table for the
  typescript domain.
---

# TypeScript Skills

**Navigation and the canonical selection table for TypeScript 5.x development**

This is the **canonical selection table** for the typescript domain. The three
leaves below carry topic depth and link back here — they never restate this
table.

## Version Snapshot

| Version | Headline (one line) |
|---------|---------------------|
| 4.9 | `satisfies` operator; `in` narrowing on `unknown` |
| 5.0 | `const` type parameters; ES standard decorators; `--moduleResolution bundler` |
| 5.2 | `using`/`await using` explicit resource management (Symbol.dispose) |
| 5.4 | `NoInfer<T>`; preserved narrowing in closures following last assignment |
| 5.5 | inferred type predicates; `${configDir}` in tsconfig; `--isolatedDeclarations` |

Compiler minutiae shift between minor releases — for anything you pin in CI,
verify against the project's `typescript` dependency and the canonical
[version-feature-matrix](../../_shared/version-feature-matrix.md).

> Requires TypeScript 5.2 `using` / 5.0 `const` type params. Fallback: `try/finally` cleanup and `as const` on TS 4.9. Canonical: _shared/version-feature-matrix.md

**Toolchain in one line:** `tsc --noEmit` (type gate, never emits) — `eslint`
with `@typescript-eslint` or `biome` (lint) — the bundler (Vite/esbuild/SWC)
owns transpilation, **not** `tsc`. Pin `typescript` in `package.json`, never in
prose.

## Skill Selection Guide

| I need to... | Use this skill |
|--------------|----------------|
| Adopt a TS 5.x feature (`using`, `const T`, `satisfies`, decorators) | [modern-typescript/SKILL.md](../modern-typescript/SKILL.md) |
| Eliminate `any`, write generics, model a discriminated union | [ts-typing/SKILL.md](../ts-typing/SKILL.md) |
| Narrow `unknown`, exhaustively check a union, pick a utility type | [ts-typing/SKILL.md](../ts-typing/SKILL.md) |
| Turn on strict flags, fix `moduleResolution`, set up project refs | [ts-config/SKILL.md](../ts-config/SKILL.md) |
| Confirm which TS version a feature needs | [version-feature-matrix](../../_shared/version-feature-matrix.md) (canonical) |

## Decision Tree

```
TypeScript task?
├── Which version has feature X? → version-feature-matrix (canonical)
├── Adopting a modern language feature → modern-typescript/SKILL.md
│   ├── resource cleanup (using/await using) → modern-typescript (TS 5.2+)
│   ├── const type params / NoInfer → modern-typescript (TS 5.0 / 5.4)
│   └── satisfies vs annotation → modern-typescript (TS 4.9+)
├── Type modeling / no-any / generics / narrowing → ts-typing/SKILL.md
├── Compiler config / strictness / module resolution → ts-config/SKILL.md
└── ES2024 runtime feature (Object.groupBy, withResolvers) → version-feature-matrix
```

## Non-Negotiables (apply to every TypeScript task)

- **`strict: true` is the floor.** All strict sub-flags on; `noUncheckedIndexedAccess` strongly recommended for new code. See [ts-config](../ts-config/SKILL.md).
- **No `any` without a justifying comment.** Prefer `unknown` + narrowing, generics, or `satisfies`. `tsc --noEmit` must be 0-error before completion (base Constraints).
- **`tsc` does not bundle.** Type-check with `tsc --noEmit`; let the bundler emit JS. Never ship `tsc`-emitted output in a Vite/Next project.
- **Public API surfaces carry TSDoc** (`/** */`, `@param`/`@returns`) per the base Code Comment Policy.

## File Overview

| File | Purpose |
|------|---------|
| [modern-typescript/SKILL.md](../modern-typescript/SKILL.md) | TS 5.x feature gates with fallbacks |
| [ts-typing/SKILL.md](../ts-typing/SKILL.md) | Strict typing patterns, generics, unions, narrowing |
| [ts-config/SKILL.md](../ts-config/SKILL.md) | tsconfig strictness, module resolution, project references |

## Related Skills

- [react-skills](../../react/react-skills/SKILL.md) / [vue-skills](../../vue/vue-skills/SKILL.md) / [svelte-skills](../../svelte/svelte-skills/SKILL.md) / [angular-skills](../../angular/angular-skills/SKILL.md) — framework code is TypeScript-first
- [secure-coding](../../_shared/secure-coding/SKILL.md) — typed boundaries for untrusted input, `unknown` over `any` at trust edges
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical TS / ES version minimums
