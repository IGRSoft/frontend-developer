# QA Stage Artifact Template (web testing)

Primary artifact `.context/testing-N.md` is owned by corpflow's qa-engineer; use this when fe-test-generator or a frontend-developer agent takes over QA or supplies the evidence body.

```markdown
---
handoff:
  stage: QA
  verdict: go           # go | no-go
  summary: "<test outcome — ≤200 chars>"
  files_touched:        # REQUIRED for QA (= tests added)
    - src/components/LoginForm.test.tsx
    - e2e/login.spec.ts
  key_decisions:        # REQUIRED for QA (= results)
    - id: q1
      summary: "<suite result — ≤160 chars, e.g. '128/128 pass; axe 0 violations; LCP 1.9s'>"
      anchor: testing-0.md#results
  open_questions: []
  refs:
    development: development-0.md#tests-added
---

# QA Testing — <worktask_id>

## results

| Suite | Command | Result | Transcript |
|-------|---------|--------|------------|
| unit/component | `npx vitest run` | 128/128 pass | .context/logs/vitest-<worktask_id>.log |
| e2e | `npx playwright test` | 24/24 pass | .context/logs/playwright-<worktask_id>.log |
| a11y (axe) | `npx playwright test --grep @axe` | 0 violations | .context/logs/axe-<worktask_id>.log |
| lighthouse | `npx lighthouse <url> --output json` | budget met | .context/logs/lighthouse-<worktask_id>.json |

Gate (all three required for `go` — workflow-integration/SKILL.md § QA Gate):
- [ ] All tests pass (full suite, not only new tests)
- [ ] axe-clean on changed views (0 violations at WCAG 2.2 AA — see _shared/accessibility-baseline.md)
- [ ] Lighthouse budget met (LCP < 2.5s, INP < 200ms, CLS < 0.1; no bundle-size budget regression)

## coverage

| Component | Tool | Line % | Target |
|-----------|------|--------|--------|
| src/components/LoginForm | vitest --coverage (c8/istanbul) | <n>% | per _shared/testing-principles.md |

## web-vitals

| Route | LCP | INP | CLS | Bundle (gzip) | Budget |
|-------|-----|-----|-----|---------------|--------|
| /login | 1.9s | 90ms | 0.02 | <n> kB | met / over |

## regressions

- <failures vs. the pre-change baseline, with suspected cause and owner, or "None">

## verdict

<go | no-go — one-line justification; on no-go list blocking defects>

Blocking defects (no-go only — these become `metadata.gate_blockers[]` verbatim on DV re-dispatch):
- <self-contained, actionable string with file:line / failing test name / axe rule id / failing vital>
```

## Notes

- The a11y and Lighthouse clauses are part of the gate, not optional garnish: run `axe-core` against the rendered changed routes (zero violations) and Lighthouse against the same routes (Core Web Vitals within budget); attach both transcript paths.
- Every transcript path in `## results` must exist under `.context/logs/`.
- Test selection for focused re-runs: `vitest -t <name>`, `playwright test -g <regex>`, `jest -t <name>`.
- Frontmatter budget: ≤200 tokens, ≤30 lines.
