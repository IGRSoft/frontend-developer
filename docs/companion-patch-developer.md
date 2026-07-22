# Companion Patch — `developer.md` (web platform specialization)

**Status (2026-07-22): applied upstream** — company-workflow's `agents/developer.md` now carries the frontend-developer Task grants and routing (see igrsoft v3.36.0). This document is retained as the historical patch spec.

**Status:** self-contained patch artifact. This document is **not** an edit to any installed
plugin cache. It describes the exact additions to apply to the **editable company-workflow
source** of `agents/developer.md` so the igrsoft `developer` router recognizes web work and
delegates it to the `frontend-developer` plugin.

> **Do not edit the read-only cache** (`/Users/korich/.claude/plugins/cache/igrsoft/...`). Apply
> these changes only in the canonical editable company-workflow repository (see §7 *Apply
> instructions*).

The patch has 7 sections:
1. Web/native precedence rule (verbatim)
2. `tools:` frontmatter entries to add
3. "Web Platform Specialization" marker→agent table
4. Detection-rule row
5. Direct Platform Specialist Routing + Routing Audit web rows
6. Screenshot Capture note (no change)
7. Apply instructions

---

## 1. Web/native precedence rule (paste verbatim)

Add this paragraph to `developer.md` immediately after the existing platform-routing preamble
(and mirror it in the `frontend-developer` plugin's `_base` Delegation Routing note, where it
already lives):

> **Web vs native precedence.** When a task carries both web markers
> (`.ts`/`.tsx`/`.jsx`/`package.json`/`tsconfig.json`/framework configs) and
> native markers (`.swift`/`.xcodeproj`/`Package.swift`/native module
> directories), route the **app/UI layer to `frontend-developer:frontend-developer`**
> and the **native-module layer to `apple-developer:*`**. The deciding question
> is *which layer the change targets*: UI/component/state/styling/build-tooling
> work is web (front-end wins); a native module, bridging header, or
> platform-API binding is native (Apple wins). React Native / Expo splits the
> same way — JS/TS surface to the (optional) `react-native-developer`, native
> modules deferred to `apple-developer:*`. Default to `frontend-developer` for
> ambiguous pure-JS/TS web work.

---

## 2. `tools:` frontmatter entries to add

Add the following `Task(...)` scopes to `developer.md`'s frontmatter `tools:` list (qualified
form; one per delegation target). These let the `developer` router hand web work to the
`frontend-developer` specialists:

```
Task(frontend-developer:frontend-developer),
Task(frontend-developer:react-developer),
Task(frontend-developer:vue-developer),
Task(frontend-developer:svelte-developer),
Task(frontend-developer:angular-developer),
Task(frontend-developer:typescript-developer),
Task(frontend-developer:css-developer),
Task(frontend-developer:fe-code-fixer),
Task(frontend-developer:fe-test-generator)
```

Notes:
- All references are **fully qualified** `frontend-developer:<agent>`. Bare names are deprecated.
- `backend-developer:*` and `apple-developer:*` remain **documented forward-reference handoffs
  only** — they are routing targets in prose, never added to a `Task(...)` tool scope here (an
  unresolvable Task scope would break the router if the plugin is not installed).
- The review-only specialists (`fe-performance-engineer`, `fe-accessibility-auditor`,
  `fe-security-auditor`) are reached through the stage flow (DR/SR/QA), not added as direct
  `developer.md` Task scopes.

---

## 3. "Web Platform Specialization" table (new — parallel to the Apple/Systems tables)

Add this table to `developer.md` alongside the existing platform-specialization sections:

### Web Platform Specialization

| Marker / signal | Route to |
|-----------------|----------|
| `.tsx`, `.jsx`, React imports, `next.config.*`, App Router | `frontend-developer:react-developer` |
| `.vue`, `<script setup>`, `nuxt.config.*`, Pinia | `frontend-developer:vue-developer` |
| `.svelte`, runes (`$state`/`$derived`), `svelte.config.*`, SvelteKit | `frontend-developer:svelte-developer` |
| `angular.json`, `@Component`, signals, standalone components | `frontend-developer:angular-developer` |
| `.ts`, `tsconfig.json`, type-layer work, generics, `tsc` errors | `frontend-developer:typescript-developer` |
| `.css`/`.scss`, Tailwind config, design tokens, layout/responsive/a11y styling | `frontend-developer:css-developer` |
| Rendering strategy, micro-frontend boundaries, state/design-system architecture | `frontend-developer:frontend-architector` |
| Component/unit/e2e test generation (Vitest/Jest/Playwright/Cypress) | `frontend-developer:fe-test-generator` |
| Cross-framework, plain HTML/CSS/TS, or ambiguous web work | `frontend-developer:frontend-developer` (index/router) |

---

## 4. Detection-rule row

Add this row to `developer.md`'s Detection Rules table:

| Detected markers | Platform | Route to |
|------------------|----------|----------|
| `.ts` / `.tsx` / `.jsx` / `.js` / `package.json` / `tsconfig.json` | **web** | `frontend-developer:frontend-developer` |

**Precedence note (attach to the row):** when native markers
(`.swift`/`.xcodeproj`/`Package.swift`/native module dirs) co-occur with these web markers, apply
§1 — the **UI/app layer routes to `frontend-developer:frontend-developer`** and the
**native-module layer routes to `apple-developer:*`**. Default to `frontend-developer` for
ambiguous pure-JS/TS web work.

---

## 5. Direct Platform Specialist Routing + Routing Audit (web rows)

### Direct Platform Specialist Routing — add web rows

| Request shape | Direct specialist |
|---------------|-------------------|
| "build/fix a React/Next component or hook" | `frontend-developer:react-developer` |
| "Vue/Nuxt composable or reactivity bug" | `frontend-developer:vue-developer` |
| "Svelte 5 runes / SvelteKit load or action" | `frontend-developer:svelte-developer` |
| "Angular signals / standalone / RxJS-to-signals" | `frontend-developer:angular-developer` |
| "TypeScript types / `tsc` errors / strictness migration" | `frontend-developer:typescript-developer` |
| "CSS architecture / Tailwind / responsive + a11y styling" | `frontend-developer:css-developer` |
| "front-end architecture / rendering strategy / migration plan" | `frontend-developer:frontend-architector` |

### Routing Audit — add web rows

| Audit check | Expected route | Failure signal |
|-------------|----------------|----------------|
| Web UI task routed to a non-web specialist | `frontend-developer:*` | UI/component/state/styling work handled by Apple/Systems agent |
| Native-module task in a web repo routed to a web agent | `apple-developer:*` | `.swift`/bridging/native-API work handled by a front-end agent |
| Ambiguous pure-JS/TS work not routed to the index agent | `frontend-developer:frontend-developer` | cross-framework task split across specialists without a router pass |

---

## 6. Screenshot Capture table — no change needed

The igrsoft `web_adapter` row is **already present** in `developer.md`'s Screenshot Capture table
(Playwright `npx playwright screenshot` / Chrome MCP). The `frontend-developer` evidence model
(§D3: `requires_screenshots: true` default, `source: web-adapter`, manifest at
`.context/images/<worktask_id>/screenshots.md`, Lighthouse/axe as supporting rows) **reuses that
existing `web_adapter` capture path** — so no edit to the Screenshot Capture table is required.

---

## 7. Apply instructions

1. **Target file (editable source, not the cache).** Locate the canonical editable
   company-workflow repository — the source that *builds* the read-only
   `~/.claude/plugins/cache/igrsoft/...` cache (commonly a `company-workflow/` checkout, path
   `agents/developer.md`). **Do not edit the cache directory.**
2. **Git context.** Work on a feature branch in the company-workflow repo (e.g.
   `feat/web-platform-routing`). Apply sections §1–§5 as additive edits to `agents/developer.md`;
   §6 is verify-only.
3. **Validate.** Re-run the company-workflow agent linter / `validate.sh` to confirm: every added
   `Task(...)` is fully qualified; no `backend-developer:*`/`apple-developer:*` entry leaked into a
   `tools:` scope; the new tables parse.
4. **Version bump.** Bump the company-workflow plugin version (this is an additive routing feature
   — minor bump, e.g. `3.17.0 → 3.18.0`) and add a changelog entry: "developer router: web platform
   specialization → `frontend-developer:*`."
5. **Rebuild the cache** from source per the company-workflow release process; do not hand-edit the
   installed cache copy.

---

*This artifact is emitted by the `frontend-developer` plugin (DV/B9). It documents the companion
change required in the igrsoft company-workflow `developer.md` router; the `frontend-developer`
plugin itself depends on none of it (the patch is additive on the igrsoft side).*
