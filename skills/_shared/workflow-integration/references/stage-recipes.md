# Per-Stage Recipes and Contract Detail

Use this when you need the field-level detail behind a specific workflow stage. `SKILL.md` carries the handoff-contract summary, the stage participation overview, and the DV Screenshot Gate binding rules; this file holds the per-stage artifact field tables, review/QA criteria, the artifact-filename and handoff-frontmatter schemas, the gate-feedback re-dispatch contract, and the supporting reference tables (error files, qualified names, budgets, sizing).

## Worktask Triggers (v4.0.0)

| Trigger | Stages | Use Case |
|---------|--------|----------|
| `micro:` | Plan → approve → edit | Single-file fixes, typos |
| `quick:` | PL → DV → DR → QA | Small features, bug fixes |
| `worktask:` | Full 9-stage (PL→AR→TL→DV→DR→QA→DC→FN→ST) | Multi-file features |
| `fworktask:` | Full 9-stage, auto-continue | Trusted full runs |
| `--secure` / `--full` | 11-stage (adds SR, RE) | Security-critical work |
| `emergency:` | IR→DV→DR→QA→RE→FN | Hotfixes, incidents (DR enforces minimal-diff) |

## DV Contract for Web Work

The DV agent writes `.context/development-N.md`. Mandatory H2 anchors are fixed by company-workflow's anchor allow-list (`handoff-protocol.md#anchor-allow-list`): `## files-changed`, `## tests-added`, `## deviations`, `## follow-ups`. Web-specific sections nest as H3 under them:

| Section | Anchor level | Content |
|---------|--------------|---------|
| Files Changed | `## files-changed` | File / change / why table |
| Decisions | `### decisions` (under files-changed) | Non-obvious implementation choices with rationale |
| Tool Invocations | `### tool-invocations` (under files-changed) | Exact build/lint commands run (`npm run build`, `npx tsc --noEmit`, `npx eslint .`, `npx vitest run`, …) — one command per call (single-command scoped Bash) |
| Tests Added | `## tests-added` | Test files + what each covers |
| Build Evidence | `### build-evidence` (under tests-added) | Toolchain + versions (e.g. `vite 5 / tsc 5.x`), `tsc --noEmit` → 0 errors, eslint/biome **zero-error**, bundle-size delta (gzip kB), test transcript path in `.context/logs/` |
| Deviations | `## deviations` | Departures from `analyzing-N.md` decisions |
| Follow-ups | `## follow-ups` | Deferred work, flagged risks |

Build Evidence is non-negotiable: a DV artifact without a toolchain+version line, a `tsc --noEmit` result, an eslint/biome status, and a test transcript path under `.context/logs/` is incomplete. Tee raw build/test output to `.context/logs/<tool>-<worktask_id>.log`. Copy-paste template: [templates/dv-development.md](../templates/dv-development.md).

## Per-Agent Error Files

Parallel-safe retry narratives live in `.context/errors/<agent-basename>.md` — basename = last `:`-separated segment of the qualified name (`task-system.md § error_file derivation`):

| File | Purpose |
|------|---------|
| `.context/errors/react-developer.md` | React-specific DV retry narratives |
| `.context/errors/vue-developer.md` | Vue DV retry narratives |
| `.context/errors/svelte-developer.md` | Svelte DV retry narratives |
| `.context/errors/angular-developer.md` | Angular DV retry narratives |
| `.context/errors/typescript-developer.md` | TypeScript DV retry narratives |
| `.context/errors/css-developer.md` | CSS/styling DV retry narratives |
| `.context/errors/fe-code-fixer.md` | DR fix-application retries |
| `.context/errors/<agent-basename>.md` | One file per agent — never overwrite a shared `error.md` |

Derive your own path from `task.metadata.error_file` or your frontmatter `name:`.

## DR Web Review Criteria

technical-lead reads `development-N.md` + error files and produces `developer-review-N.md` (verdict `pass`/`fail`). frontend-developer agents support DR and pre-check against these criteria before returning from DV:

| Area | What DR checks |
|------|----------------|
| Framework anti-patterns | Hook-rule violations (React conditional/looped hooks, missing deps); Vue reactivity loss (destructured props, non-`ref`/`reactive` state); Svelte 5 runes misuse (`$state`/`$derived`/`$effect` discipline); Angular signals vs. `async` pipe consistency, standalone-component boundaries |
| Reactivity / hydration | Server/client markup mismatch (Next.js/Nuxt/SvelteKit hydration errors); `useEffect`/`onMounted` used where render-time data suffices; non-deterministic render (Date/random) breaking SSR; stale-closure bugs |
| Accessibility | Missing/incorrect ARIA, unlabeled controls, broken keyboard/focus order, non-semantic elements, contrast failures — see `_shared/accessibility-baseline.md`; `axe-core` must be clean |
| Render performance | Unmemoized expensive renders, unnecessary client components (RSC boundary), large client bundles, layout thrash, unkeyed lists — see `quality/web-performance/SKILL.md` |
| Type safety | `any` without a justifying comment, unsafe `as` casts, untyped event handlers, missing prop/emit types; `tsc --noEmit` must be 0 errors |
| Build hygiene | eslint/biome zero-error, no committed `dist/`/`node_modules`, lockfile updated with `package.json` changes, no unbounded bundle-size regression |

Template: [templates/dr-review.md](../templates/dr-review.md).

## QA Gate for Web Work

QA (`testing-N.md`, verdict `go`/`no-go`) passes only when **all three** hold:

1. **All tests pass** — full suite, not just new tests (`npx vitest run`, `npx jest`, `npx playwright test`).
2. **axe-clean on changed views** — run `axe-core` against the rendered routes; **zero violations** at the configured WCAG level (2.2 AA baseline). See `_shared/accessibility-baseline.md`.
3. **Lighthouse budget met** — Core Web Vitals within budget on the changed routes (LCP < 2.5s, INP < 200ms, CLS < 0.1) and no bundle-size budget regression. See `quality/web-performance/SKILL.md`.

fe-test-generator supports QA with framework-native generation (Vitest/Jest + Testing Library for components, Playwright for E2E, framework detection per `_shared/language-detection.md`). Template: [templates/qa-testing.md](../templates/qa-testing.md).

## SR and RE Contributions

- **SR** — fe-security-auditor provides web platform context to company-workflow's security-reviewer: XSS sink review (`dangerouslySetInnerHTML`, `v-html`, `{@html}`, `innerHTML`, `bypassSecurityTrust*`), Content-Security-Policy and Trusted Types posture, CSRF/clickjacking, secrets-in-bundle scan (no API keys shipped to the client), SSRF via SSR/RSC fetch, and npm supply-chain audit (`npm audit`, `osv-scanner`). Review-only: findings route to `frontend-developer:fe-code-fixer` for application. See `_shared/secure-coding/SKILL.md`.
- **RE** — release-engineer owns the stage; frontend-developer contributes packaging: fe-dependency-manager freezes lockfiles/pins (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`), and the framework agents produce release artifacts (built bundles, version bumps, changelog entries) recorded in `release-N.md`.

## Artifact Filename Contract (v4.0.0)

**Numbered `<stage>-N.md` names are canonical** per company-workflow's authoritative `handoff-protocol.md#stage-artifact-map`. N is allocated by PL0 (same value as `planning-N.md`), shared across all stages within a run, and propagated via `task.metadata.run_index`; it bumps on gate loop-back re-dispatch. Readers fall back to newest-glob (`<basename>-*.md`).

| Stage | Artifact | Owner |
|-------|----------|-------|
| PL | `planning-N.md` | product-manager |
| AR | `analyzing-N.md` | software-architector |
| TL | `coordination-N.md` | team-lead |
| DV | `development-N.md` | developer / frontend-developer agents |
| DR | `developer-review-N.md` | technical-lead |
| SR | `security-review-N.md` | security-reviewer |
| QA | `testing-N.md` | qa-engineer |
| DC | `documentation-N.md` | technical-writer |
| RE | `release-N.md` | release-engineer |
| FN | `complete-summary-N.md` | project-manager |
| ST | `retrospective-N.md` | stakeholder |
| IR | `incident-N.md` | incident-responder |
| ET | `ethics-review-N.md` | ethics-reviewer |

**Emit `handoff:` frontmatter unconditionally — it is the merge input regardless of filename.** state.json reconciliation is three-layered: Layer 1 (agent runs `state-patch.sh --stage <CODE> --prev <PREV>` when its path is supplied, else skips — never a hand-rolled `jq`/manual merge), Layer 2 (orchestrator re-reads artifact frontmatter after `Task()` returns), Layer 3 (`SubagentStop` hook auto-merge). Attempt Layer 1; if the script or its path is absent, proceed — Layers 2 and 3 repair from frontmatter. An artifact without `handoff:` YAML breaks the safety net (degrades to F3 fallback: orchestrator derives a minimal handoff and logs WARN). Review-only agents (fe-performance-engineer, fe-accessibility-auditor, fe-security-auditor) do **not** patch state.json and do **not** write the stage report — they supply a ≤500-token compressed findings summary.

## Handoff Frontmatter (v4.0.0 schema)

Every stage artifact MUST start with a YAML block between `---` markers. Budgets: ≤200 tokens, ≤30 lines. Base required fields: `stage`, `verdict`, `summary` (≤200 chars), `refs`. Per-stage additions (from `handoff-protocol.md#frontmatter-schema`):

| Stage | Required beyond base | Verdict vocabulary |
|-------|----------------------|--------------------|
| DV | `files_touched`, `next_stage_focus` | ok / blocked / escalate |
| DR | `key_decisions` (= findings) | pass / fail |
| QA | `files_touched` (= tests added), `key_decisions` (= results) | go / no-go |

`key_decisions[].anchor` and `refs.*` MUST resolve to a real `## <kebab-case>` heading in the target file (anchor-lint enforces this at DR and via PostToolUse hook). Copy-paste blocks: `templates/` in this directory.

## Gate-Feedback Contract (v4.0.0)

When DR returns `verdict: fail` or QA returns `verdict: no-go`, the orchestrator re-dispatches DV (`run_index` bumped, `retry_count`++) and carries the upstream remediation **verbatim** into the retry prompt (company-workflow `worktask/SKILL.md` step 4.6). frontend-developer agents **consume** this contract; the injection is orchestrator-owned.

| Surface | Mechanism | frontend-developer action |
|---------|-----------|---------------------------|
| Orchestrator → DV prompt | On re-dispatch (`metadata.retry_count > 0`) the prompt is prepended with `REMEDIATION (from <DR\|QA> gate — fix these specific findings before re-stop:)`; `metadata.gate_from_stage` ∈ {DR, QA}; `metadata.gate_blockers[]` = DR `blockers[]` / QA `blocking_defects[]` strings. | Read both fields; fix those exact findings *first*; do not re-scope. |
| SubagentStop hook → next dispatch | A blocked gate emits `hookSpecificOutput.additionalContext` telling the next run what to fix (e.g. run `dv-screenshot-capture` when the screenshot manifest is missing). | Treat as additional remediation context; consume the same way. |

On a rework dispatch the DV/fe-code-fixer agent MUST:

1. Read `metadata.gate_from_stage` + `metadata.gate_blockers[]` (and any `REMEDIATION` block in the prompt).
2. Address each listed blocker individually; record per-blocker resolution in `.context/errors/<agent-basename>.md`.
3. Keep the diff minimal — change only what the blockers require; do not re-implement passing code.

## Qualified Agent Names

All Task delegations MUST use the fully-qualified `plugin:agent` form:

| Form | Status |
|------|--------|
| `frontend-developer:react-developer` | Required |
| `company-workflow:technical-lead` | Required |
| `react-developer` (bare) | Deprecated — back-compat shim prepends `company-workflow:` and logs a warning (would resolve to the wrong plugin) |
| `backend-developer:*` | Forward-reference handoff only (HTTP/GraphQL server, DB, auth) — **never** placed in a `tools:` `Task(...)` list; route "if installed", otherwise surface the boundary to the orchestrator |
| `apple-developer:*` | Forward-reference handoff for native-module work — same rule (see `_base/frontend-agent.md § Delegation Routing` web/native precedence) |

Task metadata carries qualified names:

```json
{
  "metadata": {
    "agent": "frontend-developer:react-developer",
    "model": "sonnet",
    "error_file": ".context/errors/react-developer.md",
    "requires_screenshots": true,
    "plan_file": "planning-0.md",
    "run_index": 0
  }
}
```

## Token Budgets

- **Incoming compressed context** (from company-workflow): 300-500 tokens (planning summary 300, architecture summary 300, development handoff 500)
- **Full stage output**: write to `.context/<stage>-N.md` (no token cap)
- **Outgoing return summary**: 500 tokens max (for the orchestrator)
- **Inter-stage handoffs**: DV→DR 300, DR→QA 300 (`company-workflow:context-compression § Context Budget by Handoff`)

## Detecting Workflow Context

1. **Context folder**: `.context/` in project root, or `.worktrees/milestone-{N}/{issue#}/.context/` in worktree mode (resolve via `task.metadata.workspace_path` + `metadata.isolation`).
2. **Plan file**: (1) `task.metadata.plan_file`; (2) newest `.context/planning-*.md`.
3. **State ledger**: read `.context/state.json` for upstream `facts`/`handoffs`/`stages` (≤500-token canonical compressed view). Legacy fallback: `metadata.context_files`.
4. **Architecture document**: newest `.context/analyzing-*.md` — or the anchors named in upstream `next_stage_focus`.
5. **Task System**: TaskList/TaskGet; inspect `task.metadata.{plan_file, agent, model, run_index, error_file, gate_from_stage, gate_blockers, requires_screenshots, workspace_path}`.

## Dynamic Worktask Sizing (v4.0.0)

PL0 assesses complexity (0-50) and creates only the stages needed:

| Score | Complexity | PL0 Creates |
|-------|------------|-------------|
| 0-10 | Low | DV0, DR0, QA0 |
| 11-20 | Medium | AR0, DV0, DR0, QA0 |
| 21-30 | Moderate | AR0, TL0, DV0, DR0, QA0 |
| 31-40 | High | AR0, TL0, DV0, DR0, QA0, DC0, FN0, ST0 |
| 41-50 | Critical | AR0, TL0, DV0, DR0, SR0, QA0, DC0, RE0, FN0, ST0 |

Security-sensitive features (authentication, payment, PII, cryptography, secrets, file uploads, untrusted HTML rendering) auto-include SR0 regardless of score.

PL0 stamps `metadata.skipped_stages = [{stage, reason}]` for every stage dropped from the full 9-stage pipeline (PL→AR→TL→DV→DR→QA→DC→FN→ST), so `state.json` self-documents the drops. It also stamps `metadata.test_mode` (`build-only` / `scoped` / `full` — defaulted by score and marker coverage) and `metadata.ui_visual_check`. Unlike the CLI-oriented sibling plugins, `ui_visual_check` **is applicable here** — web work is UI work: when PL0 sets it `true` it gates screenshot-evidence provenance (live-driven capture before comparison; the full contract lands in a later phase). The stage table above, the `test_mode` defaults, and these stamps are all defined by company-workflow `estimation-methodology § PL0 Stage-Set` (the source of truth) — keep them in lockstep with it so the next sync is a mechanical copy.

## MCP Dynamic Inheritance

Subagents inherit the parent session's MCP tools (Context7, Ref, etc.). Do not redeclare MCP tools in agent frontmatter when the parent session already provides them — redeclaration creates duplicates and bloats permission prompts.
