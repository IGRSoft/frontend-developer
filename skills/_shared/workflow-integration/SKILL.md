---
name: workflow-integration
description: Guide for integrating with the igrsoft 11-stage workflow system (v3.17.0) from frontend-developer agents. Use when participating in structured workflow stages — DV, DR, SR, QA, RE — or producing handoff artifacts.
---

# Workflow Integration Guide

When invoked from the igrsoft workflow system, follow these guidelines. frontend-developer agents author web UI, so the **DV evidence model is screenshot-first** (see § DV Screenshot Gate) — the one place the contract diverges from a CLI plugin.

## 11-Stage Pipeline (Default)

```
PL → AR → TL → DV → DR → SR → QA → DC → RE → FN → ST
          ↑    ↓    ↑    ↑    ↑              ↑
   frontend-developer agents contribute to AR, DV, DR, SR, QA, and RE
```

| Code | Stage | igrsoft Agent | frontend-developer Contribution |
|------|-------|---------------|---------------------------------|
| PL | Planning | product-manager | — |
| AR | Architecture | software-architector | frontend-architector (rendering strategy, micro-frontend boundaries, client-state & design-system architecture) |
| TL | Team Lead | team-lead | — |
| DV | Development | developer → **frontend-developer** | **Primary**: frontend-developer router + react/vue/svelte/angular/typescript/css developers |
| **DR** | **Developer Review** | **technical-lead** | **Support**: fe-code-fixer, frontend-architector (pattern consult) |
| SR | Security Review | security-reviewer | Context: fe-security-auditor |
| QA | QA Testing | qa-engineer | Support: fe-test-generator |
| DC | Documentation | technical-writer | — |
| RE | Release Engineering | release-engineer | Packaging: fe-dependency-manager (lockfiles, pins) |
| FN | Finalization | project-manager | — |
| ST | Stakeholder | stakeholder | — |

## Worktask Triggers (v3.17.0)

Trigger prefixes select which stages run: `micro:` (plan→edit), `quick:` (PL→DV→DR→QA), `worktask:`/`fworktask:` (full 9-stage), `--secure`/`--full` (11-stage, adds SR + RE), `emergency:` (IR→DV→DR→QA→RE→FN). Full mapping: [references/stage-recipes.md § Worktask Triggers](references/stage-recipes.md).

## Handoff Contract (summary)

Every stage produces a numbered artifact (`<stage>-N.md`) in `.context/`, starting with a `handoff:` YAML block (≤200 tokens, ≤30 lines: `stage`, `verdict`, `summary`, `refs`, plus per-stage fields). **Emit `handoff:` frontmatter unconditionally** — it is the merge input that feeds state.json reconciliation regardless of filename. DV writes `development-N.md` with mandatory H2 anchors (`## files-changed`, `## tests-added`, `## deviations`, `## follow-ups`) and a non-negotiable **Build Evidence** block (toolchain+versions, `tsc --noEmit` → 0 errors, eslint/biome zero-error, test transcript under `.context/logs/`). All Task delegations use the fully-qualified `plugin:agent` form. Per-agent retry narratives go in `.context/errors/<agent-basename>.md` (never a shared `error.md`).

Full field tables, the artifact-filename map, the handoff-frontmatter schema, error-file derivation, qualified-name rules, token budgets, workflow-context detection, and dynamic sizing: see [references/stage-recipes.md](references/stage-recipes.md). Copy-paste templates: [templates/dv-development.md](templates/dv-development.md), [templates/dr-review.md](templates/dr-review.md), [templates/qa-testing.md](templates/qa-testing.md).

## DV Screenshot Gate (v3.12.0 — HIGHEST INTEGRATION RISK, read this)

`metadata.requires_screenshots` defaults **TRUE** (`requires_screenshots: true`) for frontend-developer DV stages — web work is UI work, so the screenshot manifest is the **default** expectation (opposite of a CLI plugin). Before a DV agent returns, it MUST write a manifest at `.context/images/<worktask_id>/screenshots.md`. If it is absent on `SubagentStop`, igrsoft's `dv-screenshot-gate.sh` **blocks** the stop and returns `hookSpecificOutput.additionalContext` telling the run to capture via `dv-screenshot-capture` — the DV agent is **re-dispatched** until the manifest exists.

Binding rules (the contract — keep these in mind even when the detail is externalized):

- Captures come through igrsoft's **`web_adapter`** path (Playwright / Chrome MCP rendered DOM). Each rendered route/state row takes **`source: web-adapter`**; a route that cannot render headlessly falls back to `source: cli-fallback` with the reason in `notes`.
- **Lighthouse** and **axe** reports are **supporting evidence**, not a substitute — attach them as supporting rows (`source: web-adapter`, `notes: lighthouse` / `notes: axe`) or reference them from Build Evidence. They corroborate the captures; they never replace a rendered screenshot of the changed UI.
- Opt out only when `metadata.requires_screenshots: false` (non-UI changes) — then no manifest is required and the gate is skipped; flag it in your return summary if metadata says otherwise. **Never** fabricate image files — the gate re-dispatches DV until a real manifest (or the `false` flag) exists.

The capture procedure, the manifest row format, and the RMSE design-diff join key (`design_ref`): see [references/screenshot-gate.md](references/screenshot-gate.md). Manifest format and example: [templates/dv-screenshots.md](templates/dv-screenshots.md).

## Stage Participation Overview

Per-stage criteria each agent satisfies or pre-checks against (full tables in [references/stage-recipes.md](references/stage-recipes.md)):

- **DV** — author web UI; write `development-N.md` with Build Evidence and the screenshot manifest.
- **DR** — technical-lead reviews (verdict `pass`/`fail`); fe-code-fixer applies fixes. DV agents pre-check framework anti-patterns, reactivity/hydration, accessibility (`axe-core` clean), render performance, type safety (`tsc --noEmit` 0 errors), build hygiene.
- **QA** — qa-engineer gates on **all three**: full suite passes, axe-clean on changed views (WCAG 2.2 AA), Lighthouse budget met (LCP < 2.5s, INP < 200ms, CLS < 0.1). fe-test-generator supports.
- **SR** — fe-security-auditor supplies web context (XSS sinks, CSP/Trusted Types, secrets-in-bundle, SSRF, npm supply-chain) to security-reviewer; review-only, findings route to fe-code-fixer. See `_shared/secure-coding/SKILL.md`.
- **RE** — release-engineer owns the stage; fe-dependency-manager freezes lockfiles/pins; framework agents produce release artifacts.

## Gate Re-Dispatch (summary)

When DR returns `fail` or QA returns `no-go`, the orchestrator re-dispatches DV (`run_index` bumped, `retry_count`++) with the upstream remediation carried **verbatim** into the retry prompt. frontend-developer agents **consume** this: read `metadata.gate_from_stage` + `metadata.gate_blockers[]` (and any `REMEDIATION` block), fix those exact findings first, record per-blocker resolution in `.context/errors/<agent-basename>.md`, keep the diff minimal. Full mechanism + SubagentStop-hook surface: [references/stage-recipes.md § Gate-Feedback Contract](references/stage-recipes.md).

## When Not in Workflow

If no workflow context is detected (no `.context/`, no task metadata), proceed with standard implementation: follow the framework skills, run the same build/type-check/lint/test discipline, capture screenshots of changed UI when practical, and report results directly — no artifacts or frontmatter required.

## Related Skills (igrsoft plugin)

`igrsoft:worktask` (worktask system), `igrsoft:cross-plugin-handoff` (handoff protocol), `igrsoft:agent-coordination` (multi-agent patterns), `igrsoft:context-compression` (token budgets), `igrsoft:security-review-process` (SR OWASP checklists), `igrsoft:release-engineering` (RE versioning).

## Related Skills (frontend-developer plugin)

`_shared/model-selection.md` (model/effort assignments), `_shared/severity-matrix.md` (P0-P3 priorities), `_shared/testing-principles.md` (test pyramid, QA coverage), `_shared/language-detection.md` (framework → agent routing), `_shared/accessibility-baseline.md` (WCAG 2.2 axe-clean gate), `_shared/secure-coding/SKILL.md` (XSS/CSP/secrets for SR), `_shared/version-feature-matrix.md` (version markers + fallbacks).
