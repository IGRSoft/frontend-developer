---
description: Refactor web UI for clean code and SOLID — architector plans, fe-code-fixer applies; --extract to a shared package
argument-hint: [path or component (default: working changes)] [--extract <package-name>] [--dry-run]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
estimated-cost:
  min-tokens: 4000
  max-tokens: 26000
  model-distribution:
    haiku: 15%
    sonnet: 65%
    opus: 20%
---

# Refactor
<!-- Updated: June 2026 -->

Restructure web UI code for clean-code and SOLID quality **without changing behavior**. The work is split across exactly two agents: `frontend-developer:frontend-architector` **plans** the refactor (it never edits), and `frontend-developer:fe-code-fixer` **applies** it one unit at a time under a minimal-diff gate. Every applied unit is verified by `/frontend-developer:build-test` before the next begins. `--extract` switches to modularization mode: pull the target into a shared workspace package instead of restructuring it in place.

[Extended thinking: A front-end refactor fails in two distinct ways — a bad plan (splitting a component along the wrong seam, hoisting state into a store that did not need one) and a bad application (a "while I'm here" rewrite that silently changes render behavior). Separating the two roles fixes both: the architector owns the seams, the dependency direction, and the ordering, and produces a written ledger; the fixer owns nothing but the mechanical application of one ledger row, with no license to redesign. Because a refactor must be behavior-preserving, the ledger is walked leaf-first (fewest dependents first) and each row is gated on a green build + test run, so a regression is always attributable to a single small diff. `--dry-run` stops after the plan, which is the right default posture for any refactor touching shared components.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Two agents, two roles — never merged.** `frontend-developer:frontend-architector` PLANS: it reads, decides the seams, and writes the ledger. It MUST NOT edit source. `frontend-developer:fe-code-fixer` APPLIES: it edits only what the current ledger row states. It MUST NOT re-plan, re-scope, or invent additional refactors. If the fixer believes the plan is wrong, it stops and reports back to the architector — it does not improvise.
2. **Behavior-preserving.** A refactor changes structure, not observable behavior. No new features, no bug fixes, no API changes bundled in. If a bug is discovered, record it in the report and leave it — fix it separately with `/frontend-developer:fix-quick` or `/frontend-developer:review-code --fix`.
3. **One unit at a time, leaf-first.** Each ledger row is one component, hook/composable, module, or stylesheet. Order rows so units with the fewest dependents go first. Never batch unrelated units into one diff.
4. **Verify after every unit (BINDING gate).** After each applied unit run `/frontend-developer:build-test` (type-check + build + tests). Red → the unit is NOT kept: hand the failing excerpt back to the fixer for one bounded corrective pass, then re-gate; still red → revert that unit, halt the run, and report. Do not walk further rows on a broken build.
5. **`--dry-run` produces the plan only.** Write `.context/.refactor/plan.md` and stop. ZERO source edits, ZERO commits.
6. **Minimal-diff gate on every application.** The fixer changes only the lines the row requires. No reformatting untouched code, no drive-by renames, no import reordering beyond what the move requires.
7. **Tool-missing never hard-fails.** If a linter, formatter, or codemod is not installed, print the install hint, skip that step, and continue with a note. Never abort the whole refactor over one missing optional tool.
8. **Commands route, they do not orchestrate.** This command names which `frontend-developer:*` agent owns each role so Claude routes the work; it does not call `Task` itself.
9. **Never enter plan mode.** This command IS the procedure — execute it (or, with `--dry-run`, produce the ledger and stop).

## Usage

```bash
# Plan and apply a refactor across a directory
/frontend-developer:fix-refactor src/components

# Refactor a single god-component
/frontend-developer:fix-refactor src/components/Dashboard.tsx

# Review the plan before any edit lands
/frontend-developer:fix-refactor src/features/checkout --dry-run

# Refactor the working changes (default scope)
/frontend-developer:fix-refactor

# Extract a cohesive unit into a shared workspace package
/frontend-developer:fix-refactor src/lib/date-format --extract @acme/date-format
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `path` | working changes | File, directory, or component to refactor. With no argument, scope is the staged + unstaged tracked changes (`git diff --name-only HEAD`). |
| `--extract <package-name>` | off | Modularization mode: pull the target into a shared workspace package (local library) instead of restructuring in place. See Extract-to-Package Mode. |
| `--dry-run` | off | Produce `.context/.refactor/plan.md` and stop. No edits, no commits. |

## Refactor Targets

The architector looks for these seams. Each row is a candidate ledger unit.

| Target | Smell to detect | Refactor |
|--------|-----------------|----------|
| God component | One component owning fetching + transformation + layout + interaction; hundreds of lines; many unrelated state slots | Split into a container (data/state) plus presentational children; move derivations out of the render body |
| Prop drilling | The same prop threaded through 3+ intermediate components that never read it | Hoist into a context provider (React `createContext`, Vue `provide`/`inject`, Svelte context, Angular DI) or a store slice — see `skill: react-state` / `skill: vue-state` |
| Repeated stateful logic | The same `useState`/`useEffect` (or `ref`/`watch`, or `$state`/`$effect`) cluster duplicated across components | Extract a custom hook (`useX`) / composable (`useX`) / rune module with an explicit input→output contract |
| Duplicated CSS | The same declaration blocks, magic numbers, or one-off color literals repeated across stylesheets/modules | De-duplicate into design tokens, custom properties, or a shared utility/component class — see `skill: modern-css`, `skill: tailwind-design-system` |
| Loose TypeScript | `any` leaks, unsafe casts, optional-everything props, string unions that should be discriminated | Tighten to discriminated unions, exhaustive switches, and precise public prop types — see `skill: ts-typing` |
| SOLID violations | Component with several unrelated reasons to change (SRP); a `variant` prop switch that grows per feature (OCP); a child that ignores its parent's contract (LSP); a fat prop interface consumers only half-use (ISP); a UI component importing a concrete API client (DIP) | Split by reason-to-change; replace variant switches with composition/slots; narrow prop interfaces; inject the data dependency instead of importing it |
| Dead / unreachable UI | Unrendered branches, unused exports, orphaned styles and assets | Delete, once a grep confirms no dynamic reference |

Severity and ordering for the ledger follow `skill: severity-matrix` (P0-P3 plus the effort/impact ranking). Framework-specific correctness constraints while restructuring come from `skill: modern-react`, `skill: vue-composition`, `skill: svelte-runes`, and `skill: angular-signals`; the marker → framework → agent map is canonical in `skill: language-detection`.

## The Two-Agent Split

| Role | Agent | Does | Must NOT |
|------|-------|------|----------|
| Planner | `frontend-developer:frontend-architector` | Reads the scope, detects the framework, picks the seams, orders units leaf-first, writes `.context/.refactor/plan.md` | Edit any source file; apply a fix "since it's small" |
| Applier | `frontend-developer:fe-code-fixer` | Applies exactly one ledger row per pass, minimal diff, reports each file touched | Re-plan, expand scope, refactor a neighbouring unit, or fix unrelated bugs |

Hand-off brief for the planner:

> "Plan (do NOT edit) a behavior-preserving refactor of: {file_list} (framework: {framework}). Identify the seams from the Refactor Targets table — god components, prop drilling, extractable hooks/composables, duplicated CSS, loose TypeScript, SOLID violations. For each seam produce one ledger unit: `{id, target file(s), smell, refactor, kind, dependents, risk, done-criteria}`. Order the units leaf-first (fewest dependents first). Flag anything that cannot be done without changing behavior — those are excluded, not attempted. Write the ledger to `.context/.refactor/plan.md`."

Hand-off brief for the applier (one per row):

> "Apply ONLY ledger row {id} from `.context/.refactor/plan.md`: {refactor}. Minimal-diff gate — change only what this row requires; do not reformat untouched code, do not refactor neighbouring units, do not fix unrelated bugs. Behavior must be preserved exactly. Report each change as `{file, line, what, why}`. If the row cannot be applied as written, stop and report the conflict instead of improvising."

## The Refactor Ledger

`.context/.refactor/plan.md` is the source of truth across resumes — read it, do not rely on context-window memory.

```markdown
# Refactor Ledger
Scope: {path} | Framework: {detected} | Mode: in-place | extract

## Units (leaf-first)
| # | Unit | Smell | Refactor | Risk | Status |
|---|------|-------|----------|------|--------|
| 1 | components/Badge.tsx | duplicated CSS | de-dupe to tokens | low | pending |
| 2 | hooks/useCart.ts (new) | repeated stateful logic in 3 components | extract custom hook | med | pending |
| 3 | components/Dashboard.tsx | god component (410 lines) | split container/presentational | high | pending |
```

Status transitions per row: `pending → applied → verified → committed` (or `reverted` on a red gate).

## Workflow

### Phase 1: Plan (frontend-architector, read-only)

1. Resolve the scope: explicit `path` if given, otherwise the working changes. Print the concrete file list. Exclude `dist/`, `build/`, `.next/`, `.svelte-kit/`, `node_modules/`, `coverage/`.
2. Detect the framework(s) present via `skill: language-detection`.
3. Route the planner brief to `frontend-developer:frontend-architector`. It returns the ledger and writes `.context/.refactor/plan.md`.
4. If no seam is worth refactoring, say so plainly and stop — do not manufacture units.
5. **If `--dry-run`: stop here.** Report the ledger path and the planned units.

### Phase 2: Apply (fe-code-fixer, one row at a time)

Walk the ledger top-down. For each `pending` row:

1. Mark it `applied` as you start.
2. Route the applier brief for that row to `frontend-developer:fe-code-fixer`.
3. Proceed to the gate. Never start the next row before the current one is `verified`.

### Phase 3: Gate After Every Unit (BINDING)

1. Run `/frontend-developer:build-test {path}` — type-check + build + tests. Tee output to `.context/logs/`.
2. **Green** → mark the row `verified` and continue.
3. **Red** → hand the first error plus ~10 lines of context back to `frontend-developer:fe-code-fixer` for one bounded corrective pass, then re-gate. Still red → revert the unit, mark it `reverted`, **halt the run**, and report (Rule 4).

### Phase 4: Commit One Unit Per Refactor

1. Stage only the files that unit touched.
2. Commit with a conventional subject naming the unit, e.g. `refactor: split Dashboard into container and presentational components`.
3. Mark the row `committed` and move to the next `pending` row.

> Per the user's git conventions: no `--no-verify`, no AI-attribution footers, follow the repo commit format. If on a protected branch, branch first.

### Phase 5: Report

Emit the Output Format: units committed / verified / reverted / skipped, plus any behavior-changing issue found and deliberately left alone (Rule 2).

## Extract-to-Package Mode (`--extract`)

With `--extract <package-name>`, the target is pulled into a **self-contained shared workspace package** (a local library in the monorepo) with a clean public API, tests, and a README — instead of being restructured in place. The two-agent split still holds: the architector defines the package boundary and public surface; the fixer (with the owning framework agent for anything semantic) performs the move.

### Pre-Extraction Checklist

Extract only when every row holds:

| Check | Requirement |
|-------|-------------|
| Cohesion | The code is a self-contained, logical unit (one reason to change) |
| Dependencies | Every import is identifiable and classifiable (runtime / peer / dev) |
| Interface | A small public API can be stated as an explicit `exports` surface |
| Replacement | The original call sites can be cleanly replaced with a package import |
| Reuse value | At least one other consumer exists or is concretely planned |
| No app coupling | It does not reach into app-specific routing, global stores, or env |

**When NOT to extract:** single-use code with no second consumer; anything tightly coupled to app state or routing; anything requiring internal implementation details to be made public; or a split that adds a workspace boundary without a clear payoff.

### Phase E1: Package Boundary (frontend-architector)

> "Plan (do NOT edit) the extraction of {target} into a shared workspace package named `{package-name}`. Define: the exact file set to move; the public API surface (named exports, types, CSS entry points); dependency classification (runtime vs `peerDependencies` for the framework and React/Vue/Svelte/Angular runtimes); whether the package is framework-agnostic or framework-bound; the build target (ESM-only vs dual, `types` entry); and the call sites that must be rewritten to import it. Confirm the Pre-Extraction Checklist row by row and state any row that fails."

### Phase E2: Move and Wire (fe-code-fixer + framework agent)

1. Create the package directory (`packages/{name}/` by convention) with its own `package.json`: `name`, `version`, `type: "module"`, an `exports` map (plus `types`), and `sideEffects` for CSS.
2. Register it with the workspace: `workspaces` in the root `package.json` (npm/Yarn/Bun) or `packages:` in `pnpm-workspace.yaml`.
3. Move the files. Framework runtimes (`react`, `vue`, `svelte`, `@angular/core`) go in `peerDependencies`, never `dependencies` — duplicate runtimes break hooks/reactivity.
4. Mark the public surface explicitly via the `exports` map; everything not exported is internal.
5. Add tests inside the package covering every public export — route authoring to `frontend-developer:fe-test-generator` (see `skill: fe-testing`).
6. Rewrite the original call sites to import `{package-name}`; delete the moved code from its original location.
7. Update `tsconfig` path mappings / project references and the bundler alias so the consumer resolves the workspace package.
8. Add a package README: overview, install, quick start, API reference, and a migration note for the old import path.

### Extraction Verification Checklist

- [ ] The package builds standalone
- [ ] The package's own tests pass
- [ ] The consuming app builds and its tests pass (`/frontend-developer:build-test`)
- [ ] No duplicate copy of the moved code remains
- [ ] Framework runtimes are peer dependencies, not direct dependencies
- [ ] The `exports` map exposes only the intended public surface
- [ ] The README covers overview / install / quick start / API / migration

**Extraction best practices:** one responsibility per package; minimal dependencies; avoid generic names (`utils`, `helpers`, `common`); design the API from the consumer's side; start minimal and widen later; semantic versioning from the first release.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect |
|--------------|--------------|--------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | no build/test gate — the refactor cannot be verified; stop and report rather than applying blind |
| `eslint` | `npm install -D eslint` (or `npx eslint` resolves it) | plan runs without lint signal — note reduced depth |
| `prettier` | `npm install -D prettier` | skip the formatting pass; the minimal-diff gate makes it optional anyway |
| `jscodeshift` (bulk renames) | `npx jscodeshift` (resolves on demand) | fall back to per-file edits by the fixer |
| `knip` / `ts-prune` (dead-code detection) | `npm install -D knip` | dead-code units are found by grep only — note reduced confidence |
| `madge` (dependency graph) | `npm install -D madge` | derive the leaf-first order from imports by hand |

Tool-missing is always skip-with-note, never a hard failure — except the build/test gate itself (Rule 4): a refactor that cannot be verified is not applied.

## Output Format

```markdown
## Refactor Report

**Scope:** {path or "working changes"}
**Framework:** {detected}
**Mode:** in-place | extract ({package-name})
**Ledger:** .context/.refactor/plan.md
**Planner:** frontend-developer:frontend-architector | **Applier:** frontend-developer:fe-code-fixer

### Units
| # | Unit | Refactor | Status | Gate | Commit |
|---|------|----------|--------|------|--------|
| 1 | Badge.tsx | de-dupe CSS to tokens | committed | ✅ | {subject} |
| 2 | useCart.ts | extract custom hook | committed | ✅ | {subject} |
| 3 | Dashboard.tsx | split container/presentational | reverted | ❌ | — |

**Result:** COMPLETE / PARTIAL / HALTED ({reason})
- committed: {n} | verified-not-committed: {n} | reverted: {n} | skipped: {n}

### Behavior-Preserving Confirmation
- Public props/API changed: none | {list}
- Tests: {N passed} (unchanged assertions) — no test was weakened to make a unit pass.

<!-- extract mode only -->
### Package
- **Name / path:** {package-name} → packages/{name}
- **Public exports:** {list}
- **Peer deps:** {framework runtimes}
- **Call sites rewritten:** {n}

<!-- when a unit was left alone -->
### Deferred (behavior-changing — not attempted)
- {unit}: {why it would change behavior; suggested follow-up command}

<!-- when a tool was missing -->
### Reduced-Depth Notes
- {step}: {tool} unavailable — install: {hint}.
```

## Error Handling

### Path not found
```
Error: Path not found: {path}
Suggestion: Pass a file or directory that exists, e.g. /frontend-developer:fix-refactor src/components --dry-run
```

### Nothing to refactor
```
Note: No refactor-worthy seam found in {scope}.
Suggestion: Run /frontend-developer:analyze-tech-debt for a broader debt inventory, or name a specific component.
```
Do not invent units to fill the ledger (Rule 1 planner discipline).

### No test coverage on the target
```
Warning: {unit} has no test covering its current behavior.
A refactor without a behavior test is unverifiable beyond type-check + build.
Suggestion: run /frontend-developer:gen-tests {path} first, then re-run this command.
```
Proceed only for low-risk units; halt on `high` risk rows until coverage exists.

### Gate red after a unit
Halt the run (Rule 4). Revert the unit, mark it `reverted`, report the first error and the log path, and do not attempt later rows.

### Extraction checklist fails
```
Error: {package-name} fails the pre-extraction checklist: {failing row(s)}.
Suggestion: refactor in place first (drop --extract), or resolve the coupling and re-run.
```

## See Also

- `skill: severity-matrix` — P0-P3 severity and the effort/impact ranking used to order the ledger.
- `skill: language-detection` — canonical marker → framework → agent routing.
- `skill: testing-principles` — the coverage floor a behavior-preserving refactor needs.
- `skill: ts-typing` — the typing tightening targets.
- `skill: modern-css`, `skill: tailwind-design-system` — CSS de-duplication into tokens.
- `skill: react-state`, `skill: vue-state` — where hoisted state belongs when prop drilling is removed.
- `/frontend-developer:build-test` — the binding gate run after every unit.
- `/frontend-developer:gen-tests` — establish behavior coverage before refactoring an untested unit.
- `/frontend-developer:review-code` — review the refactored diff for behavioral drift once the ledger is complete.
- `/frontend-developer:analyze-tech-debt` — quantify and prioritize debt before choosing what to refactor.
- `/frontend-developer:fix-modernize` — when the goal is a framework idiom migration, not a structural clean-up.
- `/frontend-developer:fix-quick` — mechanical lint/format fixes; run first to clear noise from the diff.
