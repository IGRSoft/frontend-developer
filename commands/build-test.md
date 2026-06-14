---
description: Detect the package manager and framework, install dependencies, build, and run unit/component tests for a web project
argument-hint: [path (default .)] [--prod] [--no-install] [--no-test]
allowed-tools: Read, Glob, Grep, Bash
estimated-cost:
  band: medium
  min-tokens: 1500
  max-tokens: 12000
  model-distribution:
    haiku: 40%
    sonnet: 55%
    opus: 5%
---

# Build & Test
<!-- Updated: June 2026 -->

Detect a web project's package manager and framework, install dependencies, build it, and run its unit/component test suite with a single command. The happy path is pure shell — no agent routing. The framework specialist is engaged only when the build or tests fail, and only the framework that owns the failing layer is consulted, with the relevant log excerpt.

[Extended thinking: This command is the detection-and-execution workhorse other frontend-developer commands reuse. It resolves exactly one package manager per invocation via a strict lockfile-priority order, resolves the framework from `package.json`, runs the canonical install/build/test sequence, and tees everything to a timestamped log. Because install/build/test failures have sharply different fixes, on failure it parses the first error, classifies the stage, and routes the matching framework specialist a tight excerpt instead of the whole log. Keep the green path deterministic and shell-only so it stays cheap and scriptable.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Resolve exactly one package manager.** Walk the lockfile priority order top-down and stop at the first match. Do NOT run two package managers in one invocation. If the user disagrees, they re-run with an explicit path to the workspace package.
2. **Happy path is shell-only.** When install, build, and test all succeed, do NOT route to any agent. Report the result and stop.
3. **Single-command Bash invocations.** Use each tool's own flags (`npm ci`, `npm run build`, `npx vitest run`, `pnpm --dir <path> build`). Never `cd`-chain or `&&`-chain — scoped Bash patterns do not match compound commands.
4. **Tee every phase to the log.** Each install/build/test command pipes through `tee -a` to `.context/logs/build-<timestamp>.log`. The log is the single source of truth for triage; do not rely on terminal scrollback.
5. **On failure, classify before routing.** Parse the first error from the log, classify it as install / build / test, then route ONLY to the matching framework specialist with the excerpt — never the whole log, never a second agent "just in case."
6. **Tool-missing never hard-fails.** If the package manager binary is absent, print the install hint, fall back to the next eligible manager, and continue. Report what was skipped.
7. **Commands route, they do not orchestrate.** This command names which `frontend-developer:*` agent owns each failing layer so Claude routes the work; it does not call `Task` itself.
8. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Detect, install, build, and test the current directory
/frontend-developer:build-test .

# Build a specific workspace package
/frontend-developer:build-test packages/ui

# Production build, skip install (CI with a warm cache)
/frontend-developer:build-test . --prod --no-install

# Install and build only; skip tests
/frontend-developer:build-test . --no-test
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `path` | `.` | Directory to detect and operate on. The detection scan is rooted here. |
| `--prod` | off | Production build (`NODE_ENV=production`); use the framework's production build script and skip dev-only steps. |
| `--no-install` | off | Skip the install phase (assume `node_modules/` is present and current). |
| `--no-test` | off | Install and build only; skip the test phase. |

## Detection: Package Manager (lockfile priority)

Scan `path` and apply the **first** match top-down. The marker → framework → agent map lives in `skill: language-detection` — keep this list in sync with it, do not fork the routing logic.

| Priority | Lockfile | Package manager | Install command |
|----------|----------|-----------------|-----------------|
| 1 | `pnpm-lock.yaml` | pnpm | `pnpm install --frozen-lockfile` |
| 2 | `yarn.lock` | Yarn | `yarn install --immutable` (berry) / `yarn install --frozen-lockfile` (classic) |
| 3 | `bun.lockb` | Bun | `bun install --frozen-lockfile` |
| 4 | `package-lock.json` | npm | `npm ci` |
| 5 | `package.json` (no lockfile) | npm (fallback) | `npm install` (warn: no lockfile — non-reproducible) |

`--no-install` skips this phase entirely. With a corepack-pinned `packageManager` field in `package.json`, honor that pin over the lockfile heuristic.

## Detection: Framework (from package.json)

Resolve the framework from `package.json` `dependencies`/`devDependencies` to pick the build/test scripts and the owning agent on failure:

| Markers in `package.json` | Framework | Owning agent (on failure) |
|---------------------------|-----------|---------------------------|
| `next` | Next.js (React) | `frontend-developer:react-developer` |
| `react`, `react-dom` (no `next`) | React (Vite/CRA) | `frontend-developer:react-developer` |
| `nuxt` | Nuxt (Vue) | `frontend-developer:vue-developer` |
| `vue` (no `nuxt`) | Vue (Vite) | `frontend-developer:vue-developer` |
| `@sveltejs/kit` | SvelteKit | `frontend-developer:svelte-developer` |
| `svelte` (no kit) | Svelte (Vite) | `frontend-developer:svelte-developer` |
| `@angular/core` | Angular | `frontend-developer:angular-developer` |
| none of the above (lib/tooling) | TypeScript/tooling | `frontend-developer:typescript-developer` |

If `package.json` defines explicit `scripts.build`/`scripts.test`, prefer those over the framework defaults below.

## Canonical Command Table

Run these verbatim (substituting `path`, package manager, and `--prod`). Every command is single-invocation and tees to the log. `<pm-run>` = `npm run` / `pnpm --dir <path> run` / `yarn` / `bun run` per the detected manager.

| Framework | Build | Test (unit/component) |
|-----------|-------|------------------------|
| Next.js | `<pm-run> build` (`next build`) | `<pm-run> test` (Vitest/Jest), else `npx vitest run` |
| React (Vite) | `<pm-run> build` (`vite build`) | `npx vitest run` (fallback `npx jest`) |
| Nuxt | `<pm-run> build` (`nuxi build`) | `npx vitest run` |
| Vue (Vite) | `<pm-run> build` (`vite build`) | `npx vitest run` |
| SvelteKit | `<pm-run> build` (`vite build`) | `npx vitest run` |
| Svelte (Vite) | `<pm-run> build` (`vite build`) | `npx vitest run` |
| Angular | `<pm-run> build` (`ng build`) | `<pm-run> test -- --watch=false --browsers=ChromeHeadless` (Karma/Jest) |
| Library/tooling | `<pm-run> build` else `npx tsc --noEmit` | `<pm-run> test` else `npx vitest run` |

Notes:
- Always run `npx tsc --noEmit` as a type-check gate when a `tsconfig.json` is present, regardless of framework — a type error is a build failure.
- `--prod` swaps in the production build script and sets `NODE_ENV=production`; tests still run in their default mode.
- If no `test` script and no test runner config exists, report "no test target" rather than failing.

## Workflow

### Phase 1: Detect (Bash)

1. Confirm `path` exists. If not, emit the Error Handling "path not found" message and stop.
2. Create `.context/logs/` if absent. Compute `TS="$(date +%Y%m%d-%H%M%S)"` and `LOG=".context/logs/build-${TS}.log"`.
3. Walk the lockfile priority table; record the first matching manager. Read `package.json` for the framework and any explicit `build`/`test` scripts. If no `package.json`, emit "no web project" and stop.
4. Pre-resolve the owning framework agent (used only if a later phase fails).
5. Verify the manager binary exists (`command -v pnpm`/`yarn`/`bun`/`npm`). If missing, print the install hint (Graceful Degradation), fall back to the next eligible manager, and note the fallback.

### Phase 2: Install (Bash)

1. If `--no-install`: skip; record "install skipped (--no-install)".
2. Run the install command for the detected manager, teeing to the log.
3. Capture `${PIPESTATUS[0]}`. On non-zero, go to Failure Triage with stage `install`.

### Phase 3: Build (Bash)

1. Run `npx tsc --noEmit` (if `tsconfig.json` present), teeing to the log. On non-zero, classify as `build` (type error).
2. Run the framework build command from the table, teeing to the log.
3. Capture `${PIPESTATUS[0]}`. On non-zero, go to Failure Triage with stage `build`.

### Phase 4: Test (Bash)

1. If `--no-test`: skip; record "tests skipped (--no-test)".
2. Run the test command from the table, teeing to the log.
3. Capture `${PIPESTATUS[0]}`. On non-zero, go to Failure Triage with stage `test`.

### Phase 5: Report (Bash)

Emit the Output Format summary. On full success, stop — no routing.

## Failure Triage

Triggered only when a phase exits non-zero. Steps:

1. **Parse the first error** from the log (e.g. `npm ERR!`, `error TS####`, Vite/`esbuild` `error:`, `FAIL`/`✗`/`expected` for tests).
2. **Classify the stage:**

   | Symptom in log | Stage |
   |----------------|-------|
   | `ERESOLVE`, `peer dep`, `404 Not Found`, lockfile mismatch, `ENOENT` for a manifest | `install` |
   | `error TS####`, Vite/esbuild/webpack build error, `Module not found`, failed SSR build | `build` |
   | Vitest/Jest `FAIL`, `✗`, assertion/expectation failures, Karma test errors, nonzero test exit | `test` |

3. **Extract a tight excerpt** — the first error plus ~10 surrounding lines, not the whole log. Include the log path.
4. **Route to the matching framework agent** (resolved in Phase 1) with the excerpt:
   "Build-test failed at the **{stage}** stage for `{path}` (framework: {framework}, package manager: {pm}). First error and context from `{LOG}`:\n```\n{excerpt}\n```\nDiagnose the root cause and propose the minimal fix. If the fix touches build config (`vite.config`/`next.config`/`angular.json`/`tsconfig`), say so explicitly. Do not run the full build yourself; return the analysis and patch."
   - `install` failures route to the framework agent unless the failure is purely a dependency-resolution issue, in which case route to `frontend-developer:fe-dependency-manager`.
5. After the agent returns a fix, re-run from the failing phase. Do NOT auto-iterate silently — report each cycle.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint |
|--------------|--------------|
| `node` / `npm` | install Node.js LTS (`https://nodejs.org` or `brew install node`) |
| `pnpm` | `npm install -g pnpm` (or `corepack enable pnpm`) |
| `yarn` | `corepack enable` (Yarn ships via corepack) or `npm install -g yarn` |
| `bun` | `curl -fsSL https://bun.sh/install \| bash` (verify against your toolchain) |
| `vitest` / `jest` | `npm install -D vitest` (or `jest`) — installs into the project |

If the detected manager's binary is missing, print the hint, fall back to the next eligible manager (or npm), and note the fallback. Only when *every* eligible manager is missing — i.e. no `node` at all — does the command report a hard stop with the aggregated hints. Never hard-fail on a single missing optional tool.

## Output Format

```markdown
## Build & Test Report

**Target:** {path}
**Package manager:** {pnpm | yarn | bun | npm} ({lockfile})
**Framework:** {Next.js | React | Nuxt | Vue | SvelteKit | Svelte | Angular | library}
**Log:** .context/logs/build-{timestamp}.log

| Phase | Result | Notes |
|-------|--------|-------|
| Install | ✅ / ❌ / ⏭ skipped | {manager, or skip reason} |
| Type-check (tsc) | ✅ / ❌ / ⏭ | {0 errors, or first TS error} |
| Build | ✅ / ❌ / ⏭ | {prod/dev, warnings count} |
| Test | ✅ / ❌ / ⏭ | {N passed, M failed, or "--no-test"} |

**Result:** PASS / FAIL ({failing stage})

<!-- On failure only: -->
### Failure Triage
- **Stage:** {install | build | test}
- **First error:** {one-line summary}
- **Routed to:** frontend-developer:{agent}
- **Proposed fix:** {summary, or "see agent output"}
```

## Error Handling

### Path not found
```
Error: Path not found: {path}
Suggestion: Pass a directory that exists, e.g. /frontend-developer:build-test .
```

### No web project
```
Error: No package.json found under {path}.
Suggestion: Run from the directory that holds package.json, or scaffold one with `npm init`.
```

### No lockfile (non-reproducible install)
```
Warning: No lockfile found — falling back to `npm install` (non-reproducible).
Suggestion: commit a lockfile (pnpm-lock.yaml / package-lock.json) for reproducible CI builds.
```

### No test target
Not an error. Report "no `test` script / runner config found — build succeeded, tests skipped" and treat the run as PASS for build, N/A for test.

## See Also

- `skill: language-detection` — canonical marker → framework → agent routing (keep the priority table in sync).
- `skill: build-systems` — Vite/Next/Angular/Nuxt build idioms, monorepo workspaces, env handling.
- `/frontend-developer:lint-fix` — run linters before building to cut noise.
- `/frontend-developer:generate-tests` — add a test suite when detection finds no test target.
- `/frontend-developer:deps-audit` — when an `install`-stage failure is a missing or conflicting dependency.
- `/frontend-developer:profile-performance` — once the build is green, profile bundle and Web Vitals.
