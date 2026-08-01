# Frontend Developer Plugin

Claude Code plugin for **web front-end** development — React/Next.js, Vue/Nuxt, Svelte/SvelteKit, Angular, TypeScript, and modern CSS/Tailwind — with specialized agents, commands, and skills. Collaborates with the company-workflow plugin for full 11-stage workflow orchestration (PL→AR→TL→DV→**DR**→SR→QA→DC→RE→FN→ST). UI work defaults to `requires_screenshots: true`; DV evidence is captured via the company-workflow `web_adapter` (Playwright / Chrome MCP) as screenshots, with Lighthouse and axe reports attached as supporting evidence.

## Compatibility

| Field | Value |
|-------|-------|
| **Version** | 1.3.0 |
| **company-workflow Compatibility** | v4.0.0 |
| **claude-code min version** | 2.1.169 |

## Installation

```sh
# Add the marketplace (once), then install the plugin.
/plugin marketplace add IGRSoft/frontend-developer
/plugin install frontend-developer
```

Or, if developing locally from a checkout:

```sh
/plugin marketplace add /absolute/path/to/frontend-developer
/plugin install frontend-developer
```

After installation, restart the session so hooks (`PostToolUse`, `SubagentStop`, `PreCompact`) and the `frontend-developer:*` agents register. The plugin auto-collaborates with `company-workflow` when present; agents and commands degrade gracefully when an optional toolchain (eslint, axe-core, Lighthouse, Playwright) is missing — they print an install hint and skip, never hard-failing.

Validate a local checkout before release:

```sh
bash scripts/validate.sh          # ERROR-only gate
bash scripts/validate.sh --strict # WARN also fails (CI gate)
```

## Agents

14 named agents (the `agents/_base/frontend-agent.md` inheritance surface is shared and not invoked directly). Every `Task(...)` reference is fully qualified as `frontend-developer:<agent>`.

| Agent | Model | Effort | Purpose |
|-------|-------|--------|---------|
| `frontend-developer` | sonnet | medium | Index/router. Entry point for all web UI work — routes by framework marker to the specialist agents below; handles cross-framework and plain HTML/CSS/TS tasks. |
| `react-developer` | sonnet | high | React 19 + Next.js App Router — RSC, Server Actions, the `use` hook, disciplined hooks, hydration correctness. |
| `vue-developer` | sonnet | high | Vue 3 + Nuxt — Composition API and `<script setup>` only, Pinia state, SSR-safe components. |
| `svelte-developer` | sonnet | high | Svelte 5 + SvelteKit — runes reactivity, snippets, load/form actions. |
| `angular-developer` | sonnet | high | Angular 18+ — signals, standalone components, new control flow, disciplined RxJS interop. |
| `typescript-developer` | sonnet | high | TypeScript 5.x type layer — generics, narrowing, discriminated unions, strict `tsconfig`, `no-any` discipline. |
| `css-developer` | sonnet | high | Modern responsive accessible styling — container queries, `:has()`, cascade layers, subgrid, Tailwind, design tokens. |
| `frontend-architector` | opus | xhigh | Rendering strategy (CSR/SSR/SSG/ISR), micro-frontend/module-federation boundaries, client-state and design-system architecture. |
| `fe-test-generator` | sonnet | high | Component/integration/e2e test generation in the repo's existing framework (Vitest/Jest + Testing Library, Playwright/Cypress). |
| `fe-performance-engineer` | sonnet | high | **Review-only.** Core Web Vitals (LCP/INP/CLS), Lighthouse, bundle analysis, render/hydration tracing. Fixes route to `fe-code-fixer`. |
| `fe-accessibility-auditor` | sonnet | high | **Review-only.** WCAG 2.2 — ARIA, keyboard/focus order, contrast, target size, axe-core. Fixes route to `fe-code-fixer`. |
| `fe-security-auditor` | sonnet | high | **Review-only.** XSS sinks (CWE-79), CSP gaps, `dangerouslySetInnerHTML`/`v-html`, secrets-in-bundle, npm supply chain, SSRF via SSR fetch (CWE-918). Fixes route to `fe-code-fixer`. |
| `fe-code-fixer` | haiku | medium | Minimal-diff remediation of findings from review, security, accessibility, and performance audits. |
| `fe-dependency-manager` | haiku | low | npm/pnpm/yarn manifest + lockfile lifecycle; CVE/license audit; safe one-at-a-time upgrades with a build+test gate. |

Review-only agents carry `disallowed-tools: Write, Edit`; they emit a compressed P0–P3 findings summary and never patch `state.json` or write the stage report.

## Commands

16 slash commands, sharing the cross-plugin naming standard with `apple-developer`. Each declares `description`, `argument-hint`, and a whitelisted `allowed-tools`, and degrades gracefully on a missing optional tool. `analyze-*` commands are read-only.

### Analyze

| Command | Description |
|---------|-------------|
| `/analyze-accessibility` | Audit web UI for WCAG 2.2 conformance with axe-core and Lighthouse, triaged by severity. `--fix` routes remediation to the accessibility auditor and code fixer. |
| `/analyze-tech-debt` | Identify, quantify, and prioritize frontend tech debt across code, types, CSS, dependencies, tests, and framework majors. |

### Architecture

| Command | Description |
|---------|-------------|
| `/arch-select` | Select a frontend architecture — rendering strategy, state management, component boundaries, framework, and repo shape. |
| `/arch-review` | Review a frontend codebase's architecture — layering, state ownership, data-fetching seams, coupling, bundle split. |

### Build & diagnose

| Command | Description |
|---------|-------------|
| `/build-test` | Detect the package manager and framework, install dependencies, build, and run unit/component tests. The build gate every other command calls. |
| `/debug` | Configure browser and framework debugging workflows, or triage and root-cause a specific web error. |
| `/deps` | Audit npm dependencies for vulnerabilities and licenses, upgrade safely, or add a new package. First token selects the mode; `audit` is the default. |

### Fix

| Command | Description |
|---------|-------------|
| `/fix-quick` | Run linters and formatters (ESLint or Biome, Prettier, Stylelint) — check-only or auto-fix — then re-check. |
| `/fix-refactor` | Refactor web UI for clean code and SOLID — `frontend-architector` plans, `fe-code-fixer` applies; `--extract` pulls code into a shared package. |
| `/fix-modernize` | Migrate a web codebase to modern framework idioms (React hooks, Vue 3, Angular standalone, Svelte 5), unit by unit. |
| `/fix-performance` | Profile web performance with Lighthouse, Core Web Vitals, and bundle analysis, then optionally apply the fixes. Measure-only by default; `--apply` mutates only after a PHASE CHECKPOINT approval. |

### Generate

| Command | Description |
|---------|-------------|
| `/gen-component` | Scaffold a component (props, state, test, and story) in the project's detected framework and conventions. |
| `/gen-tests` | Generate, register, and verify a runnable unit, component, or e2e test suite using the project's framework. |
| `/gen-docs` | Generate or update TSDoc comments, typedoc API reference, Storybook docs, and README API sections. |

### Review & deliver

| Command | Description |
|---------|-------------|
| `/review-code` | Review code across React, Vue, Svelte, Angular, TypeScript, and CSS, synthesized into a P0-P3 report. |
| `/develop-feature` | Develop a web feature end-to-end — architecture, framework implementation, tests, and a security pass, build-gated. |

### Migration: old → new names

Every command below was renamed in 1.2.0 to match the `apple-developer` naming standard. The old names are gone, not aliased — update any script, skill, or worktask payload that still calls them.

| Old name (≤ 1.1.0) | New name (1.2.0+) |
|--------------------|-------------------|
| `/code-review` | `/review-code` |
| `/lint-fix` | `/fix-quick` |
| `/code-modernize` | `/fix-modernize` |
| `/profile-performance` | `/fix-performance` |
| `/generate-tests` | `/gen-tests` |
| `/deps-audit` | `/deps` |
| `/a11y-audit` | `/analyze-accessibility` |
| `/component-scaffold` | `/gen-component` |

`/build-test` keeps its name. Two renames change behavior as well as spelling: `/fix-performance` adds an opt-in apply phase (the old `/profile-performance` was measure-only, which is still the default), and `/deps` dispatches on a subcommand token where `/deps-audit` implied the audit.

## Skills

8 domains under `skills/`, plus a shared layer (`skills/_shared/`). The router `skills/SKILL.md` carries the "I need help with…" task→leaf table; each `<domain>/<domain>-skills/SKILL.md` is the canonical selection table for that domain, and leaves carry topic depth and link back to it. Every version-specific claim links the canonical `skills/_shared/version-feature-matrix.md`.

| Domain | Canonical selector | Leaves |
|--------|--------------------|--------|
| TypeScript | `typescript/typescript-skills` | `modern-typescript`, `ts-typing`, `ts-config` |
| React | `react/react-skills` | `modern-react`, `react-state`, `react-performance` |
| Vue | `vue/vue-skills` | `vue-composition`, `vue-state` |
| Svelte | `svelte/svelte-skills` | `svelte-runes`, `sveltekit` |
| Angular | `angular/angular-skills` | `angular-signals`, `angular-rxjs` |
| Styling | `styling/styling-skills` | `modern-css`, `tailwind-design-system`, `responsive-accessible-css` |
| Tooling | `tooling/tooling-skills` | `build-systems`, `fe-diagnostics`, `bundling-optimization` |
| Quality | `quality/quality-skills` | `accessibility-patterns`, `web-performance`, `fe-testing` |

Shared (`skills/_shared/`): `workflow-integration` (DV screenshot gate, stage templates), `secure-coding` (XSS/injection + CSP/secrets references), `model-selection`, `version-feature-matrix`, `language-detection`, `severity-matrix`, `testing-principles`, `accessibility-baseline`.

## Evidence model

Front-end work is UI work, so DV defaults to **`requires_screenshots: true`** (matching apple-developer, unlike system-developer's CLI default of `false`). The DV stage:

1. Captures one screenshot per meaningful screen/state via the company-workflow **`web_adapter`** (Playwright `npx playwright screenshot` / Chrome MCP rendered DOM) and writes a manifest at `.context/images/<worktask_id>/screenshots.md` with columns `| name | path | source | design_ref | notes |`. The `source` value is **`web-adapter`** (`cli-fallback` only when a route cannot render headlessly).
2. Attaches **Lighthouse** and **axe** reports as *supporting* evidence (additional `source: web-adapter` rows with `notes: lighthouse`/`notes: axe`, or referenced from the DV Build Evidence block) — never a substitute for screen captures.
3. Records Build Evidence: toolchain + versions, `tsc --noEmit` → 0 errors, eslint/biome clean, bundle-size delta, and a test transcript path under `.context/logs/`.

If the manifest is absent at `SubagentStop`, the company-workflow DV screenshot gate blocks and re-dispatches.

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
