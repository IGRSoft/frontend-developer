# Changelog

All notable changes to the frontend-developer plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.3.3] — 2026-08-15

### Fixed

- **`CORPFLOW.md` now states the build/test contract, and no longer contradicts it.** The seam
  file said nothing about how to build or test, so a sibling agent denied by corpflow's
  test-execution gate had no sanctioned next step. In a measured four-platform run two streams
  were denied, both invented `--build-only` (not a real flag on any plugin's command, so it
  classifies as a full test run and is denied again), and both then fell back to invoking the
  toolchain directly — which `agents/developer.md` forbids. The file now says: build and test
  only through this plugin's `build-test` command, pass `--no-test` when you only need to
  compile, and if the gate still denies you, record `requests_test_evidence` or return
  `verdict: blocked` — never reach for the toolchain.

- **Added a worktree-isolation section.** A sibling dispatched for DV runs in an isolated
  worktree, and the file previously said only "Write to `.context/`. Nothing else in the
  repository is yours to create" — which is wrong for DV, and silent on the tree check. Two
  agents in identical situations resolved it differently: one blocked correctly, one entered a
  different worktree and relocated its tree. The file now states that a resolved-vs-assigned
  mismatch means stop and report, that agents neither create nor move worktrees, and that the
  pin must be re-confirmed before each write batch rather than only at entry.

- **Named the authoritative screenshot manifest.** `state.json facts.screenshots` is capped and
  merged last-writer-wins, so in a multi-stream run it reflects one stream and drops the rest.
  A stream discovered this alone and invented its own ledger key to avoid erasing a sibling's
  rows. The file now directs captures to the run's `screenshots.md` manifest and says not to
  read or write the ledger key.

- **Stated that returning is what settles a stage.** The completion patch keys on the return
  summary; reporting out of band does not mark a stage complete. A stage that finished its work
  and reported by message left its artifact on disk while the ledger still read `in_progress`.

- **Corrected the `## Artifacts` scope line** so it no longer tells a DV agent that nothing
  outside `.context/` is its to create.

## [1.3.2] — 2026-08-15

### Fixed

- **Audit rows are no longer duplicated across installed plugins.** Every installed dev
  plugin registers its own copy of `hooks/audit-tooluse.sh` and `hooks/audit-subagent.sh`,
  and all of them fire on the same event, so one tool call was recorded six times — twelve
  for subagent-stop, which fires twice per stop. A measured five-hour run produced 2,837
  audit rows of which 2,265 (80%) were advisory duplicates, and every reader of
  `audit.jsonl` paid to parse them.

  `metadata.dedupe_key` was already present and already identical across all copies, and
  nothing consulted it at write time. A reconciler does exist — corpflow's
  `skills/agent-coordination/scripts/audit-dedup.sh`, which keeps the `hook:*` row and drops
  agent rows for the same key — but it is a **read-time filter invoked only by
  `/cost-report`**, not the automatic "audit-dedup hook" these headers name. So the rows still
  accumulate on disk in full, and every other reader of the trail pays for them.

  Both hooks now reconcile at the point of writing: if the key is already present in the tail
  of `audit.jsonl`, the advisory row is dropped. This is consistent with `audit-dedup.sh` —
  the canonical `hook:*` row is never suppressed, so a later read-time dedup still resolves
  every group the same way.

  The canonical orchestrator row is never suppressed — it is written by a different hook
  that carries no advisory flag and performs no such check. Distinct events are unaffected;
  only exact `dedupe_key` repeats are dropped.

## [1.3.1] — 2026-08-15

### Changed

- corpflow contract: the worktask state ledger moved from `stages.<CODE>` to `tasks.<ID>`
  (`state.json` `version: 2`), so `CORPFLOW.md` names the new path. corpflow retired Claude Code's
  Task System after CC 2.1.233 removed those tools on every model it dispatches.
- `hooks/README.md`: corpflow's audit hook now matches `Write|Edit|Bash`, recording Bash rows only
  for ledger patches.

## [Unreleased]

## [1.3.0] — 2026-07-29

Single-framework and single-toolchain rules that had been stated as universal are now
scoped to the framework or package manager they are actually true for. This plugin serves
React, Vue, Svelte, Angular, TypeScript, and plain CSS/HTML; a rule written for one of
them misfires silently on the other five.

### Fixed

- **Package-manager lockouts in agent `tools:` grants.** `fe-code-fixer` and
  `frontend-architector` granted only `Bash(npm:*)`, and `fe-security-auditor` granted no
  manager at all — while all three inherit the base rule to detect the manager from the
  lockfile. On a pnpm/yarn repo the correct command was silently unrunnable, so the
  security audit could report "CVE-clear" on `osv-scanner` alone. All three now grant
  npm/pnpm/yarn.
- **React hooks prescribed as the cross-framework perf fix.** `fe-code-fixer`'s playbook
  offered `useCallback`/`useMemo` for an unstable prop re-rendering a hot child with no
  qualifier, sending a haiku-tier mechanical applier after React hooks in Vue, Svelte, and
  Angular code. Row is now React-scoped, with the `computed`/`$derived`/`OnPush`+signals
  counterpart restored from `fe-performance-engineer`.
- **Single test runners named under "Tooling Mandates".** `css-developer` (Playwright) and
  `typescript-developer` (Vitest) contradicted the plugin's own "never introduce a second
  framework" rule on Cypress, Jest, and Angular repos. Both now defer to the project's
  configured runner, as does the base's "Always Enforce" testing row.
- **`npm ci` as *the* reproducible CI install** in the neutral-scoped `tooling-skills`
  domain constraint and the `build-systems` anti-pattern table — despite `build-systems`
  citing `fe-dependency-manager` as canonical, which forbids cross-manager use. Both now
  give the per-manager frozen install.
- **No Angular row in the bundler-selection table**, with Vite as the unqualified default
  for new apps, leaving an Angular build with no correct answer. Added `@angular/build`
  and qualified the Vite recommendation.
- Hardcoded npm commands in manager-detecting docs: the base's **mandatory**
  screenshot-capture step, and the `npm i -D …` install hints in `fe-test-generator`,
  `css-developer`, and `typescript-developer`.

## [1.2.0] — 2026-07-29

Command-surface unification with the `apple-developer` plugin. The command set grows
from 9 to 16 and adopts the shared cross-plugin naming standard, so the same verb
means the same thing in every company-workflow platform plugin.

### Added

- `/arch-select` — select a frontend architecture: rendering strategy (CSR/SSR/SSG/ISR),
  state management, component boundaries, framework, and repo shape. Routes to
  `frontend-architector`.
- `/arch-review` — review an existing codebase's architecture: layering, state ownership,
  data-fetching seams, component coupling, bundle split. Routes to `frontend-architector`.
- `/analyze-tech-debt` — identify, quantify, and prioritize frontend tech debt across code,
  types, CSS, dependencies, tests, and framework majors. Read-only: no `Write`/`Edit` in
  `allowed-tools`. Routes to `frontend-architector`.
- `/gen-docs` — generate or update TSDoc comments, typedoc API reference, Storybook docs
  (CSF3 + autodocs), and README API sections. Routes to `typescript-developer`.
- `/debug` — configure browser and framework debugging workflows, or triage and root-cause
  a specific web error, over browser devtools, source maps, and framework devtools.
  Routes to `frontend-developer`.
- `/fix-refactor` — clean-code and SOLID refactoring where `frontend-architector` plans and
  `fe-code-fixer` applies; `--extract` pulls code into a shared workspace package.
- `/develop-feature` — end-to-end feature development: `frontend-architector` →
  framework developer → `fe-test-generator` → `fe-security-auditor`, build-gated at every
  phase boundary.
- `/fix-performance` gains an opt-in apply phase. The ranked plan from
  `fe-performance-engineer` can be routed to `fe-code-fixer`, followed by a build gate and
  a re-measure, producing real before/after numbers.
- `/deps` gains explicit first-token subcommand dispatch (`audit` | `upgrade` | `add`).

### Changed

- **Renamed 8 commands** to the cross-plugin standard (see Removed for the mapping).
- `/fix-performance` remains **measure-only by default**. The new apply phase requires both
  the `--apply` flag and explicit user approval at a `PHASE CHECKPOINT`; nothing is written
  before that gate. `Write` and `Edit` were added to its `allowed-tools` solely for that step.
- `/analyze-accessibility` keeps its `--fix` routing flag as a platform extension; the base
  run stays read-only in `allowed-tools`.
- Command frontmatter is unified across all 16 files: `description` is verb-first and at most
  120 characters, `allowed-tools` is minimal, and `estimated-cost` carries `min-tokens`,
  `max-tokens`, and a `model-distribution` summing to 100.
- H1 titles normalized to functional names without a platform suffix (for example
  `# Framework-Aware Code Review` → `# Code Review`).
- In-body command cross-references, `agents/fe-code-fixer.md`, `skills/angular/angular-skills`,
  `skills/_shared/secure-coding`, `README.md`, and `.claude-plugin/marketplace.json` all
  updated to the new names.

### Removed

- The frontend-specific `band:` sub-key under `estimated-cost` is dropped from every command;
  it was not part of the shared standard.
- The old command names. They are **not aliased** — update any script, skill, or worktask
  payload that still calls them:

  | Old name (≤ 1.1.0) | New name (1.2.0+) |
  |--------------------|-------------------|
  | `/code-review` | `/review-code` |
  | `/lint-fix` | `/fix-quick` |
  | `/code-modernize` | `/fix-modernize` |
  | `/profile-performance` | `/fix-performance` |
  | `/generate-tests` | `/gen-tests` |
  | `/deps-audit` | `/deps` |
  | `/a11y-audit` | `/analyze-accessibility` |
  | `/component-scaffold` | `/gen-component` |

  `/build-test` is unchanged.

### Migration

Replace the old command names with the new ones. Two renames change behavior as well as
spelling:

- `/profile-performance` → `/fix-performance`: the default run is still measure-only and
  produces the same artifacts, so existing invocations behave as before. Add `--apply` to
  opt into remediation.
- `/deps-audit` → `/deps`: the audit is now a subcommand rather than the whole command.
  `/deps-audit audit` becomes `/deps audit`; a bare `/deps` still defaults to `audit`.

`scripts/validate.sh --strict` reports 0 errors and 0 warnings on this release.

## [1.1.0] — 2026-07-22

Compatibility ported from company-workflow v3.17.0 to v3.36.0 across the README, the agent
stage-participation headers, the `workflow-integration` skill, and `stage-recipes.md`.
Added the native web port of the visual track: a live-drive provenance gate
(`ui_visual_check`) and its E2E-side twin, the Visible-Enabled Control Sweep.
See `MEMORY.md` § Version History for the full decision record.

<!-- Version-comparison links are omitted: the repository carries no release tags yet. -->

