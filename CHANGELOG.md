# Changelog

All notable changes to the frontend-developer plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] — 2026-07-29

Command-surface unification with the `apple-developer` plugin. The command set grows
from 9 to 16 and adopts the shared cross-plugin naming standard, so the same verb
means the same thing in every igrsoft platform plugin.

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

Compatibility ported from igrsoft v3.17.0 to v3.36.0 across the README, the agent
stage-participation headers, the `workflow-integration` skill, and `stage-recipes.md`.
Added the native web port of the visual track: a live-drive provenance gate
(`ui_visual_check`) and its E2E-side twin, the Visible-Enabled Control Sweep.
See `MEMORY.md` § Version History for the full decision record.

<!-- Version-comparison links are omitted: the repository carries no release tags yet. -->

