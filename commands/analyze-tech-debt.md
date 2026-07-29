---
description: Identify, quantify, and prioritize frontend tech debt across code, types, CSS, dependencies, tests, and framework majors
argument-hint: [scope: file/dir — default: repo root] [--category code|types|css|deps|tests|framework] [--budget]
allowed-tools: Read, Glob, Grep, Bash, WebSearch, WebFetch
estimated-cost:
  min-tokens: 5000
  max-tokens: 30000
  model-distribution:
    haiku: 15%
    sonnet: 55%
    opus: 30%
---

# Technical Debt Analysis
<!-- Updated: July 2026 -->

Inventory a web frontend's technical debt, attach a number to each item, and rank the whole set by payback rather than by how much it annoys anyone. Covers six categories: legacy code patterns (class components, options-API holdouts, NgModule-era Angular), type debt (`any` sprawl, `@ts-ignore`, non-strict `tsconfig`), CSS debt (`!important` wars, dead selectors, forked design tokens), dependency debt (unused, duplicated, abandoned, CVE-bearing), test debt (coverage gaps on critical paths, skipped and flaky specs), and framework debt (majors behind, deprecated APIs, EOL runtimes). **Read-only — this command measures and plans; it never edits.**

[Extended thinking: "Tech debt" reports fail in two directions. They either list every code smell in the repo — producing an unactionable wall — or they describe the debt qualitatively, so the business has no basis to fund the work. The fix is to make every item countable with a cheap, repeatable command (`grep -c ' as any'`, `npx depcheck`, `npm outdated`, coverage totals) so the same measurement can be re-run next quarter to show the trend. Frontend debt also has a distinctive shape the generic playbooks miss: it compounds through the *bundle* (an abandoned date library is both dead weight and a CVE surface), through the *type boundary* (one `any` at a fetch seam erases types for everything downstream), and through *framework majors* (skipping one React/Angular/Vue major makes the next one exponentially harder). So the categories here are frontend-specific and the prioritization weighs blast radius — user-facing risk and bundle cost — not just effort. Remediation is deliberately out of scope: this command hands off a ranked plan, and every remediation step is gated on a green build.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **READ-ONLY. Always.** This command MUST NOT write or edit any file — not source, not manifests, not config. It produces a report as text. Remediation is a separate, explicit command.
2. **Every item is quantified.** Each finding carries a **count** and the **exact command or file:line** that produced it. "Lots of `any`" is not a finding; "`147` occurrences of `as any` across `39` files, via `grep -rc`" is. An unmeasurable item is a note, not a debt item.
3. **Measure with cheap, repeatable commands.** Use the commands in the Measurement Table so the same numbers can be regenerated later and trended. Record the command next to the number.
4. **Rank by payback, not by annoyance.** Every item gets impact (user-facing risk, velocity cost, bundle cost) × effort. The output is ordered by that ranking, not by category.
5. **Do not confuse debt with a bug.** A live defect goes to `/frontend-developer:review-code` or `/frontend-developer:debug`. Debt is the structural cost that makes future defects likelier or changes slower.
6. **Every remediation step is build-gated.** The plan must state that each step runs `/frontend-developer:build-test` before the next begins. Never propose a batched multi-category sweep with a single gate at the end.
7. **Tool-missing never hard-fails.** If a scanner is absent, print the install hint, fall back to the grep-level measurement, and mark that category **partial**. Never silently drop a category.
8. **Commands route, they do not orchestrate.** This command names `frontend-developer:frontend-architector` as the owner so Claude routes the prioritization; it does not call `Task` itself.
9. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Full debt inventory for the repo
/frontend-developer:analyze-tech-debt

# Scope to one area
/frontend-developer:analyze-tech-debt src/features/checkout

# One category only
/frontend-developer:analyze-tech-debt --category types
/frontend-developer:analyze-tech-debt --category deps

# Add a debt-budget section with quarterly targets and prevention gates
/frontend-developer:analyze-tech-debt --budget
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `scope` | repo root | Directory or file to inventory. Build artifacts are always excluded. |
| `--category code\|types\|css\|deps\|tests\|framework` | all six | Restrict the inventory to one category for a focused pass. |
| `--budget` | off | Add a Debt Budget section: quarterly reduction targets, prevention gates, and the CI checks that hold the line. |

Always exclude `node_modules/`, `dist/`, `build/`, `.next/`, `.svelte-kit/`, `.nuxt/`, `coverage/`, and lockfiles from every scan.

## Debt Categories

### 1. Legacy Code Debt

| Item | Signal | Why it costs |
|------|--------|--------------|
| React class components | `extends React.Component` / `extends Component` | Excluded from hooks, Suspense, and the React Compiler; every new feature needs a wrapper |
| Legacy lifecycle methods | `componentWillMount`, `componentWillReceiveProps`, `UNSAFE_` | Blocks concurrent rendering |
| Vue Options API holdouts | `export default { data()`, `methods:` in `.vue` | Splits the codebase into two mental models; no `<script setup>` typing |
| Svelte 4 reactive statements | `$:` labels in a Svelte 5 project | Mixed reactivity models; runes migration debt |
| Angular NgModule-era code | `@NgModule` with declarations, constructor DI in new code | Blocks standalone components, signals, and modern DI |
| Deprecated framework APIs | `ReactDOM.render`, `defaultProps` on function components, `entryComponents`, `v-deep`/`::v-deep` | Removed or removal-scheduled in the next major |
| God components | Files > ~400 lines owning fetch + transform + layout + interaction | Untestable; every change is a merge conflict |
| Prop-drilling chains | Same prop threaded 4+ levels | Every signature change ripples |

### 2. Type Debt

| Item | Signal | Why it costs |
|------|--------|--------------|
| `any` sprawl | `: any`, `as any`, `<any>`, implicit any from untyped modules | One `any` at a fetch seam erases typing for everything downstream |
| Suppressed errors | `@ts-ignore`, `@ts-expect-error` without a reason comment, `@ts-nocheck` | Hides real breakage; survives refactors invisibly |
| Non-strict config | `strict: false`, `noImplicitAny: false`, `strictNullChecks: false` in `tsconfig.json` | The whole codebase's null-safety guarantee is off |
| Untyped module boundaries | Missing `@types/*`, `declare module '*'` catch-alls | Cross-package type erasure |
| `Function`/`object`/`{}` types | Bare structural escape hatches | No call-signature or shape checking |
| JS files in a TS project | `.js`/`.jsx` under `src/` with `allowJs` | Migration stalled mid-way |

See `skill: ts-typing` for elimination patterns and `skill: ts-config` for the strict-flag ladder.

### 3. CSS & Design-System Debt

| Item | Signal | Why it costs |
|------|--------|--------------|
| `!important` escalation | `!important` count per file | Specificity wars; each new rule needs a bigger hammer |
| Deep selector nesting | 4+ level selectors, `>>>`/`::v-deep` piercing | Breaks on any markup change |
| Hard-coded design values | Raw hex colors, `px` spacing where tokens exist | Theming and dark mode become impossible without a sweep |
| Forked/duplicate tokens | Multiple sources of truth for color/spacing scales | Visual drift across the app |
| Dead CSS | Selectors unmatched anywhere in templates/JSX | Pure bundle weight |
| Unprefixed modern features | `:has()`, container queries, subgrid with no fallback | Silent layout break on older targets (`skill: version-feature-matrix`) |
| Missing reduced-motion / focus-visible | No `prefers-reduced-motion`, removed focus outlines | Accessibility debt with a WCAG mapping (`skill: accessibility-baseline`) |

See `skill: modern-css` and `skill: responsive-accessible-css`.

### 4. Dependency Debt

| Item | Signal | Why it costs |
|------|--------|--------------|
| Unused dependencies | `npx depcheck` / `npx knip` | Install time, audit noise, false CVE surface |
| Duplicated packages | `npm ls <pkg>` showing multiple versions; two React copies | Bundle bloat; duplicate-React context breakage |
| Abandoned packages | Last publish > 2 years, archived repo, open CVE with no fix | Unpatchable risk; blocks framework majors |
| Heavy deps for small use | `moment`, full `lodash`, full `date-fns` for one helper | Direct bundle cost (`skill: bundling-optimization`) |
| Outstanding CVEs | `npm audit` / `pnpm audit` + osv.dev cross-check | Security debt with an SLA |
| Peer-dependency conflicts | `ERESOLVE`, `--legacy-peer-deps` in CI | Upgrades are already blocked |

### 5. Test Debt

| Item | Signal | Why it costs |
|------|--------|--------------|
| Coverage gaps on critical paths | Coverage report vs the auth/checkout/data-mutation paths | Debt is measured on *critical* paths, not on the global % |
| Skipped / focused specs | `it.skip`, `describe.skip`, `xit`, `it.only`, `fit` | Silently unenforced behavior; `.only` can hide a whole suite |
| No component tests for shared primitives | Design-system components with no spec | Every consumer inherits the risk |
| No E2E on the money path | No Playwright/Cypress spec covering the primary user journey | Regressions reach production |
| Flaky specs | Retry config, arbitrary `waitForTimeout`, timing-dependent assertions | Erodes trust in the whole suite |
| Slow suite | Unit suite > ~5 min, CI > ~20 min | Developers stop running it locally |

See `skill: fe-testing` and `skill: testing-principles` (test pyramid and quality gates).

### 6. Framework & Runtime Debt

| Item | Signal | Why it costs |
|------|--------|--------------|
| Majors behind | `npm outdated` on `react`/`vue`/`svelte`/`@angular/core`/`next`/`nuxt` | Each skipped major makes the next migration superlinearly harder |
| EOL Node version | `engines` field / `.nvmrc` vs the Node release schedule | Loses security patches; blocks tooling upgrades |
| Deprecated build tooling | Webpack 4, CRA, Node-Sass, legacy Babel-only pipeline | No modern plugin ecosystem; slow builds |
| Missing modern capability | No RSC/streaming/signals/runes where the version supports them | Not debt by itself — debt when a workaround exists *because* of the gap |
| Browser-target drift | `browserslist` targeting browsers no longer in use | Shipping unnecessary polyfills and transpilation weight |

Check every version claim against `skill: version-feature-matrix`.

## Measurement Table

Run these to attach a number to each category. All are read-only. Record the command beside each count so the measurement is reproducible next quarter.

| Category | Command | Yields |
|----------|---------|--------|
| Legacy code | `grep -rn "extends React.Component\|extends Component" src --include=*.tsx --include=*.jsx` | class-component count + locations |
| Legacy code | `grep -rn "UNSAFE_\|componentWillReceiveProps\|ReactDOM.render" src` | deprecated-API count |
| Legacy code | `grep -rln "@NgModule" src` | NgModule file count |
| Legacy code | `grep -rn '^\$:' src --include=*.svelte` | Svelte 4 reactive-statement count |
| Types | `grep -rn ": any\|as any\|<any>" src --include=*.ts --include=*.tsx` | `any` count + locations |
| Types | `grep -rn "@ts-ignore\|@ts-nocheck\|@ts-expect-error" src` | suppression count |
| Types | `npx tsc --noEmit --strict` | errors that appear once strict is on (the true cost of `strict: false`) |
| CSS | `grep -rc "!important" src --include=*.css --include=*.scss` | `!important` count per file |
| CSS | `grep -rnE "#[0-9a-fA-F]{3,6}\b" src --include=*.css --include=*.scss` | hard-coded color count |
| Deps | `npx depcheck` (or `npx knip`) | unused deps + unused exports |
| Deps | `npm ls --all --depth=99` (or `pnpm why <pkg>`) | duplicate versions |
| Deps | `npm outdated --json` | majors/minors behind per package |
| Deps | `npm audit --json` (or `pnpm audit --json`) | open advisories by severity |
| Tests | `grep -rn "\.skip(\|\.only(\|xit(\|fit(" src tests e2e` | skipped/focused spec count |
| Tests | `npx vitest run --coverage` (or the project's coverage script) | coverage totals per path |
| Framework | `node -v` + `engines` in `package.json` | runtime version gap |
| Bundle impact | `npx vite-bundle-visualizer` / `npx source-map-explorer dist/**/*.js` | per-dependency bundle cost |

Prefer the project's own scripts when they exist. Use single-invocation commands — never `cd`-chain or `&&`-chain.

## Workflow

### Phase 1: Scope & Detect

Resolve the scope, exclude build artifacts, and detect the framework and package manager via `skill: language-detection`. Print the framework, versions, package manager, and file count before scanning.

### Phase 2: Measure (Bash, read-only)

Run the Measurement Table commands for every in-scope category. Capture counts and representative `file:line` locations (three per aggregate finding, plus the total). Mark any category whose scanner was unavailable as **partial**, with the install hint.

### Phase 3: Enrich

For abandoned or CVE-bearing dependencies, confirm maintenance status and advisories via WebSearch/WebFetch (registry metadata, repository archive status, `https://api.osv.dev/v1/query`). For framework majors, look up the migration guide for the *next single major* — never two.

### Phase 4: Quantify Impact

Score each item:

| Dimension | Scale | Reading |
|-----------|-------|---------|
| **User-facing risk** | 1-5 | Can this ship a defect, a security hole, or an accessibility failure to users? |
| **Velocity cost** | 1-5 | How much does this slow a typical change in the area? |
| **Bundle / runtime cost** | 1-5 | Measured KB or measured runtime cost, not a guess |
| **Blast radius** | 1-5 | How many files/routes/teams does the item touch? |
| **Effort** | S / M / L | S ≤ 1 day, M ≤ 1 week, L > 1 week |

`Payback = (risk + velocity + bundle + radius) / effort`. Rank descending.

### Phase 5: Route the Prioritization

Route to `frontend-developer:frontend-architector` (`subagent_type="frontend-developer:frontend-architector"`, model `opus`):

"Read-only frontend tech-debt prioritization for `{scope}` (framework: {framework} {version}, package manager: {pm}). Measured inventory:\n```\n{per-category counts + representative file:line locations + the command that produced each number}\n```\nFor each item: score user-facing risk, velocity cost, bundle/runtime cost, and blast radius (1-5 each) and effort (S/M/L), then rank by payback. Group into Quick Wins (≤1 week), Medium-Term (1-3 months), and Structural (quarter-scale). For framework-major debt, plan **one major at a time** with the migration guide for that single step. State explicitly that each remediation step is gated on `/frontend-developer:build-test` before the next begins. Call out any item where paying the debt is NOT worth it and the right answer is to accept and document it. Do NOT edit any file — this is a read-only analysis."

### Phase 6: Report

Emit the Output Format. Do not begin remediation — hand off the ranked plan.

## Output Format

```markdown
## Frontend Tech Debt Report

**Scope:** {path} ({N} files)
**Framework:** {name} {version} ({M} majors behind latest)
**Package manager:** {npm | pnpm | yarn | bun}
**Categories scanned:** {list} {(partial: <category> — scanner missing)}

### Debt Score
| Category | Items | Headline metric | Trend hook |
|----------|-------|-----------------|------------|
| Legacy code | {n} | {e.g. 23 class components} | `grep -rn "extends Component" src` |
| Types | {n} | {147 `any`, strict off} | `npx tsc --noEmit --strict` |
| CSS | {n} | {89 `!important`} | `grep -rc "!important" src` |
| Dependencies | {n} | {12 unused, 3 CVE, 2 abandoned} | `npx depcheck`; `npm audit` |
| Tests | {n} | {critical path 34% covered, 11 skipped} | coverage script |
| Framework | {n} | {2 majors behind, Node EOL} | `npm outdated` |

### Prioritized Debt Register
| # | Item | Category | Count | Risk | Velocity | Bundle | Radius | Effort | Payback |
|---|------|----------|-------|------|----------|--------|--------|--------|---------|
| 1 | {item} | {cat} | {n} | {1-5} | {1-5} | {1-5} | {1-5} | {S/M/L} | {score} |

### Quick Wins (≤ 1 week)
| Item | Evidence | Fix | Gate |
|------|----------|-----|------|
| {item} | {file:line ×3, count} | {concrete change} | `/frontend-developer:build-test` |

### Medium-Term (1-3 months)
{same table shape}

### Structural (quarter-scale)
| Item | Why it is structural | Phased path | Risk points |
|------|----------------------|-------------|-------------|

### Framework Upgrade Path
| Step | From → To | Breaking changes | Gate |
|------|-----------|------------------|------|
| 1 | {pkg} {vN} → {vN+1} | {summary} | build-test green before step 2 |

### Accepted Debt (not worth paying)
| Item | Why accepting is correct | Revisit when |
|------|-------------------------|--------------|

<!-- --budget only: -->
### Debt Budget
| Parameter | Target |
|-----------|--------|
| Quarterly reduction | {n}% of the register |
| New-code coverage floor | {n}% |
| New `any` / `@ts-ignore` | 0 (CI-enforced) |
| Critical/High CVE SLA | Critical 24h, High 7d |
| Bundle budget | {KB} per route entry |

**Prevention gates:** {lint rules, CI checks, PR conditions that stop the register from regrowing}

<!-- When a scanner was unavailable: -->
### Partial Coverage
- {category}: {tool} unavailable — measured by grep only. Install: {hint}.
```

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Fallback |
|--------------|--------------|----------|
| `node` / `npm` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | grep-only measurement across every category; mark all **partial** |
| `depcheck` / `knip` | `npm install -D knip` (or `npx depcheck`) | detect unused deps by grepping imports for each `package.json` entry |
| `tsc` | `npm install -D typescript` | count `any`/suppressions by grep; skip the strict-mode error count |
| coverage runner (`vitest`/`jest`) | `npm install -D vitest` (or `jest`) | count spec files vs source files as a coarse proxy; mark test debt **partial** |
| bundle analyzer | `npm install -D rollup-plugin-visualizer` (or `npx source-map-explorer`) | estimate bundle cost from package size on the registry; label the number as an estimate |
| `gh` | `brew install gh` then `gh auth login` | skip repository archive-status checks; rely on registry publish dates |

A missing scanner degrades a category to **partial**, never to silence: print the hint, run the grep-level fallback, and label the category in the report. The command only reports a hard stop when there is no `package.json` in scope at all.

## Error Handling

### No web project in scope
```
Error: No package.json found under {scope}.
Suggestion: run from the project root, or pass the app directory explicitly.
```

### Category has zero findings
Not an error. Report the category as clean with the measurement command that proved it — a zero backed by a command is a useful baseline for the next run.

### Debt item is actually a live bug
Not debt. Note it, and route it: `/frontend-developer:review-code` for a correctness pass or `/frontend-developer:debug` for a specific failure. Keep the debt register structural.

### Remediation requested
Out of scope — this command is read-only. Hand the ranked register to `/frontend-developer:fix-refactor`, `/frontend-developer:fix-modernize`, `/frontend-developer:deps`, or `/frontend-developer:gen-tests`, one step at a time, each gated on `/frontend-developer:build-test`.

## See Also

- `skill: severity-matrix` — severity and coverage thresholds behind the scoring.
- `skill: version-feature-matrix` — canonical version gates for every framework-debt claim.
- `skill: language-detection` — framework and package-manager detection used in Phase 1.
- `skill: ts-typing` / `skill: ts-config` — `any`-elimination patterns and the strict-flag ladder.
- `skill: modern-css` / `skill: responsive-accessible-css` — the CSS patterns that replace the debt items above.
- `skill: bundling-optimization` — measuring and cutting the bundle cost of dependency debt.
- `skill: fe-testing` / `skill: testing-principles` — test-pyramid targets behind the test-debt category.
- `/frontend-developer:arch-review` — structural boundary violations that this register quantifies.
- `/frontend-developer:fix-modernize` — execute legacy-pattern and framework-major migrations.
- `/frontend-developer:fix-refactor` — execute the clean-code restructuring this register recommends.
- `/frontend-developer:deps` — audit and upgrade the dependency-debt items one at a time.
- `/frontend-developer:gen-tests` — close the coverage gaps on the critical paths named here.
- `/frontend-developer:build-test` — the gate every remediation step must pass before the next begins.
