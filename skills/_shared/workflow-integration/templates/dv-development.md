# DV Stage Artifact Template (web work)

Copy this to `.context/development-N.md` (`N` from `task.metadata.run_index`; e.g. `development-0.md`). H2 anchors are fixed by company-workflow's anchor allow-list — keep them exactly as written (kebab-case, H2); web sections nest as H3.

```markdown
---
handoff:
  stage: DV
  verdict: ok           # ok | blocked | escalate
  summary: "<what was implemented — ≤200 chars>"
  files_touched:        # REQUIRED for DV
    - src/components/LoginForm.tsx
    - src/components/LoginForm.test.tsx
  next_stage_focus: "<hint for DR/QA — ≤240 chars>"
  key_decisions: []
  open_questions: []
  remediation_consumed: []   # rework only (metadata.retry_count>0): gate_blockers[] addressed this run
  refs:
    plan: planning-0.md#requirements
    decisions: analyzing-0.md#decisions
---

# DV Development — <worktask_id>

## files-changed

| File | Change | Why |
|------|--------|-----|
| src/components/LoginForm.tsx | <summary> | <reason> |

### decisions

- <non-obvious implementation choice + rationale; reference analyzing-N.md anchors>

### tool-invocations

- `npm run build`
- `npx tsc --noEmit`
- `npx eslint .`
- `npx vitest run`
- `npx playwright screenshot http://localhost:4173/login .context/images/<worktask_id>/login.png`

## tests-added

| Test | Framework | Covers |
|------|-----------|--------|
| src/components/LoginForm.test.tsx | Vitest + Testing Library | <behavior> |
| e2e/login.spec.ts | Playwright | <flow> |

### build-evidence

- Toolchain + versions: <e.g. vite 5 / tsc 5.x / node 20 — verify against your project>
- `tsc --noEmit`: 0 errors
- eslint/biome: clean (zero-error)
- Bundle-size delta: <+/- kB gzip vs. baseline, or "no significant change">
- Test transcript: .context/logs/<tool>-<worktask_id>.log

## deviations

- <departures from analyzing-N.md, or "None">

## follow-ups

- <deferred work, flagged risks, or "None">
```

## Notes

- **Screenshots**: web/UI work defaults `metadata.requires_screenshots: true` — a manifest is **required**. Write it at `.context/images/<worktask_id>/screenshots.md` (rows with `source: web-adapter` from `npx playwright screenshot` / Chrome MCP; `screenshot_count` = row count) before returning, or `dv-screenshot-gate.sh` blocks `SubagentStop`. Use `source: cli-fallback` only when a route cannot render headlessly. Opt out only when `metadata.requires_screenshots: false` (non-UI change). See `workflow-integration/SKILL.md § DV Screenshot Gate` and [templates/dv-screenshots.md](dv-screenshots.md).
- `remediation_consumed:` is populated only on a rework re-dispatch — list the `metadata.gate_blockers[]` strings (from the DR/QA gate) this run fixed. See `workflow-integration/SKILL.md § Gate-Feedback Contract`.
- Frontmatter budget: ≤200 tokens, ≤30 lines. Emit it unconditionally — it is the state.json merge input regardless of filename.
- **state.json patch**: on completion run `state-patch.sh --stage DV --prev <PREV>` when its path is supplied (`task.metadata.state_patch_script`; ships under company-workflow `skills/worktask/scripts/`) to merge `stages.DV` + the `<PREV>→DV` edge from this frontmatter; if the script/`jq`/`state.json` is absent, skip — never hand-roll the merge; Layers 2/3 repair from the frontmatter. See `workflow-integration/SKILL.md § Artifact Filename Contract`.
- Tee raw build/test output to `.context/logs/` — the Build Evidence transcript path must exist on disk.
