---
description: Run linters and formatters (ESLint or Biome, Prettier, Stylelint) over a web project — check-only or auto-fix — then re-check
argument-hint: [path (default .)] [--check | --fix] [--only eslint|biome|prettier|stylelint]
allowed-tools: Read, Edit, Glob, Grep, Bash
estimated-cost:
  band: low
  min-tokens: 500
  max-tokens: 6000
  model-distribution:
    haiku: 90%
    sonnet: 10%
---

# Lint & Fix
<!-- Updated: June 2026 -->

Run the project's standard linter and formatters over the target, report violations, and — in `--fix` mode — apply the safe, deterministic auto-fixes, then re-check. Fast, cheap, and reversible: this is the deterministic-cleanup pass, not a review. Deep, judgment-bearing fixes escalate to `/frontend-developer:code-review --fix`.

[Extended thinking: This command is the frontend-developer analogue of a pre-commit hook. It detects which linter/formatter stack the project uses (ESLint *or* Biome — never both as the JS/TS linter; Prettier for formatting unless Biome owns it; Stylelint for CSS), discovers each tool's config so it honors project rules instead of imposing its own, and runs them in a fixed order. `--check` is the CI mode — no edits, exit-code-honest, with a per-rule violation count — and `--fix` applies only the mechanical fixes (`eslint --fix`, `biome check --write`, `prettier --write`, `stylelint --fix`) then re-runs the linters to confirm. Anything a `--fix` rule cannot resolve mechanically (type errors, accessibility-rule violations needing markup changes) is reported, not forced; those land in `code-review --fix`. Keep it on haiku: the work is tool invocation and table assembly, not analysis.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **`--check` never edits.** In `--check` (or default-with-no-flag) mode, run every tool in its report-only variant. Do NOT pass `--fix`/`--write`. If any tool reports a violation, the result is FAIL — surface the per-rule violation counts.
2. **`--fix` applies only mechanical fixes, then re-checks.** Run the auto-fixers, then re-run the linters in report-only mode. Report what was fixed and what remains. Never claim "clean" without the post-fix re-check passing.
3. **Honor project config, do not impose.** Discover and use `eslint.config.*`/`.eslintrc.*`, `biome.json`, `.prettierrc*`, `.stylelintrc*`, and `.editorconfig` (see Config Discovery). Pass no opinionated overrides when a config exists; fall back to documented defaults only when none is found, and say so in the report.
4. **Never run two JS/TS linters.** ESLint and Biome are mutually exclusive as the JS/TS linter. If both configs exist, prefer the one wired into `package.json` `scripts.lint`; if still ambiguous, prefer ESLint and note the choice. Biome can also own formatting — if `biome.json` formats, do not also run Prettier on the same files.
5. **Single-command Bash invocations.** Use each tool's own path/recursion flags. Never `cd`-chain or `&&`-chain — scoped Bash patterns do not match compound commands.
6. **Tool-missing never hard-fails.** If a linter/formatter binary is absent, print the install hint, skip that pass, and continue. Report what was skipped.
7. **This is the shallow pass.** Do NOT attempt semantic refactors, type fixes, or accessibility-markup changes. When a finding needs judgment, list it under "Needs review" and point to `/frontend-developer:code-review --fix`. Do not route to an agent from this command.
8. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Report violations across all detected tools (CI-safe, no edits)
/frontend-developer:lint-fix . --check

# Auto-fix everything fixable, then re-check
/frontend-developer:lint-fix . --fix

# Fix only the CSS under a subtree
/frontend-developer:lint-fix src/styles --fix --only stylelint

# Check just the JS/TS lint (exit non-zero if any violation)
/frontend-developer:lint-fix src/ --check --only eslint
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `path` | `.` | Directory or file to lint. Detection and tool discovery are rooted here. |
| `--check` | default | Report-only. No edits. FAIL if any violation remains. This is the CI mode. |
| `--fix` | off | Apply mechanical fixer changes, then re-check. Mutually exclusive with `--check`; `--fix` wins if both are passed (with a warning). |
| `--only eslint\|biome\|prettier\|stylelint` | all | Restrict the run to one tool. Without it, every detected tool is processed. |

When neither `--check` nor `--fix` is given, default to `--check`.

## Tool Detection

Detect which tools the project uses, then run each one's pass. Prefer the tool wired into `package.json` `scripts`.

| Tool | Detect via | Owns |
|------|------------|------|
| ESLint | `eslint.config.*` / `.eslintrc.*`; `eslint` in deps | JS/TS lint (incl. `eslint-plugin-jsx-a11y`, framework plugins) |
| Biome | `biome.json` / `biome.jsonc`; `@biomejs/biome` in deps | JS/TS lint + format (alternative to ESLint + Prettier) |
| Prettier | `.prettierrc*` / `prettier` key in `package.json`; `prettier` in deps | formatting (unless Biome formats) |
| Stylelint | `.stylelintrc*`; `stylelint` in deps | CSS/SCSS/`*.module.css` lint |

ESLint and Biome are mutually exclusive (Rule 4). `--only` overrides detection and processes only that tool.

## Config Discovery

Before running each tool, discover its configuration so the run honors project rules. Record which config (if any) was found in the report.

| Tool | Config (precedence) | If missing |
|------|---------------------|------------|
| ESLint | `eslint.config.js`/`.mjs`/`.ts` (flat) → `.eslintrc.*` (legacy) | use `eslint --no-eslintrc` only if explicitly intended; otherwise report "no ESLint config — skipped" |
| Biome | `biome.json` / `biome.jsonc` | run with Biome defaults and note it |
| Prettier | `.prettierrc*` / `prettier` in `package.json` / `.prettierrc` in `.editorconfig` | Prettier defaults |
| Stylelint | `.stylelintrc*` / `stylelint` in `package.json` | report "no Stylelint config — skipped" (no opinionated default) |
| All | `.editorconfig` | informational — Prettier and Biome both consult it |

## Per-Tool Order

Run tools in this order. In `--fix`, formatters run **after** lint-fixes so formatting churn does not mask real lint findings, then a final lint re-check confirms.

| Tool | Lint (check) | Format (check) | Fix (mechanical) |
|------|--------------|----------------|------------------|
| ESLint | `npx eslint <path>` | — | `npx eslint <path> --fix` (only auto-fixable rules) |
| Biome | `npx biome lint <path>` | `npx biome format <path>` (no `--write`) | `npx biome check <path> --write` (lint + format) |
| Prettier | — | `npx prettier --check <path>` | `npx prettier --write <path>` |
| Stylelint | `npx stylelint "<path>/**/*.{css,scss}"` | — | `npx stylelint "<path>/**/*.{css,scss}" --fix` |

Notes:
- **ESLint type-aware and a11y rules are report-only for behavior changes.** `eslint --fix` applies only the rules ESLint marks auto-fixable; an `jsx-a11y` rule that needs a label/role added is reported, not forced (Rule 7).
- `npx eslint <path> --format json` / `npx stylelint --formatter json` produce the per-rule counts used for the `--check` summary.
- If Biome owns the project, it replaces both the ESLint and Prettier rows; do not double-run.

## Workflow

### Phase 1: Detect & Discover (Bash)

1. Confirm `path` exists; if not, emit the Error Handling "path not found" message and stop.
2. Resolve mode: `--fix` if present (warn if `--check` also passed), else `--check`.
3. Detect the linter/formatter stack (honoring `--only`). Resolve the ESLint-vs-Biome choice (Rule 4). If nothing detected, emit "no lintable config" and stop.
4. For each detected tool, run Config Discovery and note which config was found.
5. Verify each tool's binary exists (`command -v npx`; tools resolve via the project). Missing → print install hint, skip that pass, note the skip.

### Phase 2a: Check mode (`--check`)

1. For each tool, run the **lint (check)** and **format (check)** commands from the order table. Do NOT edit.
2. Collect violations per tool with per-rule counts (`eslint --format json`, `stylelint --formatter json`).
3. Assemble the Output Format report. If any tool reported a violation, result is FAIL (CI-honest). If all clean, PASS.

### Phase 2b: Fix mode (`--fix`)

1. For each tool, apply the **fix (mechanical)** commands: `eslint --fix`, then `prettier --write` (or `biome check --write` if Biome owns both), then `stylelint --fix`.
2. Record each file touched and the rule/category of each applied fix (for the File | Line | Fix table).
3. **Re-check**: re-run the report-only lint commands. Anything still failing is reported as remaining, with judgment items (type errors, a11y-markup rules) routed to "Needs review."
4. Result is PASS only if the re-check is clean; otherwise PARTIAL with the remaining count.

### Phase 3: Report (Bash)

Emit the Output Format. In `--fix`, include the applied-fix table, the rollback hint, and any "Needs review" escalation. In `--check`, include the violation table and per-rule counts.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint |
|--------------|--------------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) |
| `eslint` | `npm install -D eslint` (and the project's plugins, e.g. `eslint-plugin-jsx-a11y`) |
| `@biomejs/biome` | `npm install -D --save-exact @biomejs/biome` then `npx biome init` |
| `prettier` | `npm install -D prettier` |
| `stylelint` | `npm install -D stylelint stylelint-config-standard` |

Exact flag spellings vary across tool releases — verify against your toolchain when a flag is rejected. Never hard-fail on a missing tool: print the hint, skip that pass, continue, and report the skip. Only when *every* detected tool is missing does the command report FAIL with the aggregated install hints.

## Output Format

```markdown
## Lint & Fix Report

**Target:** {path}
**Mode:** check | fix
**Stack:** {ESLint + Prettier + Stylelint | Biome + Stylelint}
**Configs found:** {eslint.config.js ✅ | biome.json ✅ | .prettierrc ✅ | .stylelintrc ❌ (skipped)}

| Tool | Violations | Status |
|------|-----------:|--------|
| ESLint | 5 | ❌ (no-unused-vars ×2, jsx-a11y/alt-text ×1, react-hooks/exhaustive-deps ×2) |
| Prettier | 3 files | ❌ would reformat |
| Stylelint | 2 | ❌ (no-duplicate-selectors ×1, color-no-invalid-hex ×1) |

**Result:** PASS / FAIL / PARTIAL — {N violations across M tools}

<!-- --fix mode only: applied fixes -->
### Fixes Applied ({total})
| File | Line | Fix |
|------|------|-----|
| src/App.tsx | 12 | eslint no-unused-vars: removed unused import `clsx` |
| src/App.tsx | — | prettier: reformatted |
| styles/card.css | 8 | stylelint: lowercased hex color |

### Rollback
To undo every change this run made:
```bash
git checkout -- {files}
```

<!-- when mechanical fixes cannot resolve everything -->
### Needs review ({count})
- {file}:{line}: {jsx-a11y/alt-text or react-hooks/exhaustive-deps finding that needs judgment}
- Escalate with: `/frontend-developer:code-review --fix {path}`

<!-- on skipped tools only -->
### Skipped
- {tool}: {missing binary or config} — install hint printed above.
```

In `--check` mode, the "Violations" column doubles as the per-rule summary: each cell shows the count and rule codes, so a CI run surfaces exactly which rules tripped.

## Error Handling

### Path not found
```
Error: Path not found: {path}
Suggestion: Pass a directory or file that exists, e.g. /frontend-developer:lint-fix . --check
```

### No lintable config
```
Error: No ESLint/Biome/Prettier/Stylelint config found under {path}.
Suggestion: Add a config (e.g. `npx eslint --init`, `npx biome init`) or pass --only to target a specific tool.
```

### Both --check and --fix passed
```
Warning: --check and --fix are mutually exclusive; proceeding with --fix.
(Run again with only --check for a CI-safe, no-edit pass.)
```

### Both ESLint and Biome configs present
```
Note: both ESLint and Biome configs found. Using {chosen} (wired into package.json scripts / ESLint default).
Run with --only biome to force Biome instead.
```

### Tool missing
Print the install hint, skip that pass, continue. Only when *every* detected tool is missing does the command report FAIL with the aggregated install hints.

## See Also

- `/frontend-developer:build-test` — run before building to cut warning noise; lint, then build green.
- `/frontend-developer:code-review --fix` — escalation target for findings that need judgment (type errors, a11y-markup, semantic refactors) beyond mechanical lint fixes.
- `/frontend-developer:code-modernize` — for framework migrations, which go deeper than this command's mechanical pass.
- `skill: fe-diagnostics` — ESLint/Biome/Stylelint flag reference and the formatter-vs-linter division.
