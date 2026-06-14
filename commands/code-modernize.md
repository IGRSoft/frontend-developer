---
description: Migrate a web codebase to a modern framework idiom (React class->hooks, Vue 2->3 Composition API, Angular NgModule->standalone) one component or class at a time, gating each migration on a green build and test run
argument-hint: [path (default .)] --target react-hooks|vue3|angular-standalone|svelte5 [--dry-run]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
estimated-cost:
  band: high
  min-tokens: 4000
  max-tokens: 30000
  model-distribution:
    haiku: 30%
    sonnet: 55%
    opus: 15%
---

# Code Modernize
<!-- Updated: June 2026 -->

Move a web codebase to a modern framework idiom incrementally and safely. Modernization is sequenced as a ledger of discrete *migration units* (one component, one class, one module at a time), and every unit is verified by a full build + test run before its commit and before the next unit begins. Mechanical rewrites are routed to `frontend-developer:fe-code-fixer`; semantic migrations that need judgment go to the owning framework agent (`react`/`vue`/`svelte`/`angular`-developer).

[Extended thinking: A framework migration is dozens of independent component rewrites with sharply different risk. Doing them all at once produces an un-reviewable diff and a build that is either green or broken with no way to bisect which component broke it. This command instead builds an ordered ledger at `.context/.modernize/plan.md` — leaf components first (fewest dependents), shared components last — then walks it one unit at a time, running `/frontend-developer:build-test` after each and committing one unit per commit. Crucially, it never co-mingles units: a React class component becomes a function component with hooks in its own commit, verified green, before the next. `--dry-run` produces the ledger and stops, so you can review the plan before any edit. Mechanical-vs-semantic routing keeps cheap deterministic rewrites (lifecycle-method → `useEffect` skeleton, `Vue.extend` → `defineComponent`) on the code-fixer and reserves the framework developer for the rewrites that change shape (state colocated into hooks, `this.$emit` → `defineEmits`, RxJS-in-template → signals).]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **One component/class at a time.** Each ledger row is exactly one component, class, or module. Never batch unrelated units into one diff. Migrate leaf units (fewest dependents) before shared ones so each green checkpoint is small and bisectable.
2. **One unit per commit.** Each unit is applied, verified, and committed on its own. The commit subject names the unit (e.g. `refactor: convert UserCard class component to hooks`).
3. **Verify after every unit (BINDING gate).** After applying a unit, run `/frontend-developer:build-test` (type-check + build + tests). If it is not green, the unit is NOT committed — revert or fix before moving on. A red build halts the run; report it and stop.
4. **`--dry-run` produces the ledger only.** In `--dry-run`, write `.context/.modernize/plan.md` and stop. Make ZERO source edits and ZERO commits. This is the review-the-plan mode.
5. **Mechanical vs semantic routing.** Pure mechanical rewrites (codemod-able skeletons, import swaps, decorator-to-standalone metadata) route to `frontend-developer:fe-code-fixer`. Rewrites needing judgment (state/effect colocation, lifecycle-to-hooks semantics, reactivity migration) route to the owning framework agent. Never hand a semantic migration to the code-fixer.
6. **Prefer the official codemod, then verify.** Where the framework ships a codemod (`npx types-react-codemod`, Vue's migration build, `ng update`/`ng generate @angular/core:standalone`, `npx svelte-migrate`), run it as the mechanical first pass, then hand the residue to the framework agent. Gate every adopted modern feature on the version marker — see `skill: version-feature-matrix`. If the toolchain cannot reach the target framework version, report the gap and stop.
7. **Single-command Bash invocations.** Use each tool's own path/recursion flags. Never `cd`-chain or `&&`-chain — scoped Bash patterns do not match compound commands.
8. **Tool-missing never hard-fails.** If a codemod or the build/test toolchain is absent, print the install hint, skip that unit (or fall back to a hand migration), and continue. Report what was skipped.
9. **Commands route, they do not orchestrate.** This command names `frontend-developer:fe-code-fixer` and the framework agents so Claude routes the work; it does not call `Task`.
10. **Never enter plan mode.** This command IS the procedure — execute it (or, with `--dry-run`, produce the ledger and stop).

## Usage

```bash
# Preview the React class->hooks migration ledger without touching source
/frontend-developer:code-modernize src/ --target react-hooks --dry-run

# Convert Vue 2 Options API components to Vue 3 Composition API, one at a time
/frontend-developer:code-modernize src/components --target vue3

# Migrate Angular NgModule app to standalone components
/frontend-developer:code-modernize . --target angular-standalone

# Migrate Svelte 4 reactive statements to Svelte 5 runes
/frontend-developer:code-modernize src/ --target svelte5
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `path` | `.` | Directory or file to modernize. Inventory and detection are rooted here. |
| `--target react-hooks\|vue3\|angular-standalone\|svelte5` | required | The destination idiom. The command detects the current idiom and migrates only the units not already there. |
| `--dry-run` | off | Produce `.context/.modernize/plan.md` (the migration ledger) and stop. No edits, no commits. |

`--target` is required — there is no default, because the right destination depends on the project's framework and version floor.

## Inventory

Before building the ledger, establish the *current* idiom and the dependency order so units migrate leaf-first.

| Target | Detect current idiom by | Migration unit |
|--------|-------------------------|----------------|
| `react-hooks` | `class X extends React.Component` / `Component`; lifecycle methods (`componentDidMount`, ...) | one class component → one function component with hooks |
| `vue3` | Options API (`export default { data(), methods, ... }`), `Vue.extend`, Vue 2 in `package.json` | one Options-API component → `<script setup>` Composition API |
| `angular-standalone` | `@NgModule`, `declarations: [...]`, non-standalone components | one NgModule's components → `standalone: true` + explicit `imports` |
| `svelte5` | `$:` reactive statements, `export let` props, Svelte 4 in `package.json` | one component → `$state`/`$derived`/`$props` runes |

The marker → framework map is canonical in `skill: language-detection`. Use `skill: version-feature-matrix` to confirm the framework version supports the target idiom (React 19 for the modern `use`/RSC patterns, Vue 3, Angular 18+ for standalone-by-default + signals, Svelte 5 for runes). Build the dependency graph (who imports whom) and order the ledger leaf-first.

> Requires the modern idiom (React 19+ / Vue 3+ / Angular 18+ / Svelte 5+). Fallback: keep the legacy idiom on the pre-version toolchain; do not migrate code that the installed framework version cannot run. Canonical: _shared/version-feature-matrix.md

## The Migration Ledger

The inventory's output is `.context/.modernize/plan.md`: an ordered checklist of migration units, leaf-first. Template:

```markdown
# Modernization Ledger
Target: {react-hooks|vue3|angular-standalone|svelte5} | Framework version: {detected} | Path: {path}
Version gate: {framework + min version from version-feature-matrix} — {PASS | GAP: ...}

## Units (leaf-first)
| # | Unit (component/class/module) | Kind | Owner | Status |
|---|-------------------------------|------|-------|--------|
| 1 | components/Badge.tsx (leaf) | mechanical-then-semantic | fe-code-fixer → react-developer | pending |
| 2 | components/UserCard.tsx | semantic | react-developer | pending |
| 3 | components/Dashboard.tsx (shared) | semantic | react-developer | pending |
```

Status transitions per row: `pending → applied → verified → committed` (or `reverted` on a red build). The ledger is the source of truth across resumes — read it, do not rely on context-window memory.

## Per-Target Playbooks

Each playbook leads with the official codemod (mechanical), then the semantic residue. Every feature claim carries a version marker and a fallback from `skill: version-feature-matrix`.

### `--target react-hooks`

Version gate: React 16.8+ for hooks; React 19 for `use`/Actions patterns (`skill: version-feature-matrix`).

| Order | Step | Kind | Notes |
|-------|------|------|-------|
| 1 | `npx types-react-codemod` + manual class→function skeleton | mechanical | Convert the class shell, props typing. Route to `fe-code-fixer`. |
| 2 | lifecycle → `useEffect`/`useLayoutEffect` | semantic | Map `componentDidMount`/`DidUpdate`/`WillUnmount` to effects with correct deps + cleanup — semantics, not transliteration. Route to `react-developer`. |
| 3 | `this.state`/`setState` → `useState`/`useReducer` | semantic | Colocate related state; choose `useReducer` for coupled transitions. |
| 4 | instance methods → `useCallback`/plain functions | semantic | Stabilize identities only where a dependent needs it. |

### `--target vue3`

Version gate: Vue 3.x; the Vue 2 → 3 migration build helps interop (`skill: version-feature-matrix`).

| Order | Step | Kind | Notes |
|-------|------|------|-------|
| 1 | `Vue.extend`/`defineComponent` shell + import swaps | mechanical | Move to `<script setup>` skeleton. Route to `fe-code-fixer`. |
| 2 | `data()`/`computed`/`methods` → `ref`/`reactive`/`computed` | semantic | Choose `ref` vs `reactive`; preserve reactivity (no destructuring loss). Route to `vue-developer`. |
| 3 | `this.$emit`/`props` → `defineEmits`/`defineProps` (typed) | semantic | Typed emits/props; remove `this`. |
| 4 | lifecycle hooks → `onMounted`/`onUnmounted`/... | semantic | Map Options lifecycle to composition lifecycle. |

### `--target angular-standalone`

Version gate: Angular 15+ for standalone, 18+ for standalone-by-default + signals (`skill: version-feature-matrix`).

| Order | Step | Kind | Notes |
|-------|------|------|-------|
| 1 | `ng generate @angular/core:standalone` codemod | mechanical | Official migration: adds `standalone: true`, hoists `imports`, removes `declarations`. Route to `fe-code-fixer`. |
| 2 | NgModule removal + `bootstrapApplication` | semantic | Replace `AppModule` bootstrap; wire providers via `ApplicationConfig`. Route to `angular-developer`. |
| 3 | `@Input`/`@Output` → input/output signals (18+) | semantic | Optional, version-gated; convert to `input()`/`output()` signals where the version allows. |
| 4 | RxJS-in-template → signals (where apt) | semantic | Convert `async`-pipe state to `signal`/`computed` per `skill: angular-signals`. |

### `--target svelte5`

Version gate: Svelte 5+ for runes (`skill: version-feature-matrix`).

| Order | Step | Kind | Notes |
|-------|------|------|-------|
| 1 | `npx svelte-migrate svelte-5` codemod | mechanical | Official migration of `export let` → `$props`, `$:` → `$derived`/`$effect` skeleton. Route to `fe-code-fixer`. |
| 2 | `$:` reactive statements → `$derived` vs `$effect` | semantic | Pure derivations → `$derived`; side effects → `$effect`. Do not use `$effect` for derived state. Route to `svelte-developer`. |
| 3 | stores → `$state`/runes where local | semantic | Migrate component-local stores to `$state`; keep cross-component stores. |

## Workflow

### Phase 1: Inventory & Ledger (Bash + Read)

1. Confirm `path` exists; if not, emit the "path not found" message and stop.
2. Validate `--target`; if missing/invalid, emit the "missing target" message and stop.
3. Detect the current idiom and framework version (Inventory). Resolve the version gate via `skill: version-feature-matrix`. If GAP (framework too old for the target idiom), report and stop (Rule 6).
4. If everything is already at the target idiom, report "already modernized" and stop.
5. Build the dependency graph and write `.context/.modernize/plan.md`, leaf-first, all rows `pending`.
6. **If `--dry-run`: stop here.** Report the ledger path and planned units. No edits, no commits.

### Phase 2: Apply Units In Order (routed)

Walk the ledger top-down, one row at a time. For each `pending` row:

1. **Mark `applied`** as you start it.
2. **Route by kind:**
   - **mechanical** → route to `frontend-developer:fe-code-fixer`:
     "Apply ONLY the mechanical first pass for migrating **{unit}** to {target}. Run the official codemod where available ({codemod cmd}) and convert the component/class shell + import swaps. Do NOT migrate semantics (effects/state/reactivity) — leave a clear skeleton for the framework agent. Report every file touched. Do not run tests."
   - **semantic** → route to the owning framework agent (`react`/`vue`/`svelte`/`angular`-developer):
     "Migrate ONLY **{unit}** to {target} ({from-idiom}→{to-idiom}). {e.g. Map lifecycle to useEffect with correct deps+cleanup / colocate state into useState/useReducer / convert Options data+computed to ref/reactive preserving reactivity / convert $: statements to $derived vs $effect}. Gate any version-specific feature on the framework version. Do not touch other units. Return the diff and the rationale for each semantic choice."
3. **Mark `verified`** only after the build+test gate (Phase 3) is green for this unit.

### Phase 3: Verify After Every Unit (per unit, BINDING)

1. Run `/frontend-developer:build-test {path}` (type-check + build + tests). Tee output to `.context/logs/`.
2. **Green** → mark the row `verified`, proceed to commit.
3. **Red** → the unit did NOT pass:
   - Mechanical unit: revert it and report — a codemod that breaks the build is a real signal.
   - Semantic unit: hand the failing build/test excerpt back to the same framework agent for one bounded corrective pass, then re-gate.
   - If still red, **halt the run**, mark the row `reverted`, and report (Rule 3). Do not proceed to later units on a broken build.

### Phase 4: Commit One Unit Per Migration (per unit)

1. Stage only the files that unit touched.
2. Commit with a conventional subject naming the unit, e.g.:
   - `refactor: convert UserCard class component to hooks`
   - `refactor: migrate ProductList to Vue 3 Composition API`
   - `refactor: make Dashboard component standalone`
   - `refactor: convert Counter reactive statements to Svelte 5 runes`
3. Mark the row `committed`. Move to the next `pending` row.

> Per the user's git conventions: no `--no-verify`, no AI-attribution footers, follow the repo commit format. If mid-feature on a protected branch, branch first.

### Phase 5: Report

Emit the Output Format. Summarize units applied/verified/committed/reverted/skipped, and the ledger path for the audit trail.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint |
|--------------|--------------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) |
| `types-react-codemod` | `npx types-react-codemod@latest` (resolves on demand) |
| Vue migration build | `npm install -D @vue/compat` (verify against your toolchain) |
| `@angular/cli` (`ng update`/`ng generate`) | `npm install -D @angular/cli` then `npx ng ...` |
| `svelte-migrate` | `npx svelte-migrate@latest svelte-5` (resolves on demand) |

When a codemod is unavailable, print the hint and fall back to a hand migration via the framework agent (skip the mechanical pass, do the whole unit semantically) — note the fallback. Never hard-fail on a single missing tool: print the hint, skip/fallback the unit, continue, and report.

## Output Format

```markdown
## Code Modernization Report

**Target idiom:** {react-hooks | vue3 | angular-standalone | svelte5}
**Framework version:** {detected}
**Path:** {path}
**Mode:** dry-run | apply
**Ledger:** .context/.modernize/plan.md
**Version gate:** {framework + min version} — {PASS | GAP}

<!-- dry-run: stop after the ledger -->
### Planned Migration Units ({count})
{rendered ledger table, all rows pending}

<!-- apply mode -->
### Units
| # | Unit | Kind | Owner | Status | Commit |
|---|------|------|-------|--------|--------|
| 1 | Badge.tsx | mechanical→semantic | fe-code-fixer→react-developer | committed | {sha/subject} |
| 2 | UserCard.tsx | semantic | react-developer | committed | {sha/subject} |
| 3 | Dashboard.tsx | semantic | react-developer | reverted | (build red) |

**Result:** COMPLETE / PARTIAL / HALTED ({reason})
- Units committed: {n} | verified-not-committed: {n} | reverted: {n} | skipped (tool missing): {n}

<!-- on a halt -->
### Halt
- **Unit:** {unit}
- **Build/test:** RED — {one-line first error; full log in .context/logs/...}
- **Action:** row marked reverted; later units not attempted.

<!-- on skipped units/tools -->
### Skipped
- {unit}: {missing codemod — hand-migrated via {agent} | skipped} — hint above.
```

## Error Handling

### Path not found
```
Error: Path not found: {path}
Suggestion: Pass a directory or file that exists, e.g. /frontend-developer:code-modernize src/ --target react-hooks --dry-run
```

### Missing or invalid --target
```
Error: --target is required and must be one of: react-hooks, vue3, angular-standalone, svelte5.
Suggestion: /frontend-developer:code-modernize . --target react-hooks --dry-run
```

### Version gate failure
```
Error: The installed framework version cannot run the {target} idiom.
{framework} {detected} < required {min} (see skill: version-feature-matrix).
Suggestion: upgrade the framework first (e.g. /frontend-developer:deps-audit upgrade {framework}), then modernize.
```
Stop — do not write code the framework version cannot run (Rule 6).

### Already modernized
```
Note: {path} is already on the {target} idiom (detected: {current}).
Nothing to modernize. Suggestion: target a different idiom or path.
```

### Build red after a unit
Halt the run (Rule 3). Mark the row `reverted`, report the first error and log path, and do NOT attempt later units. For a semantic unit, one bounded corrective cycle with the owning agent is allowed before the halt.

### Tool missing
Print the install hint, fall back to a hand migration (skip the mechanical pass), continue, and note the fallback. Only when no unit can be migrated does the command report HALTED with aggregated hints.

## See Also

- `skill: version-feature-matrix` — canonical framework-version → feature/fallback tables (gate every adopted idiom here).
- `skill: language-detection` — marker → framework → agent routing (keep per-file detection in sync).
- `skill: modern-react`, `skill: modern-css`, `skill: svelte-runes`, `skill: angular-signals` — the target-idiom playbooks the framework agents follow.
- `/frontend-developer:build-test` — the build + test gate run after every migration unit.
- `/frontend-developer:lint-fix` — the shallow mechanical pass; this command sequences codemods plus semantic migrations across units.
- `/frontend-developer:code-review` — review the modernized diff for behavioral drift once the ledger is complete.
- `/frontend-developer:deps-audit` — upgrade the framework version first when the version gate is a GAP.
