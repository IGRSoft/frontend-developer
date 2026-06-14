---
name: modern-typescript
description: >-
  Modern TypeScript language features for TS 5.x with explicit version gates:
  `using`/`await using` resource management (5.2), `const` type parameters and
  ES standard decorators (5.0), `satisfies` (4.9), `NoInfer<T>` (5.4), and
  inferred type predicates (5.5). Use when adopting a TS 5.x feature, checking
  which version a feature needs, finding a pre-version fallback, or reviewing
  code for legacy idioms newer syntax replaces.
---

# Modern TypeScript (5.x)

**Version-gated feature adoption with explicit fallbacks.** This leaf carries
topic depth only; the canonical selection table lives in
[typescript-skills/SKILL.md](../typescript-skills/SKILL.md).

## When to Use

- Adopting a new language feature and you need its minimum version + fallback
- Choosing between `satisfies` and an explicit annotation
- Adding deterministic cleanup with `using`/`await using`
- Reviewing code for legacy idioms that newer syntax replaces
- Deciding between standard decorators and `experimentalDecorators`

## Per-Feature Version Gate

Pick the lowest TS version that has the feature; if `typescript` is pinned
lower, use the fallback column. Canonical minimums live in the
[version-feature-matrix](../../_shared/version-feature-matrix.md).

| Feature | Min version | Pre-version fallback |
|---------|-------------|----------------------|
| `satisfies` operator | TS 4.9+ | explicit annotation + careful widening |
| `const` type parameters (`<const T>`) | TS 5.0+ | `as const` at call sites |
| ES standard decorators | TS 5.0+ | `experimentalDecorators: true` (legacy semantics) |
| `using` / `await using` (explicit resource management) | TS 5.2+ | manual `try/finally` cleanup |
| `NoInfer<T>` | TS 5.4+ | hand-rolled inference-blocking wrapper type |
| inferred type predicates | TS 5.5+ | hand-written `x is T` predicate function |
| `${configDir}` in tsconfig / `module: "preserve"` | TS 5.5+ *(verify)* | relative paths; `module: "esnext"` |

> Requires `using` explicit resource management (TypeScript 5.2+). Fallback: manual `try/finally` cleanup on TS 4.9–5.1. Canonical: _shared/version-feature-matrix.md

## `satisfies`: validate without widening (4.9+)

`satisfies` checks a value against a type **without** changing the inferred type,
so you keep literal narrowness *and* get constraint checking.

```ts
type Route = { path: string; method: "GET" | "POST" };

// `as const` loses the Route check; an annotation widens `method` to string.
const routes = {
  home:  { path: "/",      method: "GET" },
  login: { path: "/login", method: "POST" },
} satisfies Record<string, Route>;

routes.home.method; // narrowed to "GET", and the shape is verified
```

**Rule:** reach for `satisfies` when you want both the literal type *and* a
guarantee the value conforms — config objects, route tables, theme maps.

## `using`: deterministic cleanup (5.2+)

A `using` declaration disposes the resource at scope exit (calls
`[Symbol.dispose]`); `await using` awaits `[Symbol.asyncDispose]`.

```ts
function openHandle(): Disposable & { read(): string } {
  const id = acquire();
  return { read: () => peek(id), [Symbol.dispose]: () => release(id) };
}

function work(): void {
  using h = openHandle();   // released automatically at end of scope, even on throw
  process(h.read());
}                            // no try/finally needed
```

Pre-5.2 fallback: `try { ... } finally { release() }`. Useful for DB handles,
file/stream wrappers, observers, AbortController-backed subscriptions.

## `const` type parameters and `NoInfer` (5.0 / 5.4)

```ts
// const T: infer the narrowest literal type from the argument, no `as const` at the call site.
function tuple<const T extends readonly unknown[]>(...xs: T): T { return xs; }
const t = tuple("a", 1);          // type: readonly ["a", 1]

// NoInfer: block one parameter from driving inference.
function createState<T>(initial: T, fallback: NoInfer<T>): T { return initial ?? fallback; }
```

`const` type params remove the call-site `as const`; `NoInfer<T>` (5.4) stops a
secondary argument from widening `T`. Pre-version: `as const` and a manually
named `TypeVar`-style wrapper.

## Standard decorators vs experimental (5.0)

TS 5.0 ships the **ES standard** decorator semantics (no `experimentalDecorators`
flag, no `emitDecoratorMetadata`). Frameworks that rely on the old metadata
reflection (Angular pre-16 DI, some NestJS/TypeORM setups) still need
`experimentalDecorators: true`.

> Requires ES standard decorators (TypeScript 5.0+). Fallback: `experimentalDecorators: true` for metadata-reflection frameworks. Canonical: _shared/version-feature-matrix.md

**Rule:** use standard decorators for new code; keep `experimentalDecorators`
only where a framework demands legacy metadata — never mix the two in one project.

## Legacy idioms newer syntax replaces

| Legacy | Modern (TS version) | Why |
|--------|---------------------|-----|
| `as const` for a checked config | `satisfies T` (4.9) | keeps literals *and* validates the shape |
| `try { } finally { cleanup() }` for scoped resources | `using` (5.2) | cleanup is declarative and throw-safe |
| `enum` for a closed string set | union of string literals + `as const` object | smaller output, no reverse-mapping footguns |
| `namespace` for module grouping | ES modules (`import`/`export`) | tree-shakeable, standard |
| `Function`/`object`/`{}` loose types | `unknown` + narrowing, precise signatures | `{}` and `Function` are unsafe escape hatches |

## ES2024 runtime surface (compile target awareness)

TS types track the ECMAScript library; runtime availability is the limiter.
`Object.groupBy`, `Promise.withResolvers()`, and `Array.fromAsync()` are ES2024.

> Requires `Object.groupBy` / `Promise.withResolvers` (ES2024; verify runtime + `lib`/`target`). Fallback: manual `Map` grouping and an explicit promise executor. Canonical: _shared/version-feature-matrix.md

## Diagnostics

| Symptom | Cause | Fix |
|---------|-------|-----|
| `using` flagged as a syntax error | TS < 5.2 or target lib missing `Symbol.dispose` | upgrade TS, add `lib: ["esnext.disposable"]`, or use `try/finally` |
| Decorator metadata `undefined` at runtime | switched to standard decorators but framework expects legacy reflection | set `experimentalDecorators: true` + `emitDecoratorMetadata` for that project |
| `satisfies` value still widened | annotation also present, or `as const` removed by mistake | drop the annotation; let `satisfies` do the check |
| `Object.groupBy` is not a function | runtime predates ES2024 | polyfill or reduce into a `Map` manually |

## Related Skills

- [ts-typing](../ts-typing/SKILL.md) — generics, unions, narrowing that these features build on
- [ts-config](../ts-config/SKILL.md) — `lib`/`target`/`module` settings these features require
- [typescript-skills](../typescript-skills/SKILL.md) — canonical selection table (start here)
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical TS / ES version minimums
