---
description: Develop a web feature end-to-end — architecture, framework implementation, tests, and a security pass, build-gated
argument-hint: [feature name or issue reference] [--framework react|vue|svelte|angular] [--methodology tdd|traditional] [--resume]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
estimated-cost:
  min-tokens: 8000
  max-tokens: 45000
  model-distribution:
    haiku: 10%
    sonnet: 60%
    opus: 30%
---

# Feature Development
<!-- Updated: June 2026 -->

Take a web feature from a one-line request to reviewed, tested, security-checked code. The chain is fixed: `frontend-developer:frontend-architector` designs it, the framework developer selected by detection implements it, `frontend-developer:fe-test-generator` covers it, and `frontend-developer:fe-security-auditor` audits it. Every phase boundary is a `/frontend-developer:build-test` gate plus a written artifact, and the phases that change direction stop for your approval.

[Extended thinking: The failure mode of "build me this feature" is a single long agent turn that designs, implements, and self-certifies in one breath — no seam to review, no artifact to resume from, and a security pass that is really the implementer marking its own homework. This command splits the work into four phases with distinct owners, each writing a file under `.context/.feature-dev/` that the next phase reads instead of trusting the context window. Detection picks the implementer once, from real markers, so a Nuxt feature never lands with a React reviewer. The build gate between phases means a phase can only hand forward a green tree, and the security pass is a separate read-only agent that never wrote the code it audits. Checkpoints exist at exactly the two points where continuing is expensive to undo: after the design, and before the security-driven remediation lands.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Execute phases in order.** Do NOT skip ahead, reorder, or merge phases. Design precedes implementation; implementation precedes tests; tests precede the security pass.
2. **Write the artifact for every phase.** Each phase MUST produce its file under `.context/.feature-dev/` before the next phase begins, and each phase reads its inputs from the prior phase's file — NOT from context-window memory.
3. **Stop at every `PHASE CHECKPOINT`.** Present the phase's findings and wait for explicit user approval before continuing. Do not self-approve.
4. **Build gate between phases (BINDING).** Run `/frontend-developer:build-test` at each phase boundary. A red gate blocks the hand-off: report the failing stage and the first error, and do NOT begin the next phase on a broken tree.
5. **Halt on failure.** If an agent errors, a required file is missing, or a gate stays red after one bounded corrective pass, STOP, present the error, and ask how to proceed. Never silently continue.
6. **Detection selects the implementer — once.** Resolve the framework from real markers per `skill: language-detection` at pre-flight and record it in `state.json`. `--framework` overrides it. Do NOT re-detect per file or route a second framework developer for a single-framework feature.
7. **The security pass never audits its own code.** `frontend-developer:fe-security-auditor` is read-only. Its findings route back to the implementing framework developer (or `frontend-developer:fe-code-fixer` for mechanical fixes) — the auditor does not patch.
8. **Tool-missing never hard-fails.** If an optional tool (linter, e2e runner, scanner) is unavailable, print the install hint, skip that step, note the reduced depth, and continue. Only a missing Node toolchain — which makes the build gate impossible — is a hard stop.
9. **Commands route, they do not orchestrate.** This command names which `frontend-developer:*` agent owns each phase so Claude routes the work; it does not call `Task` itself.
10. **Never enter plan mode.** This command IS the plan — execute it.

## Usage

```bash
# Full feature run from a description
/frontend-developer:develop-feature "multi-step checkout with saved addresses"

# From an issue reference
/frontend-developer:develop-feature "#412 dark-mode toggle with system preference"

# Force the framework when a monorepo makes detection ambiguous
/frontend-developer:develop-feature "notification centre" --framework vue

# Test-first: fe-test-generator writes failing tests before implementation
/frontend-developer:develop-feature "coupon validation form" --methodology tdd

# Resume an interrupted run from its state file
/frontend-developer:develop-feature --resume
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `feature` | required | Feature name, description, or issue reference. Drives every phase brief. |
| `--framework react\|vue\|svelte\|angular` | auto | Force the implementing developer instead of detecting. Use for monorepos or a greenfield directory with no markers yet. |
| `--methodology tdd\|traditional` | `traditional` | `tdd` runs `fe-test-generator` before implementation to author failing tests, then implements against them. `traditional` implements first, then covers. |
| `--resume` | off | Read `.context/.feature-dev/state.json` and continue from `current_phase` instead of starting over. |

## Pre-flight Checks

### 1. Check for an existing session

Read `.context/.feature-dev/state.json` if present:

- `status: "in_progress"` → display the current phase and ask whether to resume or start fresh. `--resume` answers this non-interactively.
- `status: "complete"` → ask whether to archive it (`.context/.feature-dev/archive/{timestamp}/`) and start fresh.

### 2. Detect the framework

Resolve the implementing agent from the markers in `package.json` plus file extensions, per the canonical table in `skill: language-detection`. Summary:

| Markers | Framework | Implementing agent |
|---------|-----------|--------------------|
| `next` / `react` + `*.tsx`, `*.jsx` | Next.js / React | `frontend-developer:react-developer` |
| `nuxt` / `vue` + `*.vue` | Nuxt / Vue | `frontend-developer:vue-developer` |
| `@sveltejs/kit` / `svelte` + `*.svelte` | SvelteKit / Svelte | `frontend-developer:svelte-developer` |
| `@angular/core` + `*.component.ts` | Angular | `frontend-developer:angular-developer` |
| `*.ts` with no framework markers (lib, SDK, utilities) | TypeScript | `frontend-developer:typescript-developer` |
| Feature is styling/design-system work (`*.css`, `*.scss`, tokens, theming) | CSS | `frontend-developer:css-developer` |

- A feature that is substantially both component work and design-system work routes the framework developer as the owner and `frontend-developer:css-developer` as a second implementer for the styling layer only.
- Confirm every version-gated API the design intends to use against `skill: version-feature-matrix` before the design is approved.

### 3. Initialize state

Create `.context/.feature-dev/` and `state.json`:

```json
{
  "command": "$ARGUMENTS",
  "feature": "...",
  "status": "in_progress",
  "framework": "react",
  "implementer": "frontend-developer:react-developer",
  "methodology": "traditional",
  "current_phase": 1,
  "completed_phases": [],
  "artifacts": [],
  "started_at": "ISO_TIMESTAMP",
  "last_updated": "ISO_TIMESTAMP"
}
```

Update `state.json` after every phase and every gate.

## Artifacts

| Phase | Artifact | Written by |
|-------|----------|------------|
| 1 | `.context/.feature-dev/01-design.md` | `frontend-developer:frontend-architector` |
| 2 | `.context/.feature-dev/02-implementation.md` | the implementing framework developer |
| 3 | `.context/.feature-dev/03-tests.md` | `frontend-developer:fe-test-generator` |
| 4 | `.context/.feature-dev/04-security.md` | `frontend-developer:fe-security-auditor` |
| all | `.context/.feature-dev/state.json` | this command |
| gates | `.context/logs/build-{timestamp}.log` | `/frontend-developer:build-test` |

## Phase 1: Design

**Owner: `frontend-developer:frontend-architector`** (read-only — it designs, it does not implement).

Brief:

> "Design the front-end architecture for this feature: **{feature}**. Framework: {framework}. Read the existing code to match its conventions. Produce: (1) component tree and the container/presentational split; (2) state ownership — what is server state vs client state, which store or context owns each slice, and why (see `skill: react-state` / `skill: vue-state`); (3) data flow and the API/loader contract, including loading, empty, and error states; (4) routing and code-splitting boundaries; (5) the public props/types of each new component; (6) accessibility requirements up front — semantics, focus management, keyboard path (floor: `skill: accessibility-baseline`); (7) performance budget for the feature (`skill: web-performance`); (8) version gates for every API used, per `skill: version-feature-matrix`; (9) an ordered build sequence of small units, leaf-first, with a done-criterion each; (10) risks and the alternatives you rejected. Do NOT write feature code. Write the design to `.context/.feature-dev/01-design.md`."

If the feature needs an architecture-pattern decision rather than a component design (state library selection, monorepo boundary, rendering strategy), run `/frontend-developer:arch-select` first and feed its output in as an input.

**Gate:** run `/frontend-developer:build-test` to establish a green baseline before any edit. A tree that is already red is reported now, not blamed on the feature later.

---

### PHASE CHECKPOINT

**Completed:** Phase 1 — Design (component tree, state ownership, data flow, a11y and performance requirements, build sequence).

**Next:** Phase 2 — Implementation by `{implementer}`.

Stop here. Present the design summary, the build sequence, and the baseline gate result, and ask the user to approve proceeding.

---

## Phase 2: Implementation

**Owner: the framework developer selected at pre-flight** (`react-developer` / `vue-developer` / `svelte-developer` / `angular-developer` / `typescript-developer` / `css-developer`).

With `--methodology tdd`, `frontend-developer:fe-test-generator` runs FIRST for each unit — authoring failing tests against the design's stated contracts — and the implementer then makes them pass. With `traditional`, implementation comes first and Phase 3 covers it.

Brief:

> "Implement the feature **{feature}** following the approved design in `.context/.feature-dev/01-design.md`. Work through the design's build sequence one unit at a time, leaf-first. For each unit: match the repo's existing conventions; implement loading, empty, and error states — not just the happy path; wire the accessibility requirements from the design (semantics, labels, focus management, keyboard operability); keep state ownership exactly where the design put it; gate every version-specific API on `skill: version-feature-matrix`. Do not exceed the design's scope — if the design is wrong, stop and report it rather than redesigning mid-implementation. Record every file created/modified and every deviation from the design in `.context/.feature-dev/02-implementation.md`."

Styling layer, when the feature has one: route it to `frontend-developer:css-developer` with the design's token/theming constraints (`skill: modern-css`, `skill: tailwind-design-system`).

**Gate:** run `/frontend-developer:build-test` after the last unit (and after each unit for a large feature). Red → hand the first error plus ~10 lines of context back to the implementer for one bounded corrective pass, then re-gate. Still red → halt (Rule 5).

## Phase 3: Tests

**Owner: `frontend-developer:fe-test-generator`.**

Brief:

> "Generate and register tests for the feature **{feature}**, using the project's existing runner and renderer — do NOT introduce a second framework (see `skill: fe-testing`). Inputs: the design contracts in `01-design.md` and the implemented files in `02-implementation.md`. Cover: component render and prop contracts; user interaction paths; the loading/empty/error states the design specifies; edge cases and failure modes (rejected fetch, empty list, permission denied); accessibility assertions on the keyboard path and accessible names; and one e2e path for the feature's primary user journey if the repo already has an e2e runner. Follow the test pyramid and coverage floors in `skill: testing-principles`. Registration is part of the deliverable — tests the runner does not discover do not count. Write the coverage summary and any deliberate gap to `.context/.feature-dev/03-tests.md`."

Under `--methodology tdd` this phase becomes a verification pass: confirm the pre-written tests now pass and fill only the gaps the implementation revealed.

**Gate:** run `/frontend-developer:build-test`. The suite must be green and the new tests must be discovered by the runner (`npx vitest list` / `npx playwright test --list`). A test suite that does not run is a Phase 3 failure, not a pass.

## Phase 4: Security Pass

**Owner: `frontend-developer:fe-security-auditor`** (read-only — it audits, it never patches).

Brief:

> "Read-only web-security review of the feature **{feature}** (framework: {framework}). Scope: the files listed in `02-implementation.md` and `03-tests.md`. Cover XSS sinks (`dangerouslySetInnerHTML`, `v-html`, `{@html}`, `innerHTML`, `eval`), unsanitized URL/`href` injection, CSP gaps, secrets or API keys reaching the client bundle, unsafe `target=_blank` without `rel`, SSRF via SSR/loader fetches, auth/authorization decisions made only on the client, CSRF on state-changing requests, prototype pollution, unsafe deserialization of API data, and supply-chain risk in any dependency this feature added. Map each finding to a CWE and rank it with `skill: severity-matrix` (see `skill: secure-coding` for the rule set). Do NOT edit. Write findings to `.context/.feature-dev/04-security.md` as `{file, line, CWE, severity, why, fix, confidence}` — and say so plainly if the feature is clean."

Remediation routing (Rule 7): P0/P1 findings go back to the implementing framework developer; purely mechanical fixes (add `rel="noreferrer"`, remove a stray console log with a token) go to `frontend-developer:fe-code-fixer` under a minimal-diff gate. Re-run the gate after remediation, then re-run the auditor on the touched files only.

---

### PHASE CHECKPOINT

**Completed:** Phase 4 — Security pass (findings, severities, proposed remediation).

**Next:** remediation of P0/P1 findings, then the final report.

Stop here. Present the findings table and the proposed routing, and ask the user to approve the remediation (or to accept the risk and finish).

---

## Phase 5: Report

Emit the Output Format, set `state.json` `status` to `"complete"`, and list the artifacts. Optional follow-ups worth naming in the report: `/frontend-developer:analyze-accessibility` for a full axe/Lighthouse audit, `/frontend-developer:fix-performance` if the feature moved a Core Web Vital, and `/frontend-developer:gen-docs` for the feature's documentation.

## Success Criteria

- Every design contract in `01-design.md` is implemented or explicitly deferred with a reason.
- Loading, empty, and error states exist — not just the happy path.
- The build gate is green: type-check, build, and the full test suite.
- New tests are discovered by the runner and cover interaction, edge cases, and the a11y path (floors per `skill: testing-principles`).
- Accessibility floor met per `skill: accessibility-baseline` (accessible names, keyboard operability, focus visibility).
- No P0/P1 security finding remains open, or each is explicitly accepted by the user.
- All four artifacts exist under `.context/.feature-dev/`.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect |
|--------------|--------------|--------|
| `node` / `npm` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | HARD STOP — the build gate is impossible; report and exit before Phase 2 |
| `pnpm` / `yarn` / `bun` | `npm install -g pnpm` (or `corepack enable`) | fall back to the next eligible manager per `/frontend-developer:build-test` |
| `vitest` / `jest` | `npm install -D vitest` | Phase 3 scaffolds the runner config and announces it; note reduced depth |
| `@playwright/test` | `npm install -D @playwright/test && npx playwright install` | skip the e2e journey; component + unit coverage continues |
| `@axe-core/cli` | `npm install -D @axe-core/cli` | a11y assertions stay static-only; note reduced depth |
| `eslint` | `npm install -D eslint` | implementation and security passes run without lint signal |
| `gh` (issue reference) | `brew install gh` then `gh auth login` | treat the issue reference as plain text; ask the user for the requirements |

Every row except the Node toolchain is skip-with-note: print the hint, record the reduced depth in the report, and continue.

## Output Format

```markdown
## Feature Development Report

**Feature:** {feature}
**Framework:** {detected or forced} | **Implementer:** frontend-developer:{agent}
**Methodology:** {traditional | tdd}
**State:** .context/.feature-dev/state.json

| Phase | Owner | Result | Artifact |
|-------|-------|--------|----------|
| 1 Design | frontend-architector | ✅ | 01-design.md |
| 2 Implementation | {implementer} | ✅ | 02-implementation.md |
| 3 Tests | fe-test-generator | ✅ | 03-tests.md |
| 4 Security | fe-security-auditor | ✅ | 04-security.md |

### Build Gates
| After phase | Type-check | Build | Tests | Log |
|-------------|-----------|-------|-------|-----|
| baseline | ✅ | ✅ | ✅ | build-{ts}.log |
| 2 | ✅ | ✅ | ✅ | build-{ts}.log |
| 3 | ✅ | ✅ | {N passed} | build-{ts}.log |

### Delivered
- **Components/modules:** {list}
- **Tests:** {N} across {unit/component/e2e}
- **Deviations from the design:** {list, or "none"}

### Security
| File:Line | CWE | Severity | Status |
|-----------|-----|----------|--------|
| {file}:{line} | {CWE} | P0-P3 | fixed / accepted / open |

**Result:** COMPLETE / PARTIAL / HALTED ({phase, reason})

<!-- when a tool was missing -->
### Reduced-Depth Notes
- {phase}: {tool} unavailable — install: {hint}.
```

## Error Handling

### No feature description
```
Error: A feature name or description is required.
Suggestion: /frontend-developer:develop-feature "multi-step checkout with saved addresses"
```

### Framework detection ambiguous
```
Note: Multiple frameworks detected in scope: {list}.
Suggestion: re-run with --framework <react|vue|svelte|angular>, or point the command at one workspace package.
```

### Baseline build already red
```
Error: The build gate is red before any feature code was written.
First error: {one-line}. Log: {path}.
Suggestion: fix the existing failure (or run /frontend-developer:debug) before starting the feature.
```
Do not start Phase 2 on a red baseline.

### Gate red after a phase
Hand the first error plus ~10 lines of context back to that phase's owner for one bounded corrective pass, then re-gate. Still red → halt, report the phase and the log path, and do not begin the next phase (Rules 4-5).

### Missing artifact on resume
```
Error: --resume expected {artifact} for phase {n}, which is missing.
Suggestion: re-run that phase, or start fresh (the old session archives to .context/.feature-dev/archive/).
```

### Design rejected at the checkpoint
Not an error. Capture the user's objection, route it back to `frontend-developer:frontend-architector` for a revised `01-design.md`, and re-present the checkpoint. Do not implement an unapproved design.

## See Also

- `skill: language-detection` — canonical marker → framework → agent routing used at pre-flight.
- `skill: version-feature-matrix` — version gates for every API the design adopts.
- `skill: testing-principles` — test pyramid and coverage floors Phase 3 must meet.
- `skill: fe-testing` — runner/renderer selection and per-framework test patterns.
- `skill: secure-coding` — the rule set behind the Phase 4 audit.
- `skill: severity-matrix` — P0-P3 ranking for security and review findings.
- `skill: accessibility-baseline` — the WCAG 2.2 floor the design and implementation must clear.
- `skill: web-performance` — Core Web Vitals budget set in the design.
- `/frontend-developer:build-test` — the binding gate at every phase boundary.
- `/frontend-developer:arch-select` — run first when the feature needs a pattern or state-library decision.
- `/frontend-developer:arch-review` — review the delivered structure against the chosen pattern.
- `/frontend-developer:review-code` — full multi-lens review of the feature diff before merge.
- `/frontend-developer:gen-component` — scaffold a single component instead of a whole feature.
- `/frontend-developer:analyze-accessibility` — escalate the a11y floor to a full audit.
- `/frontend-developer:fix-performance` — when the feature regresses a Core Web Vital.
- `/frontend-developer:gen-docs` — document the delivered feature.
- `/frontend-developer:debug` — triage a failing gate that resists one corrective pass.
