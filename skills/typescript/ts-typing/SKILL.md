---
name: ts-typing
description: >-
  Strict TypeScript typing patterns: eliminating `any`, writing generics and
  constraints, modeling discriminated unions, narrowing `unknown` at trust
  boundaries, exhaustiveness checks, and choosing the right utility type. Use
  when removing `any`, designing a typed API, narrowing untrusted data, making
  a union exhaustive, or reviewing type safety.
---

# TypeScript Typing

**Strict typing patterns for safe, expressive types.** The canonical selection
table lives in [typescript-skills/SKILL.md](../typescript-skills/SKILL.md); this
leaf carries depth only.

## When to Use

- Replacing an `any` with a precise type, `unknown`, or a generic
- Modeling state/results as a discriminated union
- Narrowing `unknown` data coming from `fetch`, `JSON.parse`, or message events
- Guaranteeing a `switch`/`match` covers every variant
- Picking a utility type instead of hand-rolling one

## No `any` — the trust-boundary rule

`any` disables checking and spreads silently. The base Constraints forbid `any`
without a justifying comment. At every trust boundary (network, `JSON.parse`,
`postMessage`, third-party callbacks) start from `unknown` and **narrow**.

```ts
async function loadUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  const data: unknown = await res.json();      // never `any`
  return parseUser(data);                       // validate before trusting
}

function parseUser(d: unknown): User {
  if (
    typeof d === "object" && d !== null &&
    "id" in d && typeof (d as Record<string, unknown>).id === "string"
  ) {
    return d as User;
  }
  throw new TypeError("invalid User payload");
}
```

For non-trivial shapes prefer a schema validator (Zod/Valibot/ArkType) that
*derives* the type — one source of truth, runtime + compile-time.

## Discriminated unions over boolean flags

Model mutually exclusive states as a tagged union so impossible states are
unrepresentable.

```ts
type Result<T> =
  | { status: "loading" }
  | { status: "error"; error: Error }
  | { status: "ok"; data: T };

function render(r: Result<User>) {
  switch (r.status) {
    case "loading": return spinner();
    case "error":   return errorBox(r.error);     // r.error exists only here
    case "ok":      return profile(r.data);
    default:        return assertNever(r);          // exhaustiveness gate
  }
}

function assertNever(x: never): never { throw new Error(`unhandled: ${JSON.stringify(x)}`); }
```

The `assertNever` arm turns a missing case into a **compile error** when a new
variant is added — far stronger than a runtime `default`.

## Generics with constraints

```ts
function pluck<T, K extends keyof T>(items: readonly T[], key: K): T[K][] {
  return items.map((it) => it[key]);
}
const names = pluck(users, "name"); // string[]
```

Rules: constrain type parameters (`K extends keyof T`) so the body is safe;
prefer inference over explicit type arguments at the call site; avoid generics
that appear in only one position (they add noise without safety).

## Narrowing toolkit

| Technique | Use for |
|-----------|---------|
| `typeof` / `instanceof` | primitives and class instances |
| `in` operator | discriminating object shapes by key presence |
| user-defined predicate `x is T` | reusable structural checks (5.5 can infer these) |
| `Array.isArray` | array vs object |
| `assertNever(x: never)` | exhaustiveness at the end of a closed union |

> Requires inferred type predicates (TypeScript 5.5+). Fallback: write the `x is T` predicate by hand on TS ≤ 5.4. Canonical: _shared/version-feature-matrix.md

## Utility types — reach for the built-in first

| Goal | Utility |
|------|---------|
| make all/some fields optional | `Partial<T>`, `Pick<T, K> & Partial<...>` |
| make fields required/readonly | `Required<T>`, `Readonly<T>` |
| derive a subset / omit keys | `Pick<T, K>`, `Omit<T, K>` |
| key-driven map | `Record<K, V>` |
| function/instance shapes | `ReturnType<F>`, `Parameters<F>`, `InstanceType<C>`, `Awaited<P>` |
| template-literal key derivation | template literal types (`` `on${Capitalize<E>}` ``) |

Avoid deep conditional-type gymnastics in app code — if a type needs a comment
to explain its mechanics, prefer an explicit interface.

## Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| `any` at a network/parse boundary | `unknown` + a validator (Zod/Valibot) |
| `as Foo` to silence an error | narrow with a guard, or fix the upstream type |
| `// @ts-ignore` | `// @ts-expect-error` with a reason, then delete when fixed |
| boolean-flag state (`isLoading`, `isError` both `false`) | discriminated union |
| `enum` for a closed string set | union of string literals + `as const` object |
| `Function`/`object`/`{}` parameter types | precise signature or `unknown` |

## Related Skills

- [modern-typescript](../modern-typescript/SKILL.md) — `satisfies`, `const` type params, `NoInfer` that sharpen these patterns
- [ts-config](../ts-config/SKILL.md) — `noUncheckedIndexedAccess` / `exactOptionalPropertyTypes` strictness that makes narrowing matter
- [secure-coding](../../_shared/secure-coding/SKILL.md) — typed validation at untrusted input boundaries
- [typescript-skills](../typescript-skills/SKILL.md) — canonical selection table (start here)
