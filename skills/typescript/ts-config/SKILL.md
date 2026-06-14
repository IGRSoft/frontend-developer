---
name: ts-config
description: >-
  TypeScript compiler configuration discipline: the strict-mode flag set,
  `module`/`moduleResolution` choices for bundler vs Node, project references
  for monorepos, and the type-check-vs-emit boundary (tsc as gate, bundler as
  emitter). Use when setting up `tsconfig.json`, turning on strict flags,
  fixing module-resolution errors, or splitting a monorepo into project refs.
---

# TypeScript Config (tsconfig.json)

**Strict, bundler-aware compiler configuration.** The canonical selection table
lives in [typescript-skills/SKILL.md](../typescript-skills/SKILL.md); this leaf
carries depth only.

## When to Use

- Standing up a new `tsconfig.json` for a Vite/Next/framework project
- Turning on strict flags incrementally on a legacy codebase
- Debugging `Cannot find module` / wrong `moduleResolution`
- Setting up project references in a monorepo
- Deciding whether `tsc` should emit at all (usually: no)

## Type-check, do not bundle

In a modern front-end project the bundler (Vite/esbuild/SWC/Next) transpiles;
`tsc` is the **type gate** only.

```jsonc
{
  "compilerOptions": {
    "noEmit": true,            // bundler emits JS; tsc only checks
    "skipLibCheck": true,      // skip type-checking .d.ts of deps (speed)
    "isolatedModules": true    // each file transpiles alone (matches esbuild/SWC)
  }
}
```

Run `tsc --noEmit` in CI and as the base Constraints gate (must be 0 errors).
Never ship `tsc` output from a bundled app.

## The strict baseline (non-negotiable floor)

`strict: true` enables the whole family at once; the rows below are the ones
worth knowing by name. New code should add `noUncheckedIndexedAccess`.

```jsonc
{
  "compilerOptions": {
    "strict": true,                       // umbrella: the 8 strict sub-flags
    "noUncheckedIndexedAccess": true,     // arr[i] is T | undefined — catches OOB access
    "noImplicitOverride": true,           // require `override` keyword
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,   // `?:` ≠ `| undefined` (stricter, opt-in)
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true          // explicit `import type`; predictable elision
  }
}
```

`strict` covers `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`,
`strictBindCallApply`, `strictPropertyInitialization`,
`useUnknownInCatchVariables`, `alwaysStrict`, `noImplicitThis`. Turning
`strict` off to "fix" errors is forbidden — fix the types instead.

## module / moduleResolution: pick by environment

| Environment | `module` | `moduleResolution` | Notes |
|-------------|----------|--------------------|-------|
| Vite / esbuild / app bundled | `esnext` | `bundler` | the bundler owns resolution; `bundler` allows extensionless imports |
| Library shipped as ESM | `esnext` / `node16` | `node16` / `nodenext` | honors `package.json` `exports`, requires explicit extensions |
| Node CJS/ESM dual | `nodenext` | `nodenext` | strictest; `.js`/`.mjs`/`.cjs` semantics enforced |

> Requires `moduleResolution: "bundler"` and `module: "preserve"` (TypeScript 5.0 / 5.4+). Fallback: `moduleResolution: "node16"` with explicit extensions. Canonical: _shared/version-feature-matrix.md

**Rule:** in a bundled app use `"bundler"`. For a published package use
`node16`/`nodenext` so consumers' resolvers match.

## target / lib

- `target`: set to what your runtime supports (`es2022`/`esnext` for evergreen
  browsers; the bundler may down-level further). Don't down-target to `es5`
  unless a legacy browser matrix requires it.
- `lib`: include the DOM and the ECMAScript libs your `target` implies. For
  `using`, add `esnext.disposable`. For ES2024 APIs, ensure `lib` includes them
  (availability is still runtime-gated — see version matrix).

## Project references (monorepos)

Split a workspace into independently type-checked projects so `tsc --build`
caches and only rechecks what changed.

```jsonc
// tsconfig.json (solution)
{ "files": [], "references": [{ "path": "./packages/core" }, { "path": "./packages/ui" }] }

// packages/ui/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "composite": true, "outDir": "dist", "rootDir": "src" },
  "references": [{ "path": "../core" }]
}
```

`composite: true` is required on referenced projects; build with
`tsc --build` (single `Bash(npx:*)` invocation). Share strictness via a
`tsconfig.base.json` and `extends` — never duplicate flags per package.

> Requires `${configDir}` token in shared base configs (TypeScript 5.5+). Fallback: relative `extends` paths per package on TS ≤ 5.4. Canonical: _shared/version-feature-matrix.md

## Common errors → fix

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot find module 'x' or its type declarations` | wrong `moduleResolution`, or missing `@types/x` | switch to `bundler`/`node16`; install types |
| `Relative import paths need explicit file extensions` | `node16`/`nodenext` resolution | add `.js` extension to relative imports (yes, `.js` even from `.ts`) |
| `Object is possibly 'undefined'` after indexing | `noUncheckedIndexedAccess` on | guard the access or assert non-null with a reason |
| `Cannot use import statement outside a module` at runtime | `module`/`type` mismatch with the runtime | align `package.json` `"type"` and `module` setting |
| slow `tsc` in a monorepo | no project references / no `composite` | add references + `tsc --build` incremental cache |

## Related Skills

- [modern-typescript](../modern-typescript/SKILL.md) — feature flags (`lib` for `using`, decorators config)
- [ts-typing](../ts-typing/SKILL.md) — the strictness flags above are what make narrowing meaningful
- [build-systems](../../tooling/build-systems/SKILL.md) — how the bundler consumes/ignores these settings
- [typescript-skills](../typescript-skills/SKILL.md) — canonical selection table (start here)
