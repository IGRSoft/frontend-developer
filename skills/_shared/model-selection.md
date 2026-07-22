---
name: model-selection
description: Model and effort selection for frontend-developer agents — cost tiers, per-agent assignments, and the opus+xhigh per-invocation override paths for the hardest accessibility, performance, and security work. Use when delegating to or overriding a frontend-developer specialist.
effort: low
---

# Model & Effort Selection (frontend-developer)

Companion to igrsoft's `skills/shared/model-selection.md`. This file pins the
**frontend-developer** per-agent assignments and the override paths the framework
agents expose. Frontmatter in `agents/*.md` is the source of truth — keep this
table in sync with it.

## Cost Tiers

| Model | Relative Cost | Use For |
|-------|---------------|---------|
| **haiku** | 1x (baseline) | Mechanical remediation, dependency operations, formatting, component scaffolds |
| **sonnet** | ~10x haiku | Framework/language implementation, review, test generation, routing |
| **opus** | ~50x haiku | Rendering-strategy/architecture selection, the deepest a11y/perf/security analysis |

## Effort Levels

`low` ○, `medium` ◐, `high` ●, `xhigh` ⬣.

- `xhigh` is honored **only on Opus** — Sonnet/Haiku silently fall back to
  `high`, so raising effort without raising the model is a no-op. To get `xhigh`
  reasoning you must raise **both** `model: opus` **and** `effort: xhigh`.
- Reserve `xhigh` for the hardest long-chain reasoning: rendering-architecture
  trade-offs (CSR/SSR/SSG/ISR, micro-frontend boundaries), a WCAG audit of a
  complex composite widget, root-causing an INP/LCP regression across the bundle
  and render path, or threat-modeling a cross-origin/SSR data flow.

## Per-Agent Assignment

| Agent | Model | Effort | maxTurns | Override path |
|-------|-------|--------|----------|---------------|
| `frontend-developer` (router) | sonnet | medium | 40 | — routes work to specialists |
| `react-developer` | sonnet | high | 50 | → `opus` + `xhigh` for novel RSC/data-flow architecture or a hairy hydration mismatch |
| `vue-developer` | sonnet | high | 50 | → `opus` + `xhigh` for large Composition-API refactors or reactivity-edge bugs |
| `svelte-developer` | sonnet | high | 50 | → `opus` + `xhigh` for runes-migration design across a component tree |
| `angular-developer` | sonnet | high | 50 | → `opus` + `xhigh` for signals migration or zoneless change-detection design |
| `typescript-developer` | sonnet | high | 50 | → `opus` + `xhigh` for advanced generic/type-level inference work |
| `css-developer` | sonnet | high | 50 | — sonnet sufficient for styling/layout work |
| `frontend-architector` | opus | xhigh | 60 | already top tier; self-limits scope at Low complexity per § Complexity Triage |
| `fe-test-generator` | sonnet | high | 50 | — sonnet sufficient for pattern work |
| `fe-performance-engineer` | sonnet | high | 50 | → `opus` + `xhigh` for deep Core Web Vitals / bundle trace analysis (review-only: `disallowed-tools: Write, Edit`) |
| `fe-accessibility-auditor` | sonnet | high | 50 | → `opus` + `xhigh` for auditing complex ARIA composite widgets (review-only: `disallowed-tools: Write, Edit`) |
| `fe-security-auditor` | sonnet | high | 50 | → `opus` + `xhigh` for deep XSS/CSP/SSRF threat modeling (review-only: `disallowed-tools: Write, Edit`) |
| `fe-code-fixer` | haiku | medium | 30 | — deterministic minimal-diff remediation |
| `fe-dependency-manager` | haiku | low | 20 | — mechanical lockfile/manifest operations |

## Applying an Override

Pass `model`/`effort` on the Task() call (per-invocation, does not edit
frontmatter). Callers of the three review-only auditors and the framework agents
may raise to `opus` + `xhigh` when the work spans the whole render path, a deep
component tree, or a cross-origin data flow and needs long-chain causal reasoning:

```
Task({ subagent_type: "frontend-developer:fe-performance-engineer",
       model: "opus", effort: "xhigh",
       prompt: "Root-cause the INP regression: trace the long task from the hydration
                boundary through the third-party script to the layout thrash…" })
```

```
Task({ subagent_type: "frontend-developer:fe-accessibility-auditor",
       model: "opus", effort: "xhigh",
       prompt: "Audit the combobox + listbox + tree composite against WCAG 2.2 and
                the ARIA Authoring Practices — focus order, roles, live regions…" })
```

Only override when complexity warrants it — the sonnet/high default covers the
overwhelming majority of front-end work, and `xhigh` on sonnet is a silent no-op.
All three review-only agents keep their `disallowed-tools: Write, Edit`
restriction regardless of model: fixes route to `frontend-developer:fe-code-fixer`.

## Related Skills

- `version-feature-matrix.md` — feature/version floors the chosen agent must respect
- `language-detection.md` — marker → framework → which agent to delegate to first
- `severity-matrix.md` — the P0–P3 scale a review-only agent's findings use
