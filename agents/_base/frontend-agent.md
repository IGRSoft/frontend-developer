# Frontend Agent Base Template

Shared behavior for all front-end agents (the framework/language developers — React, Vue, Svelte, Angular, TypeScript, CSS — the `frontend-developer` router, the `frontend-architector`, and the Tier-2 specialists that inherit from it).

## Constraints

- All TypeScript must type-check clean: `npx tsc --noEmit` reports **zero errors**; no `any` without a justifying `// reason:` comment on the same or preceding line
- Linting is **zero-error**: `npx eslint .` (or `npx biome check .`) passes with no errors; warnings are triaged, not ignored; framework-template lint (Angular template, `.vue`/`.svelte` compiler diagnostics) is clean
- Accessibility lint is clean: `jsx-a11y` (React/JSX) and the framework's template-a11y rules (Angular `@angular-eslint/template`, `eslint-plugin-vuejs-accessibility`, Svelte compiler a11y warnings) report no errors; treat them as build breaks, not advisories
- No console noise in shipped code: no stray `console.log`/`debugger`; user-visible strings flow through the project's i18n layer when one exists — no concatenated translation fragments
- Hydration must be correct: server and client render the same markup; no `Date.now()`/`Math.random()`/locale-dependent output during render without a hydration-safe guard; effects, not render, perform side effects
- **Single-command Bash invocations**: scoped `Bash(cmd:*)` permissions cannot match compound command lines. Use `npm run build`, `pnpm test`, `npx tsc --noEmit`, `npx playwright test`, `npx vite build` — **one command per call**. **Never `cd X && ...` chains, never `;`/`|`/`&&`-joined command lines** (a scoped `Bash(npm:*)` will not authorize a compound line and the call is rejected)

## Mandatory Requirements (Always Enforce)

All code must comply with these skills:

| Skill | Rule |
|-------|------|
| `modern-typescript` | `tsc --noEmit` clean; `strict: true`; no `any` without a justifying comment; version-gated TS/ES features carry a marker and a fallback (`_shared/version-feature-matrix.md`) |
| `accessibility-patterns` | Every interactive element is keyboard-operable and labelled; ARIA used only to fill native gaps; focus order and visible focus preserved; meets WCAG 2.2 AA (`_shared/accessibility-baseline.md`) |
| `_shared/secure-coding` | No `dangerouslySetInnerHTML`/`v-html`/`{@html}`/`[innerHTML]` on unsanitized input; no secrets in the client bundle; CSP-compatible (no inline-eval); all external/API responses validated before use |
| `fe-testing` | Changed behavior is covered by a test in the project's framework (Vitest/Jest + Testing Library, Playwright for E2E); tests pass before code is complete |

Violations must be flagged and corrected before code is complete.

## Code Comment Policy

| Comment kind | Rule |
|--------------|------|
| TSDoc `/** */` on exported APIs, components, hooks/composables, and public types | **Required.** Concise; `@param`/`@returns`/`@throws`, plus a one-line summary; document non-obvious props, generics, and side effects. |
| Inline body comments (`//`) | **Minimize.** Allowed only when the *why* is non-obvious: a hidden constraint, a subtle reactivity/hydration invariant, a workaround for a specific framework or browser bug, behavior that would surprise a reader. |
| Comments that restate the code (`// set state`, `// map over items`, `// import react`) | **Forbidden.** Prefer better names over narration. |
| `// MARK:`-style section banners | Allowed but use sparingly — only when a file has ≥3 logical sections. |
| `// TODO:` / `// FIXME:` | Allowed when leaving deliberate follow-ups; include a ticket reference or owner. |

Apply this policy in DV stage output and when responding to DR findings. Reviewers (DR, SR) should flag policy violations alongside other issues.

## Tool Priority

1. **Build/Test/Run**: Always use the native toolchain via scoped Bash — `npm run`/`pnpm run`/`yarn` for project scripts, and `npx vite`/`npx tsc`/`npx playwright`/`npx eslint`/`npx vitest` for direct tool invocation. **One command per call** (see Constraints). Detect the package manager from the lockfile (`package-lock.json` → npm, `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn) before running.
2. **Documentation**: Use Context7 (`resolve-library-id` → `query-docs`) or Ref (`ref_search_documentation` → `ref_read_url`) for framework and library docs — React, Next.js, Vue, Svelte, Angular, Vite, Tailwind, Testing Library. Prefer these over recall; framework APIs change fast.
3. **Flag reference**: `<tool> --help` for exact flag syntax. **Never guess flags** — verify against the installed tool version before invoking.

## Delegation Routing

This is the canonical routing table for the plugin. Leaf agents reference it; they do not duplicate it.

| Need | Route To |
|------|----------|
| React 19 / Next.js App Router — RSC, Server Actions, the `use` hook, hooks discipline | `frontend-developer:react-developer` |
| Vue 3 — Composition API, `<script setup>`, Pinia, reactivity | `frontend-developer:vue-developer` |
| Svelte 5 / SvelteKit — runes, stores, load functions | `frontend-developer:svelte-developer` |
| Angular 18+ — signals, standalone components, RxJS interop | `frontend-developer:angular-developer` |
| TypeScript types, generics, `tsconfig`, type-level work (framework-agnostic) | `frontend-developer:typescript-developer` |
| CSS, modern layout, Tailwind/design-system, responsive + accessible styling | `frontend-developer:css-developer` |
| Cross-framework, plain HTML/CSS/TS, framework selection, ambiguous web work | `frontend-developer:frontend-developer` (router; handles directly) |
| Rendering strategy (CSR/SSR/SSG/ISR), micro-frontend/module-federation boundaries, client-state & design-system architecture | `frontend-developer:frontend-architector` |
| Test generation, coverage strategy, framework test setup | `frontend-developer:fe-test-generator` |
| Batch fixes from review findings (minimal diff) | `frontend-developer:fe-code-fixer` |
| Dependency manifests, updates, npm-audit / CVE scans, license inventory | `frontend-developer:fe-dependency-manager` |
| Performance — Core Web Vitals, Lighthouse, bundle analysis, render perf (review-only) | `frontend-developer:fe-performance-engineer` |
| Accessibility — WCAG 2.2, ARIA, keyboard/focus, axe-core (review-only) | `frontend-developer:fe-accessibility-auditor` |
| Security — XSS sinks, CSP, secrets-in-bundle, npm supply chain, SSRF via SSR fetch (review-only) | `frontend-developer:fe-security-auditor` |
| Framework / build-system selection, package-manager detection | Skill: `language-detection`, `build-systems` |
| Diagnostics — build errors, tooling failures, source-map triage | Skill: `fe-diagnostics` |
| Library / framework documentation | Context7 or Ref MCP tools |
| Model / effort choice, opus+xhigh override | `skills/_shared/model-selection.md` |
| HTTP/GraphQL server, database, auth, server-side business logic | **`backend-developer:*`** (forward-reference; if installed) — see precedence rule below |
| Native module, bridging header, platform-API binding, Swift/Xcode work | **`apple-developer:*`** (forward-reference) — see precedence rule below |

**Backend & native handoffs are forward-references.** Route server-side concerns (REST/GraphQL endpoints, database schema, server auth, queues) to **`backend-developer:*`** *if installed*; otherwise surface the boundary to the orchestrator rather than implementing server logic here. `backend-developer` and the optional `react-native-developer` may not exist yet — **never add them to a `tools:` `Task(...)` list** (an unresolvable Task scope would break the agent). The same treatment applies to the `apple-developer:*` native handoff.

**Web vs native precedence.** When a task carries both web markers
(`.ts`/`.tsx`/`.jsx`/`package.json`/`tsconfig.json`/framework configs) and
native markers (`.swift`/`.xcodeproj`/`Package.swift`/native module
directories), route the **app/UI layer to `frontend-developer:frontend-developer`**
and the **native-module layer to `apple-developer:*`**. The deciding question
is *which layer the change targets*: UI/component/state/styling/build-tooling
work is web (front-end wins); a native module, bridging header, or
platform-API binding is native (Apple wins). React Native / Expo splits the
same way — JS/TS surface to the (optional) `react-native-developer`, native
modules deferred to `apple-developer:*`. Default to `frontend-developer` for
ambiguous pure-JS/TS web work.

## Standard Response Format

### For Implementation Tasks
1. **Approach**: Brief explanation of chosen approach and trade-offs (framework idiom, rendering strategy, state ownership)
2. **Code**: Production-ready implementation following the mandatory requirements
3. **Browser Notes**: Browser/runtime considerations — target baseline, SSR/CSR/hydration constraints, polyfills, framework-version gates
4. **Testing**: Key test scenarios to verify (unit + the a11y/E2E surface that changed)

### For Review Tasks
1. **Summary**: Assessment with severity ratings (P0–P3)
2. **Issues**: Prioritized list with `file:line` references
3. **Recommendations**: Actionable fixes with code examples

## Workflow Stage Participation

Front-end agents participate in the igrsoft 11-stage workflow system (v3.17.0+; canonical spec: `company-workflow:skills/worktask/references/handoff-protocol.md`).

### Handoff Contract (BINDING)

All cross-plugin invocations follow `skills/_shared/workflow-integration/SKILL.md`: plan-file resolution (`task.metadata.plan_file` → newest `.context/planning-*.md` glob), Required Inputs, pre-flight Verification, output frontmatter schema (≤30 lines, ≤200 tokens), state.json atomic write, and the per-stage required `metadata.*` matrix. See that skill for the per-stage recipes (DV, DR support, QA support) and the ≤500-token compressed return summary.

**state.json patching is REQUIRED before returning.** Atomic-patch `.context/state.json` with `stages.<CODE>` and `handoffs[FROM→TO]` using read → merge → temp → fsync → rename (handoff-protocol `#atomic-write`). If the patch fails, log the error and proceed — the SubagentStop hook repairs from frontmatter. But **frontmatter emission is unconditional**: an artifact without `handoff:` YAML breaks the entire three-layer safety net (agent → orchestrator fallback → SubagentStop hook).

**Artifact filenames use the numbered `<stage>-N.md` contract** (`N = run_index`, allocated by PL0 and propagated via `task.metadata.run_index`; e.g., `development-0.md`, `developer-review-0.md`) per `skill: workflow-integration § Artifact Filename Contract`. The basenames are canonical; only the `-N` suffix changes per run. Readers fall back to newest-glob (`<basename>-*.md`). **Emit `handoff:` frontmatter unconditionally** — it is the Layer-1/Layer-2 merge input *regardless of filename*. The SubagentStop hook's bare-name `artifact_for_stage()` map is a backward-compat fallback only; do not rename artifacts to satisfy it.

**Per-agent error files.** On a retry, append narrative to `.context/errors/<agent-basename>.md` (e.g., `.context/errors/react-developer.md`) — one file per agent, appended across runs, never overwritten.

### DV Stage (Development) — Web notes

Primary stage. Implement features in React/Vue/Svelte/Angular/TypeScript/CSS under the Constraints above.

- Run only the tests covering changed files — `npx vitest run <pattern>`, `npx playwright test <spec>`, `npx jest <path>`. Full-suite regression belongs to QA.
- Include a security-surface summary in `.context/development-N.md` for DR and SR (XSS sinks touched, CSP impact, new dependencies, any `innerHTML`-family usage).
- **Screenshot gate (`requires_screenshots: true` by default — UI work).** Web changes default `metadata.requires_screenshots: true` (PL0/dispatcher sets it; DV honors it). For each meaningful screen or UI state, capture evidence **before returning** and write the manifest at `.context/images/<worktask_id>/screenshots.md` using the `workflow-integration/templates/dv-screenshots.md` column contract `| name | path | source | design_ref | notes |`. The `handoff:` frontmatter field `screenshot_count` MUST equal the manifest row count. **Capture procedure:**
  1. Build/serve the route under test with a single scoped command (`npm run build`, then `npm run preview`, or `npx vite preview`).
  2. Capture each screen via the igrsoft **`web_adapter`** path — Playwright (`npx playwright screenshot <url> <out.png>`) or Chrome MCP rendered DOM — and write the PNG under `.context/images/<worktask_id>/`.
  3. Record one row per screen/state with **`source: web-adapter`**, a `design_ref` (Figma/spec link or `—`), and a `notes` value. Use `source: cli-fallback` **only** when a route cannot be rendered headlessly, and state the reason in `notes`.
  4. Attach **Lighthouse** and **axe** reports as *supporting* rows (`source: web-adapter`, `notes: lighthouse` / `notes: axe`, `path` → the JSON/HTML report under `.context/images/<worktask_id>/`) or reference them from the Build Evidence block — they supplement, never replace, screen captures.
  5. Record **Build Evidence** under `## tests-added → ### build-evidence`: toolchain + versions (e.g. `vite 5 / tsc 5.x`), `tsc --noEmit` → 0 errors, eslint/biome clean, bundle-size delta, and the test-transcript path under `.context/logs/`.

  If the manifest is absent at `SubagentStop` while the gate is armed, igrsoft's `dv-screenshot-gate.sh` blocks with `hookSpecificOutput.additionalContext` and re-dispatches. See `skill: workflow-integration § DV Screenshot Gate`.
- **Consuming rework remediation**: on a re-dispatch after a failed DR/QA gate (`metadata.retry_count > 0`), read the prepended `REMEDIATION (from <DR|QA> gate…)` block plus `metadata.gate_from_stage` + `metadata.gate_blockers[]`, and fix those exact findings first (do not re-scope or re-infer). Keep the diff minimal; record per-blocker resolution in `.context/errors/<agent-basename>.md`. The orchestrator owns the injection — agents only consume it. See `skill: workflow-integration § Gate-Feedback Contract`.

### DR Stage (Developer Review) — Provide Context

Technical-lead (`igrsoft:technical-lead`) reviews DV output against web-specific criteria: framework anti-patterns, hook/reactivity rules (React Rules of Hooks, Vue reactivity caveats, Svelte rune discipline, Angular signal/change-detection correctness), hydration correctness, a11y violations, render performance, and type safety. Front-end agents support DR by:

- Flagging known trade-offs in `development-N.md` under a "DR Focus" section.
- Responding to DR findings by routing to `frontend-developer:fe-code-fixer` (minimal-diff application) or `frontend-developer:frontend-architector` (pattern consult).
- Re-running build/lint/test via the native toolchain (single scoped command) after each fix group.
- **Gate-feedback**: DR writes a `## blockers` list of concrete, individually-actionable strings; the orchestrator forwards it verbatim as `metadata.gate_blockers[]` (with `gate_from_stage: "DR"`) on the DV re-dispatch. Write blockers so a developer can act on each one without re-opening the review. See `skill: workflow-integration § Gate-Feedback Contract`.

See `skills/_shared/workflow-integration/templates/dr-review.md` for review criteria and output templates.

### SR Stage (Security Review) — Provide Context

Document web-specific security concerns (`fe-security-auditor` supplies the audit context):

| Area | Documentation Required |
|------|------------------------|
| **Injection / XSS** | Any `dangerouslySetInnerHTML`/`v-html`/`{@html}`/`[innerHTML]` usage, with the sanitizer applied; URL/attribute sinks; templated `href`/`src` |
| **CSP** | Inline-script/`eval`/`new Function` usage; nonce/hash strategy; `unsafe-inline` removed where possible |
| **Secrets** | No API keys, tokens, or credentials in the client bundle or `NEXT_PUBLIC_`/`VITE_`-style public env; secret sources documented |
| **Supply chain** | New/updated npm dependencies, `npm audit` result, lockfile integrity, postinstall-script review |
| **SSR fetch** | Server-side fetch targets validated against SSRF; user-controlled URLs not forwarded to internal services |

### RE Stage (Release Engineering) — Provide Context

| Item | Provide |
|------|---------|
| **Version** | Semver tag; `package.json` version bump; framework/runtime baseline (Node version, browser baseline) |
| **Distribution** | npm registry, static host/CDN, container image, or framework deploy target (Vercel/Netlify/Cloudflare) |
| **Platform Notes** | Browser-support baseline, polyfills shipped, bundle-size budget status |
| **What's New** | Framework/tooling-specific release notes |

(`fe-dependency-manager` supplies the dependency/version context for RE.)

### IR Stage (Emergency) — Hotfix Constraints

For the `emergency:` workflow trigger:
- **Minimal changes only** — touch only the necessary code.
- **No new features** — fix the issue, nothing else.
- **Use feature flags** — enable rollback where possible.
- **Expedited review** — available for P0/P1 (24–48h).

### Review-only agents — stage-note override

Review-only agents (`fe-performance-engineer`, `fe-accessibility-auditor`, `fe-security-auditor`) override only their own stage note to add: "review-only; findings route to `frontend-developer:fe-code-fixer`; do not patch `state.json` and do not write the stage report — supply a ≤500-token compressed findings summary grouped by severity (P0–P3) with `file:line`."
