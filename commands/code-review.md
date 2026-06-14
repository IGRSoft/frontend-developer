---
description: Framework-aware code review for React, Vue, Svelte, Angular, TypeScript, and CSS — parallel per-framework reviewers plus an accessibility pass and a security pass, synthesized into a P0-P3 report
argument-hint: [scope: file/dir/PR#/branch — default: working changes] [--quick] [--fix] [--framework react|vue|svelte|angular]
allowed-tools: Read, Glob, Grep, Bash, WebSearch, WebFetch
estimated-cost:
  band: medium
  min-tokens: 4000
  max-tokens: 28000
  model-distribution:
    haiku: 15%
    sonnet: 75%
    opus: 10%
---

# Framework-Aware Code Review
<!-- Updated: June 2026 -->

Review web UI changes with the right specialist per framework, plus a dedicated accessibility pass and a dedicated security pass, then synthesize one deduplicated, prioritized P0-P3 report. Scope defaults to your working changes; reviewers run read-only and in parallel; `--fix` hands the blocking findings to the code fixer under a minimal-diff gate.

[Extended thinking: A React stale-closure in `useEffect`, a Vue reactivity loss from destructuring a `reactive()`, a Svelte rune misuse, an Angular `OnPush` change-detection trap, an unlabeled icon button, and a `dangerouslySetInnerHTML` XSS sink are six different review skills — one generalist pass misses most of them. This command resolves the scope once, detects which frameworks are actually present from `package.json` + file extensions, then routes Claude to one read-only reviewer per detected framework (each loaded with the right review focus) alongside one accessibility pass and one cross-cutting security pass, then merges and ranks the findings. The reviewers never edit; only the explicit `--fix` step does, and only for P0/P1. Keep the synthesis honest — if there are no material issues, say so rather than padding the report.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Resolve the scope before reviewing.** Apply the scope precedence (explicit args > working diff > branch/PR diff) exactly once, list the concrete files under review, and use that same file list for every reviewer. Do NOT let reviewers re-scope independently.
2. **Reviewers are read-only.** The review phase MUST NOT write or edit. Reviewers return structured findings only. The single place edits happen is the `--fix` step, after synthesis, and only for P0/P1 findings.
3. **One reviewer per detected framework.** Route a reviewer only for a framework that is actually present in the scope (or forced by `--framework`). Do NOT spawn a Vue reviewer for a pure-React change. Run the eligible reviewers in parallel — they have no dependencies on each other.
4. **Accessibility and security passes always run** (unless `--quick`). The a11y pass (`frontend-developer:fe-accessibility-auditor`) and the security pass (`frontend-developer:fe-security-auditor`) are cross-cutting and run alongside the framework reviewers, not after them.
5. **Synthesize, deduplicate, normalize.** Merge all reviewer outputs, drop duplicates and speculative claims, and normalize every surviving finding to `{file, line, category, severity, why, fix, confidence}` before ranking into P0-P3.
6. **Tool-missing never hard-fails.** If a reviewer's underlying linter/analyzer (`eslint`, `axe`, `playwright`) is unavailable, print the install hint, note the reduced depth, and continue. Never abort the whole review over one missing tool.
7. **No manufactured feedback.** If the synthesis finds no material issue, report that plainly. Do NOT invent P2/P3 nits to fill the report.
8. **Commands route, they do not orchestrate.** This command documents which `frontend-developer:*` agent owns each lens so Claude routes the work; it does not itself call `Task` — routing is performed by Claude in the surrounding session.
9. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Review your current working changes (staged + unstaged)
/frontend-developer:code-review

# Review a specific directory
/frontend-developer:code-review src/components

# Review a single file
/frontend-developer:code-review src/App.tsx

# Review a branch or PR against the base
/frontend-developer:code-review feature/new-checkout
/frontend-developer:code-review 142            # PR number

# Fast single-agent pass for quick feedback
/frontend-developer:code-review src/ --quick

# Review, then auto-fix the P0/P1 findings
/frontend-developer:code-review src/ --fix

# Force a framework when detection is ambiguous
/frontend-developer:code-review src/ --framework react
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `scope` | working changes | File, directory, PR number, or branch to review. See Scope Resolution. |
| `--quick` | off | Single combined reviewer pass for rapid feedback. Skips parallel fan-out and the dedicated a11y/security passes; folds a lightweight a11y + security check into the one pass. |
| `--fix` | off | After synthesis, route P0/P1 findings to `frontend-developer:fe-code-fixer` under a minimal-diff gate. P2/P3 are never auto-fixed. |
| `--framework react\|vue\|svelte\|angular` | auto | Force the reviewer set instead of detecting. Use for monorepos or extensionless ambiguity. |

## Scope Resolution

Resolve the set of files under review **once**, top-down — the first applicable rule wins:

1. **Explicit args** — a file, directory, PR number, or branch named on the command line.
   - File or directory → review those paths directly.
   - PR number (bare integer) → `gh pr diff <N> --name-only` for the file list (and `gh pr diff <N>` for the patch). If `gh` is unavailable, print the install hint and fall back to rule 3 against the PR's base branch.
   - Branch name → diff against the merge-base with the default branch: `git diff --name-only $(git merge-base HEAD <branch>)..<branch>`.
2. **Working changes** (no args) — staged and unstaged tracked changes:
   `git diff --name-only HEAD` (plus `git diff --cached --name-only`). This is the default.
3. **Branch/PR diff** (fallback) — when neither explicit paths nor working changes apply, diff the current branch against the default branch's merge-base.

After resolving, **print the concrete file list** and the line ranges (where a diff is involved) before launching any reviewer. Reviewers receive this exact list — they do not re-derive scope. Exclude build artifacts (`dist/`, `build/`, `.next/`, `.svelte-kit/`, `node_modules/`, `coverage/`) from the list.

## Framework Detection

Detect which frameworks appear in the resolved file list using the canonical `skill: language-detection` table — do not fork its routing logic. Summary for this command:

| Markers in scope | Reviewer to route to |
|------------------|----------------------|
| `*.tsx`/`*.jsx` + `react`/`next` in `package.json` | `frontend-developer:react-developer` |
| `*.vue` + `vue`/`nuxt` in `package.json` | `frontend-developer:vue-developer` |
| `*.svelte` + `svelte`/`@sveltejs/kit` in `package.json` | `frontend-developer:svelte-developer` |
| `@angular/core` in `package.json` + `*.component.ts` | `frontend-developer:angular-developer` |
| `*.ts` with no framework markers (libs, utilities) | `frontend-developer:typescript-developer` |
| `*.css`/`*.scss`/`*.module.css` | `frontend-developer:css-developer` |

- A `--framework` flag overrides detection for that framework.
- A change spanning several frameworks routes **one reviewer per framework present** — they run in parallel.
- If nothing recognized is in scope, report "no reviewable web sources in scope" and stop.

## Workflow

### `--quick` path (single pass)

When `--quick` is set, skip the fan-out entirely:

1. Resolve scope and detect the dominant framework.
2. Route to `frontend-developer:<dominant-framework-developer>` for a quick read-only review of the file list, focused on correctness, a11y, and security for that framework. The reviewer must NOT edit and returns findings as `{file, line, category, severity (P0-P3), why, fix, confidence}`. If there are no material issues, it says so directly.
3. Normalize and print the report (Output Format). Skip the dedicated a11y/security passes — the single reviewer folds in lightweight checks.

`--quick` is for fast feedback on a single-framework change; for mixed repos or pre-merge gates, use the full path.

### Phase 1: Parallel Read-Only Review

Route every eligible reviewer **simultaneously** (one per detected framework) plus the a11y and security passes. All are read-only and receive the same resolved file list. Each framework reviewer gets a framework-specific review focus:

**React → `frontend-developer:react-developer`**
- Focus: hook rules (conditional hooks, missing deps in `useEffect`/`useMemo`/`useCallback`, stale closures), unstable identities causing re-renders, key correctness in lists, `useState` derived-state anti-patterns, Server vs Client component boundaries and hydration mismatches (RSC), `use` hook misuse, effect cleanup, context over-subscription.
- Brief: "Read-only review of the React files: {file_list}. Review for: hook-rule violations and exhaustive-deps, stale closures, unstable callback/object identities, list keys, Server/Client boundary + hydration correctness, effect cleanup, context over-subscription. Do NOT edit. Return findings as `{file, line, category, severity (P0-P3), why, fix, confidence}`. If clean, say so."

**Vue → `frontend-developer:vue-developer`**
- Focus: Composition-API reactivity (reactivity loss from destructuring `reactive()`, missing `.value` on refs, `ref` vs `reactive` choice), `watch`/`watchEffect` dependency traps, `computed` with side effects, `v-for` keys, props mutation, `defineProps`/`defineEmits` typing, lifecycle in `<script setup>`.
- Brief: "Read-only review of the Vue files: {file_list}. Review for: reactivity loss (destructured reactive, missing .value), watch/computed traps, v-for keys, props mutation, defineProps/defineEmits typing, lifecycle correctness. Do NOT edit. Return `{file, line, category, severity (P0-P3), why, fix, confidence}`. If clean, say so."

**Svelte → `frontend-developer:svelte-developer`**
- Focus: Svelte 5 runes discipline (`$state`/`$derived`/`$effect` misuse, `$derived` with side effects, `$effect` for derived state that should be `$derived`), reactivity outside runes, store auto-subscription leaks, `{#key}` correctness, snippet/slot misuse, SSR/hydration in SvelteKit.
- Brief: "Read-only review of the Svelte files: {file_list}. Review for: runes discipline ($state/$derived/$effect misuse, $effect-for-derived-state), reactivity correctness, store subscription leaks, {#key} use, SSR/hydration. Do NOT edit. Return `{file, line, category, severity (P0-P3), why, fix, confidence}`. If clean, say so."

**Angular → `frontend-developer:angular-developer`**
- Focus: signals discipline (`signal`/`computed`/`effect` misuse, mixing signals with manual `markForCheck`), `OnPush` change-detection correctness, RxJS subscription leaks (missing `takeUntilDestroyed`/`async` pipe), standalone-component wiring, `inject()` context rules, template expression cost.
- Brief: "Read-only review of the Angular files: {file_list}. Review for: signals discipline, OnPush change-detection correctness, RxJS subscription leaks (async pipe / takeUntilDestroyed), standalone wiring, inject() context, template-expression cost. Do NOT edit. Return `{file, line, category, severity (P0-P3), why, fix, confidence}`. If clean, say so."

**TypeScript → `frontend-developer:typescript-developer`**
- Focus: `any` leaks and unsafe casts, missing discriminated-union exhaustiveness, unsound narrowing, public API typing, `strict` violations masked by `// @ts-ignore`, generic variance, module-boundary types.
- Brief: "Read-only review of the TypeScript files: {file_list}. Review for: any leaks/unsafe casts, missing exhaustiveness, unsound narrowing, public-API typing, suppressed strict errors, generic variance. Do NOT edit. Return `{file, line, category, severity (P0-P3), why, fix, confidence}`. If clean, say so."

**CSS → `frontend-developer:css-developer`**
- Focus: specificity wars and `!important` abuse, layout fragility (magic numbers vs intrinsic sizing), unguarded modern features (container queries / `:has()` without fallback), z-index stacking, RTL/logical-property gaps, reduced-motion handling.
- Brief: "Read-only review of the CSS/SCSS files: {file_list}. Review for: specificity/`!important` abuse, fragile layout, unguarded modern features, z-index stacking, logical-property/RTL gaps, reduced-motion. Do NOT edit. Return `{file, line, category, severity (P0-P3), why, fix, confidence}`. If clean, say so."

**Accessibility pass (always, unless `--quick`) → `frontend-developer:fe-accessibility-auditor`**
- Brief: "Read-only WCAG 2.2 review of: {file_list}. Cover ARIA correctness, name/role/value, keyboard operability and focus order, contrast, form labeling, live regions, motion preferences. Map findings to WCAG success criteria. Do NOT edit. Return `{file, line, category (SC), severity (P0-P3), why, fix, confidence}`. If clean, say so." (Review-only — it does not patch.)

**Security pass (always, unless `--quick`) → `frontend-developer:fe-security-auditor`**
- Brief: "Read-only cross-cutting web-security review of: {file_list} (frameworks present: {frameworks}). Cover XSS sinks (`dangerouslySetInnerHTML`, `v-html`, `{@html}`, `innerHTML`, `eval`), CSP gaps, secrets in client bundle, unsafe `target=_blank` (`rel`), SSRF via SSR fetch, prototype pollution, npm supply-chain risk in changed deps. Map each finding to a CWE where applicable. Do NOT edit. Return `{file, line, category (CWE), severity (P0-P3), why, fix, confidence}`. If clean, say so." (Review-only — it does not patch.)

[SYNC POINT: Wait for all Phase 1 reviewers before synthesis.]

### Phase 2: Synthesis

1. **Collect** every reviewer's findings (framework reviewers + a11y pass + security pass).
2. **Deduplicate** — the security and a11y passes overlap with framework reviewers (e.g. both flag an XSS sink). Merge duplicates at the same `{file, line}`, keeping the higher severity and clearer fix; credit both lenses in `why`.
3. **Filter** — drop speculative claims with no concrete evidence and pure style nits unless they hide a real defect. Per Rule 7, do not backfill.
4. **Normalize** every survivor to `{file, line, category, severity, why, fix, confidence}` (severity from `skill: severity-matrix`; confidence = high/medium/low).
5. **Rank** into P0-P3.
6. **Emit** the Output Format report.

### Optional: `--fix` (P0/P1 only)

If `--fix` is set, after synthesis route the blocking findings to the fixer:

Route to `frontend-developer:fe-code-fixer` with: "Apply minimal, targeted fixes for these P0/P1 findings: {p0_p1_findings as `{file, line, category, fix}`}. Minimal-diff gate: change only what each finding requires; do not refactor, reformat untouched code, or fix P2/P3 items. Preserve behavior outside the stated defect. After fixing, report each change as `{file, line, finding, change}` and list any finding you could NOT safely auto-fix."

- Only P0/P1 with a concrete, localized fix are eligible. Anything needing design judgment is returned for manual handling.
- Re-run a focused review on the touched files to confirm no regression (a single `--quick` pass over the changed files is sufficient).

## Output Format

```markdown
## Code Review Report

**Scope:** {resolved scope — paths / PR# / branch}
**Files reviewed:** {N} ({frameworks present})
**Reviewers:** {list of agents} {+ a11y pass + security pass}
**Mode:** {full | --quick}

### Summary
{One or two sentences. If clean: "No material issues found — the changes look correct, accessible, and secure." Otherwise: counts by priority.}

| Priority | Count |
|----------|-------|
| P0 (block merge) | {n} |
| P1 (fix in this change) | {n} |
| P2 (should fix) | {n} |
| P3 (nice to have) | {n} |

### P0 — Must Fix Before Merge
| File:Line | Category | Why | Fix | Confidence |
|-----------|----------|-----|-----|------------|
| {file}:{line} | {category/CWE/SC} | {why it's broken} | {minimal fix} | {high/med/low} |

### P1 — Fix In This Change
{same table shape}

### P2 — Should Fix
{same table shape}

### P3 — Nice To Have
{same table shape}

<!-- When --fix ran: -->
### Fixes Applied
| File:Line | Finding | Change |
|-----------|---------|--------|

**Not auto-fixed (manual):** {findings needing human judgment, or "none"}

<!-- When a tool was unavailable: -->
### Reduced-Depth Notes
- {framework}: {missing tool} unavailable — review ran without {analyzer}. Install: {hint}.
```

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect on review |
|--------------|--------------|------------------|
| `npx` / `node` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | skip all tool-backed depth; review reads source only |
| `eslint` | `npm install -D eslint` (or `npx eslint` resolves it) | framework reviewers run without lint signal — note reduced depth |
| `axe-core` / `@axe-core/cli` | `npm install -D @axe-core/cli` | a11y pass runs static-only; note reduced depth |
| `playwright` | `npm install -D @playwright/test && npx playwright install` | skip runtime DOM checks; static review continues |
| `gh` (PR scope) | `brew install gh` then `gh auth login` | fall back to a branch diff against the default branch |

Tool-missing is always a skip-with-note, never a hard failure: print the hint, record the reduced depth in the report, and continue. The command only reports "could not review" when *no* reviewable sources are in scope (not when a tool is missing).

## Error Handling

### No reviewable sources in scope
```
Note: No web sources found in the resolved scope.
Resolved scope: {scope}
Suggestion: Pass an explicit path, or check that your changes include reviewable sources.
```

### No changes detected (default scope)
```
Note: No staged or unstaged changes to review.
Suggestion: Name a path, branch, or PR number, e.g. /frontend-developer:code-review src/
```

## See Also

- `skill: language-detection` — canonical marker → framework → agent routing (keep this command's detection in sync).
- `skill: severity-matrix` — P0-P3 definitions used by the synthesis ranking.
- `skill: secure-coding` — XSS/CSP/secrets patterns the security pass draws on.
- `skill: accessibility-baseline` — WCAG 2.2 success criteria the a11y pass draws on.
- `/frontend-developer:lint-fix` — run formatters/linters first to clear P3 noise before review.
- `/frontend-developer:build-test` — confirm the change builds and tests green before or after review.
- `/frontend-developer:a11y-audit` — escalate an accessibility finding to a full axe-core/Lighthouse audit.

If there are no material issues, say that directly instead of manufacturing feedback.
