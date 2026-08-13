# DR Stage Artifact Template (web review)

Primary artifact `.context/developer-review-N.md` is owned by corpflow's technical-lead; use this when a frontend-developer agent takes over DR or contributes the review body. fe-code-fixer appends retry narratives to `.context/errors/fe-code-fixer.md` instead.

```markdown
---
handoff:
  stage: DR
  verdict: pass         # pass | fail
  summary: "<review outcome — ≤200 chars>"
  key_decisions:        # REQUIRED for DR (= findings)
    - id: f1
      summary: "<P0 finding — ≤160 chars>"
      anchor: developer-review-0.md#findings
  files_touched: []     # only when fe-code-fixer applied fixes
  next_stage_focus: "<security surface / a11y / test focus for SR/QA — ≤240 chars>"
  refs:
    development: development-0.md#files-changed
---

# Developer Review — <worktask_id>

## findings

| ID | Priority | Area | Location | Issue | Fix |
|----|----------|------|----------|-------|-----|
| f1 | P0 | Hydration | src/app/page.tsx:42 | <issue> | <fix> |

Checked areas (web criteria — see workflow-integration/SKILL.md § DR Web Review Criteria):
- [ ] Framework anti-patterns: hook-rule violations, Vue reactivity loss, Svelte runes misuse, Angular signals/standalone discipline
- [ ] Reactivity / hydration: server/client markup mismatch, non-deterministic render, stale closures, needless client components
- [ ] Accessibility: ARIA correctness, labeled controls, keyboard/focus order, semantic elements, contrast — axe-core clean
- [ ] Render performance: unmemoized renders, oversized client bundles, layout thrash, unkeyed lists
- [ ] Type safety: no unjustified `any`, no unsafe `as`, typed props/emits/handlers, `tsc --noEmit` 0 errors
- [ ] Build hygiene: eslint/biome zero-error, lockfile current with package.json, no committed dist/node_modules, no bundle-size regression

## verdict

<pass | fail — with one-line justification tied to findings>

## blockers

- <P0/P1 findings that force verdict: fail — these become metadata.gate_blockers[] verbatim on DV re-dispatch; empty list when pass>

## follow-ups

- <P2/P3 findings deferred to backlog, or "None">
```

## Notes

- `blockers` entries are injected **verbatim** into the DV retry prompt (Gate-Feedback Contract) — write them as self-contained, actionable strings with `file:line`.
- Priorities follow `_shared/severity-matrix.md` (P0 = XSS/injection/auth bypass, P1 = hydration break/a11y blocker/broken reactivity, P2 = quality/perf regression, P3 = style).
- fe-code-fixer on a fix-application pass: populate `files_touched`, enforce minimal diff, and record per-blocker resolution in `.context/errors/fe-code-fixer.md`.
