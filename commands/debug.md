---
description: Configure browser and framework debugging workflows, or triage and root-cause a specific web error
argument-hint: [error message, stack trace, file, component, or scope]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch
estimated-cost:
  min-tokens: 2000
  max-tokens: 18000
  model-distribution:
    haiku: 15%
    sonnet: 70%
    opus: 15%
---

# Debug

Set up the debugging surface for a web project — working source maps, browser devtools breakpoints, the right framework devtools, console/network capture — **or** take a concrete error already in hand and root-cause it. Which one runs depends on what you pass. Debugging work routes to `frontend-developer:frontend-developer`, which owns cross-framework triage; deep root-cause analysis delegates to `debugging-toolkit:debugging-toolkit-debugger`.

[Extended thinking: Web debugging fails for two distinct reasons, and they need opposite treatments. The first is that the debugging surface itself is broken — the stack trace points at `index-4f2a.js:1:90432` because source maps are not emitted or not resolving, there is no React/Vue devtools connection, HMR silently died and the page is showing stale code, so every subsequent observation is untrustworthy. That is Configure mode: fix the instruments before reading them. The second is a specific, reproducible error where the surface is fine and the answer is a hypothesis-driven hunt. That is Triage mode. Detection is straightforward — an argument that reads as an error message, stack trace, or console output is triage; a component, path, or empty argument is configure. Triage is deliberately advisory by default: the command classifies, root-causes, and proposes the minimal fix, because a wrong "fix" to a hydration mismatch or a bundler resolution error usually just moves the symptom. The web failure-class table is the fast path — hydration mismatch, HMR failure, bundler resolution, CORS, unhandled rejection cover most of what actually lands here, and each has a known first probe.]

## Modes

- **Configure mode (default)** — invoked with a component, path, or no argument: set up the debugging surface (source maps, breakpoints, framework devtools, console/network capture, log retention) for ongoing investigation. This is the workflow below.
- **Triage mode** — invoked with a concrete error message, stack trace, or console/network excerpt: classify → root-cause → propose the minimal fix for a single error. Jump to **Triage Mode** near the end of this file.

Detect triage mode when `$ARGUMENTS` contains an error string, a stack frame (`at fn (…:12:34)`), a framework warning (`Hydration failed…`, `Cannot read properties of undefined`, `Failed to resolve import`), or an HTTP/CORS message — rather than a component name, path, or empty scope.

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Fix the instruments before reading them.** If source maps do not resolve, or the framework devtools cannot attach, say so and repair that first. Do NOT reason about a minified stack trace as if the frame names were real.
2. **Reproduce before hypothesizing.** State the narrowest reliable trigger (URL, interaction, viewport, build mode) and whether the failure is deterministic or flaky. Dev-only and prod-only failures have different root-cause families — always record which build mode reproduces.
3. **One hypothesis at a time, each with a probe.** Every hypothesis names the single observation that would confirm or kill it (a devtools panel, a network entry, a log line, a bisect step). Do NOT list causes without probes.
4. **Triage is advisory by default.** In triage mode, classify, root-cause, and propose the minimal fix. Do NOT mutate source unless the user explicitly asks. `Write`/`Edit` are in `allowed-tools` because Configure mode writes launch configs and bundler sourcemap settings.
5. **Configure mode writes config, not features.** It may edit `vite.config.*`/`webpack.config.*` `devtool`/`sourcemap` settings, `.vscode/launch.json`, and devtools/logging setup. It MUST NOT change application logic.
6. **Tool-missing never hard-fails.** If a browser, framework devtools extension, or CLI is unavailable, print the install hint, note the reduced depth, and continue with what is observable. Never abort the session over one missing tool.
7. **Commands route, they do not orchestrate.** This command names which agent owns each lens so Claude routes the work; it does not call `Task` itself.
8. **Verify after any fix.** Once a fix is applied (by the user or by a routed agent), run `/frontend-developer:build-test` and re-run the reproduction. An unverified fix is a hypothesis, and must be reported as one.
9. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Configure the debugging surface for the whole project
/frontend-developer:debug

# Configure debugging around a specific area
/frontend-developer:debug src/features/checkout
/frontend-developer:debug CartDrawer

# Triage a specific runtime error (triage mode)
/frontend-developer:debug "TypeError: Cannot read properties of undefined (reading 'items')"

# Triage a hydration mismatch
/frontend-developer:debug "Hydration failed because the server rendered HTML didn't match the client"

# Triage a bundler resolution failure
/frontend-developer:debug "Failed to resolve import \"@/lib/api\" from \"src/App.tsx\""

# Triage a pasted stack trace (quote the whole trace)
/frontend-developer:debug "at o (index-4f2a.js:1:90432)\n at r (index-4f2a.js:1:88110)"
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `scope` | project root | Component, file, or directory to instrument. Configure mode only. |
| `error / stack trace` | — | A concrete error string switches the command into Triage mode. |

## Web Failure Classes

The fast path. Match the symptom, run the first probe, then continue with the mode workflow.

| Failure class | Typical symptom | First probe |
|---------------|-----------------|-------------|
| Hydration mismatch | "Hydration failed", "text content did not match", flash of wrong content on SSR/SSG | Diff server HTML (view-source) vs client DOM; look for `Date`/`Math.random`/`window`/locale in render, and invalid nesting (`<div>` inside `<p>`) |
| HMR failure | Edits do not appear; full reload loop; stale module state | Dev-server console + browser console for HMR errors; check the WebSocket in the Network panel (`ws://…/__vite_hmr` or webpack-dev-server) |
| Source maps not resolving | Stack frames point at `bundle.js:1:90432` | Confirm `.map` files are emitted and fetched (Network), and that devtools shows original sources in the Sources tree |
| Bundler resolution error | "Failed to resolve import", "Module not found", "Cannot find module" | Compare the specifier against `tsconfig` `paths`, the bundler `resolve.alias`, and the package's `exports` map; check case sensitivity |
| CORS / preflight | "blocked by CORS policy", failing `OPTIONS` request | Network panel: inspect the preflight and the `Access-Control-Allow-*` response headers; confirm credentials mode matches |
| Unhandled promise rejection | "Uncaught (in promise)", silent failure, no UI update | Break on `unhandledrejection`; add the listener; find the `await` with no `catch` and the missing error boundary |
| Stale/incorrect state | UI does not update, or updates twice | Framework devtools: inspect the component's props/state/signals, and the store timeline (Redux DevTools) |
| Env/config drift | Works in dev, fails in prod build | Diff `import.meta.env`/`process.env` between modes; check what is inlined at build time |

Depth for source maps and hydration lives in `skill: fe-diagnostics` — do not fork its triage tables here.

## Configure Mode

### Phase 1: Establish the surface (Bash + Read)

1. Detect the framework, bundler, and package manager from `package.json` (per `skill: language-detection`). Record dev vs prod build commands.
2. **Source maps** — verify they are configured and resolving:
   - Vite: `build.sourcemap: true` in `vite.config.*` for production traces (dev has them by default).
   - webpack: `devtool: 'source-map'` (prod) / `'eval-source-map'` (dev). Never ship `eval-*` to production.
   - Next.js: `productionBrowserSourceMaps: true` when production traces must be readable.
   - Confirm `.map` files are emitted next to the bundles and that devtools resolves original file names in the Sources tree.
3. **Framework devtools** — confirm the right extension attaches, and note what it gives you:

   | Framework | Devtools | Attaches when |
   |-----------|----------|---------------|
   | React | React DevTools (Components + Profiler) | dev build, or `react-devtools` standalone for prod |
   | Vue | Vue DevTools (component tree, Pinia timeline) | dev build with `__VUE_PROD_DEVTOOLS__` off in prod |
   | Angular | Angular DevTools (component tree, change-detection profiler) | dev build (`ng serve`) |
   | Svelte | Svelte DevTools / the SvelteKit inspector (`inspector: true`) | dev build |
   | Redux / RTK | Redux DevTools (action log, time travel, state diff) | store composed with the devtools enhancer |

4. **Console and network capture** — set console filters (preserve log across navigations), enable "break on caught/uncaught exceptions" where appropriate, and record which Network panel columns matter for this scope (status, type, initiator, timing).
5. Report the surface state before going further: source maps ✅/❌, devtools ✅/❌, HMR ✅/❌. Anything ❌ is repaired before hypothesizing (Rule 1).

### Phase 2: Instrument the scope

1. **Breakpoints** — name the concrete places to break: the render/effect boundary, the state transition, the fetch call site. Prefer conditional breakpoints (`count > 10`) and logpoints over `console.log` edits, so no application code changes.
2. **Event listener breakpoints** — for interaction bugs, break on the DOM event rather than guessing the handler.
3. **`debugger` statements** — only when a breakpoint cannot be placed (e.g. code inside a generated chunk); remove before commit.
4. **Editor attach (optional)** — write a `.vscode/launch.json` Chrome/Edge launch or attach configuration pointing at the dev-server URL with `webRoot` set, so breakpoints bind to source files.
5. **Performance/Network panels** — for jank or slow loads, record a Performance trace over the interaction and read Long Tasks, layout thrash, and the LCP/INP markers; use the Network panel waterfall for request ordering, blocking, and payload size.

Routing: framework-specific instrumentation (what to watch in the component tree, which store transition matters) goes to the owning framework agent — `frontend-developer:react-developer`, `vue-developer`, `svelte-developer`, or `angular-developer`. Bundler/source-map configuration goes to `frontend-developer:typescript-developer` when it is a `tsconfig`/type-emit question, otherwise stays in this command.

### Phase 3: Capture and retain

1. Save the reproduction as a written recipe (URL, build mode, viewport, exact interaction sequence, expected vs actual).
2. Export what devtools gives you: a HAR from the Network panel, the Performance trace, and the console log — store under `.context/logs/` with a timestamp, as `/frontend-developer:build-test` does.
3. Note which observations were **not** available (missing extension, prod build without maps) so later reasoning stays honest about its evidence.

## Triage Mode (root-cause a specific error)

When `$ARGUMENTS` is a concrete error, stack trace, or console/network excerpt, run this pipeline instead of the configure workflow.

> **Tool discipline:** triage is advisory/read-only by default — classify, root-cause, and propose the minimal fix for the user to apply (Rule 4).

### Triage Phase 1: Classify

Classify by **severity** (blocking build / blocking user flow / degraded / cosmetic) and **class** (build-time, bundler resolution, runtime type error, hydration/SSR, state/reactivity, network/CORS, performance, configuration). Match against the Web Failure Classes table above.

If the trace is minified, STOP and resolve source maps first (Configure Phase 1.2) — a minified frame name is not evidence.

Produce 3–5 ranked hypotheses, each with the single probe that confirms or kills it.

### Triage Phase 2: Root-cause

Route to `debugging-toolkit:debugging-toolkit-debugger` for the deep pass, with the classification and the ranked hypotheses attached:

"Root-cause this web error: {error}. Classification: {class/severity}. Ranked hypotheses: {hypotheses}. Build mode that reproduces: {dev|prod|both}. Framework/bundler: {detected}. Use Five Whys. Evidence available: {stack trace (source-mapped), console log, HAR, Performance trace, recent diffs}. Correlate with recent changes to the touched modules, their dependencies, and build config. Identify the root-cause category (logic, state/reactivity, SSR/hydration, module resolution, network/CORS, configuration, dependency version) and the smallest proof step that confirms it."

Run the framework-specific lens in parallel where the class calls for it: hydration and reactivity go to the owning framework agent, module resolution and type errors to `frontend-developer:typescript-developer`, dependency/version conflicts to `frontend-developer:fe-dependency-manager`.

### Triage Phase 3: Resolve

1. **Minimal fix** — the smallest change that addresses the root cause, with `file:line` references. Say explicitly when the fix touches build config (`vite.config`/`next.config`/`webpack.config`/`tsconfig`/`angular.json`) rather than application code.
2. **Verification** — the exact reproduction step that must now pass, plus `/frontend-developer:build-test`.
3. **Regression guard** — the test that would have caught this (`/frontend-developer:gen-tests` if it does not exist).
4. **Prevention** — the lint rule, type change, error boundary, or config guard that makes the class unreachable.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect |
|--------------|--------------|--------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | no dev server, no build reproduction; static source reasoning only |
| Chrome / Chromium | install Chrome or Edge (any Chromium build ships the same devtools) | no live devtools; rely on the pasted trace, logs, and source reading |
| React / Vue / Angular / Svelte devtools extension | install from the browser extension store (React DevTools also runs standalone: `npx react-devtools`) | component-tree and profiler inspection unavailable — note the reduced depth |
| Redux DevTools | `npm install -D @redux-devtools/extension` plus the browser extension | no action log or time travel; inspect store state via breakpoints instead |
| `playwright` | `npm install -D @playwright/test && npx playwright install` | no scripted reproduction or trace capture; reproduce manually |
| `source-map` CLI | `npm install -D source-map` | map frames by loading the `.map` in devtools instead |

Tool-missing is always a skip-with-note, never a hard failure. The command only reports "cannot debug" when there is no reproduction and no error evidence at all.

## Output Format

```markdown
## Debug Report

**Mode:** configure | triage
**Scope / Error:** {scope or the error's first line}
**Framework / bundler:** {detected}
**Reproduces in:** dev / prod / both / not reproduced

### Debug Surface
| Instrument | State | Note |
|------------|-------|------|
| Source maps | ✅ / ❌ | {devtool setting, .map emitted?} |
| Framework devtools | ✅ / ⏭ | {React/Vue/Angular/Svelte, attached?} |
| HMR / dev server | ✅ / ❌ | {WebSocket connected?} |
| Console / Network capture | ✅ / ⏭ | {HAR + log path under .context/logs/} |

<!-- configure mode -->
### Instrumentation
| Location | Instrument | Purpose |
|----------|------------|---------|
| src/…:{line} | conditional breakpoint / logpoint / event listener break | {what it proves} |

**Reproduction recipe:** {URL, build mode, viewport, interaction sequence, expected vs actual}

<!-- triage mode -->
### Classification
- **Class:** {hydration | HMR | resolution | CORS | unhandled rejection | state | config | performance}
- **Severity:** {blocking build | blocking flow | degraded | cosmetic}

### Hypotheses
| # | Hypothesis | Probe | Verdict | Confidence |
|--:|-----------|-------|---------|-----------|
| 1 | {statement} | {single observation} | confirmed / killed / open | high/med/low |

### Root Cause
{The fundamental cause with evidence — source-mapped frames, network entries, diffs. Say "not established" when it is not.}

### Proposed Fix (advisory)
| File:Line | Change | Touches build config? |
|-----------|--------|-----------------------|

### Verification & Prevention
- **Verify:** {reproduction step that must now pass} + `/frontend-developer:build-test`
- **Regression test:** {test to add, or "covered by {existing test}"}
- **Prevention:** {lint rule / type change / error boundary / config guard}

<!-- when a tool was unavailable -->
### Reduced-Depth Notes
- {tool} unavailable — {what could not be observed}. Install: {hint}.
```

## Error Handling

### Minified stack trace with no source maps
```
Note: The trace frames are minified and no source map resolves.
Resolving source maps first (build.sourcemap / devtool), because minified frame
names are not evidence. See the Web Failure Classes table.
```

### Not reproducible
```
Note: The reported error did not reproduce under {dev|prod}, {viewport}, {steps}.
Suggestion: capture the failing session (HAR + console log + build mode + browser
version) and re-run /frontend-developer:debug with that evidence attached.
```

### No error and no scope
```
Note: No error argument and no scope given — running Configure mode over the project root.
Suggestion: pass a component/path to narrow it, or paste the error to switch to Triage mode.
```

## See Also

- `skill: fe-diagnostics` — canonical symptom → fix routing for source maps, hydration mismatch, HMR, and build errors.
- `skill: build-systems` — Vite/webpack/Next/Angular config, `devtool`/`sourcemap` settings, env handling.
- `skill: web-performance` — when the "bug" is jank or a slow load, not an exception.
- `/frontend-developer:build-test` — the Rule 8 verification gate; also the source of the first build/test error excerpt.
- `/frontend-developer:fix-quick` — clear mechanical lint noise before triaging, so real findings stand out.
- `/frontend-developer:gen-tests` — add the regression test that would have caught the confirmed root cause.
- `/frontend-developer:fix-performance` — escalate a confirmed performance root cause to a full profiling pass.
- `/frontend-developer:review-code` — when triage exposes a defect class rather than a single bug.
- **When to use which:** `/frontend-developer:debug` with no error argument configures the debugging surface (source maps, breakpoints, framework devtools, capture) for ongoing investigation. `/frontend-developer:debug "<error|stack trace>"` triages and root-causes a *single* error already in hand.
