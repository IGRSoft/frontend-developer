---
description: Profile web performance with Lighthouse, Core Web Vitals, and bundle analysis, then route findings to the performance engineer for a ranked fix plan
argument-hint: [path or URL (default: detect dev server)] [--mode lighthouse|vitals|bundle] [--budget]
allowed-tools: Read, Glob, Grep, Bash, WebSearch, WebFetch
estimated-cost:
  band: high
  min-tokens: 2000
  max-tokens: 18000
  model-distribution:
    haiku: 25%
    sonnet: 60%
    opus: 15%
---

# Profile Performance
<!-- Updated: June 2026 -->

Collect a Lighthouse report, Core Web Vitals measurements, and a bundle analysis of a web app, then hand the raw artifacts to `frontend-developer:fe-performance-engineer` for interpretation and a ranked fix plan. Data collection is pure shell; the agent is engaged only to read the reports and rank hot paths. The command never guesses at the bottleneck itself.

[Extended thinking: Web performance is a measure-first discipline, so this command's job is to produce *trustworthy* measurements against a representative build and then defer judgment. The single most common way front-end profiling lies is a wrong build: a dev server with HMR and unminified modules has nothing to do with production LCP, so the prerequisite check refuses to profile a dev server for `lighthouse`/`bundle` modes and tells the user to build + preview production. It collects three complementary signals — Lighthouse (lab CWV + opportunities), field/lab Web Vitals (LCP, INP, CLS, TTFB), and a bundle analysis (treemap, per-chunk size, duplicate deps) — saves every artifact under `.context/images/<worktask_id>/perf-<timestamp>/`, then lets the performance engineer produce the ranked findings. `--budget` turns the run into a pass/fail gate against the Core Web Vitals thresholds. Interpretation — top hotspots, the LCP element, the INP-blocking long task, the bundle bloat source, a fix plan ranked by effort/impact — is the agent's deliverable, not this command's.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Profile a production build, never a dev server (for `lighthouse`/`bundle`).** Lighthouse and bundle numbers from `npm run dev` are meaningless. Build production and serve it (`npm run build` + `npm run preview` / serve `dist/`), or accept an explicit production URL. If only a dev server is reachable, STOP for those modes and emit the build instruction.
2. **Collection is shell-only; interpretation is the agent's.** This command runs the tools and saves artifacts. It does NOT eyeball a trace and declare a winner. Hand the artifacts to `frontend-developer:fe-performance-engineer` and let it produce the ranked findings.
3. **Save every artifact under `.context/images/<worktask_id>/perf-<timestamp>/`.** Write the Lighthouse JSON/HTML, the vitals JSON, the bundle treemap/stats, and a `meta.txt` (target, mode, build, tool) there — this is the source of truth and supporting evidence for the DV screenshot manifest.
4. **Single-command Bash invocations.** Use each tool's own output flags. Never `cd`-chain or `&&`-chain — scoped Bash patterns do not match compound commands.
5. **Pick the tool, do not invent flags.** Run the matching tool from the Tool Matrix exactly as written. If a flag is rejected, consult `--help` or `skill: fe-diagnostics` — never guess flag spellings.
6. **`--budget` records a pass/fail gate, never auto-edits.** Budget mode compares measured CWV against the thresholds and reports PASS/FAIL. It does not change code. The optimization is the agent's plan plus a follow-up `code-review`/`code-modernize` run.
7. **Tool-missing never hard-fails.** If a tool is absent, print the install hint, skip that collection, and report what was skipped. If no tool is available at all, report the aggregated hints and stop without erroring out the session.
8. **Commands route, they do not orchestrate.** This command names `frontend-developer:fe-performance-engineer` so Claude routes the interpretation; it does not call `Task`.
9. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Lighthouse + vitals + bundle on the detected production preview
/frontend-developer:profile-performance

# Lighthouse against a production URL
/frontend-developer:profile-performance https://staging.example.com --mode lighthouse

# Bundle analysis only
/frontend-developer:profile-performance . --mode bundle

# Web Vitals against a route, gated on the CWV budget
/frontend-developer:profile-performance http://localhost:4173/ --mode vitals --budget
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `path or URL` | detect preview | A production URL to profile, or a project dir (the command builds + previews it). If omitted, detect/start a production preview server. |
| `--mode lighthouse\|vitals\|bundle` | all three | `lighthouse` = full lab report (perf score + opportunities); `vitals` = LCP/INP/CLS/TTFB measurement; `bundle` = treemap + per-chunk size + duplicate deps. Default runs all three. |
| `--budget` | off | Compare measured Core Web Vitals against the thresholds and report PASS/FAIL as a gate (LCP < 2.5s, INP < 200ms, CLS < 0.1, TTFB < 800ms). |

## Build Prerequisite Check (lighthouse / bundle)

Before Lighthouse or bundle collection, confirm a production build is what is being profiled:

| Check | How | Fail action |
|-------|-----|-------------|
| Production build exists | `dist/`/`.next/`/`build/` present and newer than source, OR an explicit prod URL given | Build it: `npm run build`, then preview |
| Served from a preview, not dev | URL is a preview/static server (`vite preview`/`serve`), not `vite`/`next dev` HMR | Emit the build+preview instruction; do not profile a dev server |

Build+preview instruction to print on failure:

```bash
# Production build + preview (Vite example)
npm run build
npm run preview   # serves the optimized build on a local port
# Next.js: npm run build && npm run start
# Then re-run profile-performance against the preview URL.
```

`vitals` mode can run against any reachable URL (field measurement is meaningful on staging/prod), but for a *representative* lab measurement, prefer the production preview too.

## Tool Matrix

Run the command verbatim, substituting target/URL and output dir (`OUT=.context/images/<worktask_id>/perf-<timestamp>`). The canonical flag reference is `skill: fe-diagnostics` — keep this matrix in sync with it, do not fork the flag spellings.

### Lighthouse (`--mode lighthouse`)

| Step | Command |
|------|---------|
| Lab report | `npx lighthouse "<url>" --only-categories=performance --output=json --output=html --output-path="$OUT/lh" --quiet --chrome-flags="--headless"` |
| CI thresholds (optional) | `npx lhci autorun` if `lighthouserc.*` exists — record the assertion results |

### Web Vitals (`--mode vitals`)

| Step | Command |
|------|---------|
| Lab CWV | parse `largest-contentful-paint`, `interaction-to-next-paint` (or `total-blocking-time` proxy), `cumulative-layout-shift`, `server-response-time` from the Lighthouse JSON into `"$OUT/vitals.json"` |
| Field CWV (if available) | query the CrUX API via WebFetch for the origin/URL: `https://chromeuxreport.googleapis.com/...` (real-user p75 LCP/INP/CLS) — note when field data is unavailable for low-traffic origins |

### Bundle Analysis (`--mode bundle`)

| Bundler | Command |
|---------|---------|
| Vite | `npx vite-bundle-visualizer -o "$OUT/bundle-treemap.html"` (or `rollup-plugin-visualizer` if configured) |
| webpack | `npx webpack-bundle-analyzer <stats.json> "$OUT" --mode static --report "$OUT/bundle-report.html"` (generate `stats.json` via `--json`) |
| Next.js | `ANALYZE=true npm run build` with `@next/bundle-analyzer`, capture the report into `"$OUT"` |
| any | `npx source-map-explorer 'dist/**/*.js' --html "$OUT/sme.html"` as a bundler-agnostic fallback |

Record per-chunk sizes (gzip + brotli), the largest modules, and duplicate dependencies (multiple versions of the same package).

## Workflow

### Phase 1: Resolve target & build (Bash)

1. Confirm `path or URL` exists/reachable. If not, emit the Error Handling "target not found" message and stop.
2. For `lighthouse`/`bundle`, run the Build Prerequisite Check. If it fails, print the build+preview instruction and STOP for those modes (vitals can still run against a reachable URL).
3. Detect the bundler from `package.json`/config (`vite.config`/`next.config`/`webpack.config`) for the bundle row.
4. Create the artifact dir once: `TS="$(date +%Y%m%d-%H%M%S)"; OUT=".context/images/<worktask_id>/perf-${TS}"; mkdir -p "$OUT"`. Write `meta.txt`.

### Phase 2: Collect (Bash)

1. Verify each mode's tool exists (`command -v npx`; `lighthouse`, the visualizer resolve via the project/npx). Missing → print install hint, skip that collection, note the skip.
2. Run each requested mode's commands, writing artifacts into `"$OUT"`. Capture exit status (`${PIPESTATUS[0]}` where piped).
3. For `--budget`, extract the measured CWV from the Lighthouse/vitals JSON and compare against the thresholds.

### Phase 3: Interpret (route)

After artifacts are written, hand them to the performance engineer for the ranked analysis.

Route to `frontend-developer:fe-performance-engineer`:
"Interpret the web-performance profile of `{target}` ({modes}). Artifacts in `{OUT}` (build/tool in `{OUT}/meta.txt`). Produce: (1) the **Core Web Vitals** verdict (LCP/INP/CLS/TTFB vs thresholds) and the LCP element / INP-blocking long task identified from the trace; (2) the **top opportunities** as `file:line` or chunk with their estimated savings; (3) for bundle mode, the **largest modules and duplicate dependencies** with a code-split / tree-shake / dynamic-import recommendation; (4) a **fix plan ranked by effort/impact** (render-blocking and LCP wins before micro-optimizations), each item naming the framework agent that would implement it. Do NOT edit code — return the ranked analysis."

Model note: `fe-performance-engineer` defaults to sonnet/high; a caller may raise it to opus for a large cross-route profile.

### Phase 4: Report (Bash)

Emit the Output Format summary, pointing at `{OUT}` and folding in the engineer's ranked findings (or the skip note when collection did not run). For `--budget`, lead with the PASS/FAIL gate.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect |
|--------------|--------------|--------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | no collection — report what is missing |
| `lighthouse` | `npm install -D lighthouse` (or `npx lighthouse`) | skip lighthouse + lab vitals; bundle still runs |
| Chrome/Chromium | install Chrome, or `npx playwright install chromium` | Lighthouse cannot render → skip with note |
| `vite-bundle-visualizer` / `source-map-explorer` | `npm install -D vite-bundle-visualizer` (or `source-map-explorer`) | skip bundle treemap; report per-chunk size from build output instead |
| `webpack-bundle-analyzer` | `npm install -D webpack-bundle-analyzer` | skip webpack treemap; fall back to `source-map-explorer` |

If a tool is missing, degrade to the remaining signals (e.g. report raw `dist/` chunk sizes from the build manifest when no visualizer is installed) and note the reduced depth. Never hard-fail on a single missing optional tool: print the hint, skip the collection, continue.

## Output Format

```markdown
## Performance Profile Report

**Target:** {url | path}
**Modes:** {lighthouse, vitals, bundle}
**Build:** production preview ✅ / (vitals against {url})
**Tools:** {lighthouse | vite-bundle-visualizer | source-map-explorer | CrUX}
**Artifacts:** .context/images/{worktask_id}/perf-{timestamp}/

<!-- --budget mode: lead with the gate -->
### Core Web Vitals Gate
| Metric | Measured | Threshold | Verdict |
|--------|---------:|----------:|---------|
| LCP | 3.1 s | < 2.5 s | ❌ |
| INP | 140 ms | < 200 ms | ✅ |
| CLS | 0.04 | < 0.1 | ✅ |
| TTFB | 620 ms | < 800 ms | ✅ |

**Budget:** PASS / FAIL ({n metrics over budget})

### Lighthouse
- Performance score: {0-100}
- Top opportunities: {render-blocking resources, unused JS, image format, ...}

### Top Hotspots
<!-- from fe-performance-engineer -->
| Rank | Location (file:line / chunk) | Cost | Note |
|-----:|------------------------------|-----:|------|
| 1 | vendor-charts.js | 280 KB gzip | charting lib loaded eagerly — dynamic-import |
| 2 | Hero.tsx:40 | LCP element | unoptimized hero image — use responsive `srcset` |

### Bundle
| Chunk | Size (gzip) | Note |
|-------|------------:|------|
| index.js | 120 KB | within budget |
| vendor.js | 310 KB | `moment` + `date-fns` — duplicate date libs, drop one |

### Ranked Fix Plan
<!-- from fe-performance-engineer; effort/impact order -->
1. {high-impact / low-effort fix} — implement via frontend-developer:{agent}
2. {next} — ...

<!-- on skip only -->
### Skipped
- {tool}: {missing — install hint above} | {dev-server only — build production first}
```

## Error Handling

### Target not found
```
Error: Target not found / unreachable: {target}
Suggestion: Pass a reachable URL or a project directory, e.g.
/frontend-developer:profile-performance http://localhost:4173 --mode lighthouse
```

### Dev server given for lighthouse/bundle
```
Error: {url} looks like a dev server (HMR / unminified). Lighthouse and bundle
numbers from a dev build are not representative.
Build production and preview it:
  npm run build && npm run preview
Then re-run against the preview URL.
```
This is a STOP for lighthouse/bundle, not a skip. (vitals may still run.)

### Rendering unavailable (no browser)
```
Warning: No Chromium available for Lighthouse.
Install: npx playwright install chromium
Skipping Lighthouse — bundle analysis (if requested) still runs.
```

### Tool missing
Print the install hint, skip that collection, continue. Only when *every* requested collection is unavailable does the command report "no profiler available" with the aggregated hints (no hard failure).

## See Also

- `skill: fe-diagnostics` — Lighthouse/bundle-analyzer/source-map-explorer flag reference and the symptom→tool table. Keep the Tool Matrix in sync with it.
- `skill: web-performance` — Core Web Vitals budgets (LCP < 2.5s, INP < 200ms, CLS < 0.1), the optimization playbook.
- `skill: bundling-optimization` — code-splitting, tree-shaking, dynamic imports, dependency de-duplication.
- `/frontend-developer:build-test` — produce the production build first, then profile it.
- `/frontend-developer:code-modernize` — apply the ranked fixes (e.g. migrate to a lighter library) the engineer recommends.
- `frontend-developer:fe-performance-engineer` — the review-only agent the interpretation routes to.
