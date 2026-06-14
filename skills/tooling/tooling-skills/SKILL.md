---
name: tooling-skills
description: >-
  Build, diagnostics, and bundling skills navigation for front-end tooling —
  Vite/Webpack/Turbopack/esbuild build systems, source-map and hydration-error
  diagnostics, and bundle optimization (code-split, tree-shake, lazy). Use when
  configuring a build, debugging a build/runtime/hydration error, reading source
  maps, or shrinking a bundle.
---

# Tooling Skills

**Canonical selection table for front-end build, diagnostics, and bundling.**
This is *the* selection table for the `tooling/` domain — every leaf below links
back here and does not duplicate it.

## Tool Snapshot

| Tool | Role | One line |
|------|------|----------|
| Vite | dev + build | esbuild dev (no bundle) + Rollup production build; default for new apps |
| Webpack | build | mature, plugin-rich; legacy/enterprise and complex module-federation setups |
| Turbopack | dev (Next) | Rust bundler, `next dev --turbopack`; incremental |
| esbuild | transpile/bundle | extremely fast Go bundler; powers Vite dev + many toolchains |
| Rollup | build | library-grade bundler underneath Vite's production build |

Pin tool versions in the lockfile, never in prose. Version-gated tool features
(Vite Environment API, Turbopack dev stability) link the canonical
[version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md).

> Requires the Vite Environment API (Vite 6+; verify — still RC in Vite 8). Fallback: Vite 5 single-environment dev/build. Canonical: _shared/version-feature-matrix.md

## Skill Selection Guide

| I need to... | Use this skill |
|--------------|----------------|
| Configure or choose a build system (Vite/Webpack/Turbopack/esbuild) | [build-systems/SKILL.md](../build-systems/SKILL.md) |
| Debug a build/runtime error, read source maps, fix a hydration mismatch | [fe-diagnostics/SKILL.md](../fe-diagnostics/SKILL.md) |
| Shrink the bundle: code-split, tree-shake, lazy-load, analyze | [bundling-optimization/SKILL.md](../bundling-optimization/SKILL.md) |
| Tie bundle size to a perf budget | [web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md) |
| Configure tests run by the toolchain | [fe-testing](${CLAUDE_SKILL_DIR}/quality/fe-testing/SKILL.md) |

## Decision Tree

```
Tooling task?
├── Set up / pick a bundler → build-systems/SKILL.md
│   ├── new app → Vite (esbuild dev + Rollup build)
│   ├── Next.js → Turbopack dev / built-in build
│   └── legacy / module federation → Webpack
├── Something is broken → fe-diagnostics/SKILL.md
│   ├── stack trace points at bundled code → source maps
│   ├── "hydration mismatch" / "text content did not match" → hydration errors
│   └── env/config not applied → build-time vs runtime env
├── Bundle too big / slow → bundling-optimization/SKILL.md
│   ├── route/feature splitting → dynamic import()
│   ├── dead code shipped → tree-shaking (sideEffects, ESM)
│   └── measure first → analyzer
└── Bundle vs perf budget → quality/web-performance
```

## Domain Constraints (tooling delta)

These augment the inherited `_base/frontend-agent.md` Constraints and Tool Priority
— they do not restate or weaken them.

- **Single-command scoped Bash only.** Run `npm run build`, `npx vite build`,
  `npx tsc --noEmit` as separate calls — never `&&`-chained (scoped `Bash(cmd:*)`
  cannot match compound lines). This is the inherited rule; tooling work hits it most.
- **Measure before optimizing.** Run an analyzer (`rollup-plugin-visualizer`,
  `webpack-bundle-analyzer`, `vite build --report`) before changing config; record
  the before/after delta in DV Build Evidence.
- **Source maps for diagnosis, not for production exposure.** Generate maps for
  debugging; decide deliberately whether to ship them (and never ship secrets via
  inline source content).
- **Never guess CLI flags.** Use `<tool> --help` / Context7 / Ref — inherited Tool
  Priority.
- **Reproducible builds:** the lockfile is authoritative; `npm ci` (not `npm install`)
  in CI; do not float versions.

## File Overview

| File | Purpose |
|------|---------|
| [build-systems/SKILL.md](../build-systems/SKILL.md) | Vite/Webpack/Turbopack/esbuild selection and config |
| [fe-diagnostics/SKILL.md](../fe-diagnostics/SKILL.md) | Source maps, hydration errors, build-vs-runtime debugging |
| [bundling-optimization/SKILL.md](../bundling-optimization/SKILL.md) | Code-splitting, tree-shaking, lazy-loading, analysis |

## Related Skills

- [web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md) — bundle size → Core Web Vitals budgets
- [fe-testing](${CLAUDE_SKILL_DIR}/quality/fe-testing/SKILL.md) — Vitest tracks the Vite toolchain
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — canonical build-tool minimums
- [secure-coding](${CLAUDE_SKILL_DIR}/_shared/secure-coding/SKILL.md) — secrets-in-bundle, dependency supply chain
