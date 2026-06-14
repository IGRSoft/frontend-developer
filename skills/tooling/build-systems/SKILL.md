---
name: build-systems
description: >-
  Choosing and configuring front-end build systems — Vite (esbuild dev + Rollup
  build), Webpack, Turbopack, and esbuild — with version gates and migration
  notes. Use when starting a project's build, picking a bundler, configuring
  dev/prod modes and env handling, or migrating from Webpack to Vite.
---

# Build Systems

**Bundler selection and configuration.** For the domain selection table, see the
canonical [tooling-skills/SKILL.md](../tooling-skills/SKILL.md) — this leaf carries
depth and does not duplicate it.

## When to Use

Use this skill when:
- Choosing a bundler for a new app or library
- Configuring dev server, production build, aliases, env, or asset handling
- Migrating from Webpack to Vite, or onto Turbopack in Next.js
- Deciding which build-tool version a feature needs

## Choosing a bundler

| Choose | When |
|--------|------|
| **Vite** | New apps and libraries (React/Vue/Svelte/Solid). esbuild dev (no bundling = fast HMR) + Rollup production build. Default recommendation. |
| **Turbopack** | Next.js dev (`next dev --turbopack`). Incremental Rust bundler; production build still uses Next's pipeline. |
| **Webpack** | Legacy/enterprise codebases, complex Module Federation, or plugins with no Vite/Rollup equivalent. Mature but slower. |
| **esbuild** (direct) | Build scripts, simple bundling, transpile-only steps. Powers Vite dev under the hood. |
| **Rollup** (direct) | Publishing a library (clean ESM/CJS output, smallest bundles). Vite uses it for production. |

> Requires the Vite Environment API (Vite 6+; verify — still RC in Vite 8) for multi-environment dev/build. Fallback: Vite 5 single-environment dev server. Canonical: _shared/version-feature-matrix.md
> Requires Turbopack (Next.js dev stable since 15; default bundler for dev+build in Next.js 16). Fallback: webpack dev server (`next dev` without `--turbopack`). Canonical: _shared/version-feature-matrix.md

## Vite config essentials

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': '/src' } },
  build: {
    target: 'es2022',            // match the version-feature-matrix / browserslist
    sourcemap: true,             // for diagnosis; decide separately whether to ship
    rollupOptions: {
      output: { manualChunks: { vendor: ['react', 'react-dom'] } },  // see bundling-optimization
    },
  },
});
```

- **Dev** uses esbuild + native ESM — no bundling, so HMR is near-instant.
- **Build** uses Rollup — that is where code-splitting and tree-shaking happen.
- **Targets** should match your `browserslist`; do not down-level further than needed.

## Env handling: build-time vs runtime (a common bug)

```ts
// Vite: only VITE_-prefixed vars are exposed to client code, inlined at BUILD time
const api = import.meta.env.VITE_API_URL;   // baked into the bundle — NOT read at runtime
```

- **Client env is inlined at build time** — changing it requires a rebuild, not a
  restart. Do not expect to swap it per-deploy without rebuilding (or use a runtime
  config file fetched at boot).
- **Never put secrets in client-exposed env** (`VITE_*`, `NEXT_PUBLIC_*`) — they
  ship in the bundle. See [secure-coding](${CLAUDE_SKILL_DIR}/_shared/secure-coding/SKILL.md).
- Webpack's equivalent is `DefinePlugin` / `process.env.*` inlining — same caveat.

## Webpack → Vite migration notes

| Webpack concept | Vite equivalent |
|-----------------|-----------------|
| `webpack.config.js` loaders | built-in (TS/JSX/CSS) + plugins |
| `DefinePlugin` env inlining | `import.meta.env` + `define` |
| `resolve.alias` | `resolve.alias` |
| `splitChunks` | `build.rollupOptions.output.manualChunks` |
| `devServer.proxy` | `server.proxy` |
| CommonJS-heavy deps | may need `optimizeDeps.include` / `@rollup/plugin-commonjs` |

Migrate incrementally; the biggest wins are dev-server speed and simpler config.
Watch for CJS-only dependencies and dynamic `require()` that Rollup cannot statically analyze.

## Build diagnostics quick links

- Build fails / stack trace in bundled output → [fe-diagnostics](../fe-diagnostics/SKILL.md) (source maps)
- Bundle too big after a successful build → [bundling-optimization](../bundling-optimization/SKILL.md)
- Run builds as single scoped commands (`npx vite build`), never `&&`-chained.

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| Secret in `VITE_*`/`NEXT_PUBLIC_*` env | server-only env; never client-exposed |
| Expecting client env to change without rebuild | rebuild, or fetch a runtime config at boot |
| `npm install` in CI (non-reproducible) | `npm ci` against the lockfile |
| Over-broad `manualChunks` defeating caching | split by stable vendor boundaries; measure |
| Floating tool versions in prose | pin in the lockfile |

## Related Skills

- [tooling-skills/SKILL.md](../tooling-skills/SKILL.md) — canonical domain selection table
- [bundling-optimization/SKILL.md](../bundling-optimization/SKILL.md) — chunking and tree-shaking config lives here
- [fe-diagnostics/SKILL.md](../fe-diagnostics/SKILL.md) — when the build or its output breaks
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — canonical build-tool minimums
