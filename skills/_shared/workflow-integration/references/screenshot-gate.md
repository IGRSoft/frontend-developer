# DV Screenshot Gate — Capture Recipe and Manifest Format

Use this when you are a frontend-developer DV agent producing the screenshot manifest that `dv-screenshot-gate.sh` enforces. The binding rules (`requires_screenshots` default, `source: web-adapter`, Lighthouse/axe as supporting evidence, re-dispatch behavior) live in `SKILL.md § DV Screenshot Gate`; this file holds the step-by-step capture procedure, the supporting-evidence detail, the RMSE join-key semantics, and the manifest row format.

## web_adapter Capture Procedure

Web DV produces the manifest through igrsoft's **`web_adapter`** capture path (Playwright / Chrome MCP rendered DOM) — already wired in `company-workflow/agents/developer.md`'s Screenshot Capture table:

1. Build the app and serve it (or run the dev server) — `npm run build` then a static preview, or `npm run dev`. One command per scoped-Bash call; no `&&` chains.
2. Capture each meaningful route/state with the `web_adapter` — Playwright `npx playwright screenshot <url> <out.png>` (headless Chromium) or the Chrome MCP screenshot tool. These rows take **`source: web-adapter`**.
3. If a route cannot be rendered headlessly (auth-walled flow, native bridge, hardware-gated state), fall back to a CLI/textual capture (`source: cli-fallback`) and put the reason in `notes`.
4. Write `.context/images/<worktask_id>/screenshots.md` — one row per route/state (name, path, source, optional `design_ref`, notes). Format and example: [templates/dv-screenshots.md](../templates/dv-screenshots.md). Frontmatter `screenshot_count` MUST equal the row count.

## Live-drive capture (`ui_visual_check: true`)

When `metadata.ui_visual_check: true`, a capture is valid only if it was **live-driven this run** — an un-interacted route screenshot or a Storybook/component render does not satisfy it (see `_base/frontend-agent.md § Live-drive verification`). For each such row:

1. Serve the real route from a production-like build (`npm run build` + `npm run preview`) or the dev server — no state injection, no pre-baked deep-link.
2. Drive the app to each target substate the acceptance criteria name through real interactions only — clicks, typing, submissions via Playwright or Chrome MCP.
3. Confirm each primary control is visible and enabled, THEN capture; never capture a substate not reached through real interactions.
4. In `notes`, record the interaction path that reached it.

**Freshness.** Every capture is taken this run; QA direct-reads each image and flags byte-identical, blank/error, wrong-route, or stale reused captures — re-opening DV.

## Supporting Evidence (not a substitute)

Attach **Lighthouse** and **axe** reports as supporting rows in the same manifest (`source: web-adapter`, `notes: lighthouse` / `notes: axe`, `path` → the JSON/HTML report under `.context/images/<worktask_id>/`) — or reference them from the DV Build Evidence block. They corroborate the captures; they never replace a rendered screenshot of the changed UI.

## RMSE Design-Diff Join Key

The manifest is also the join key for QA's RMSE design-diff (igrsoft 3.11.4): `design_ref` links a screenshot to its `designs/` mockup so QA can compare pixel deltas — leave it blank for net-new screens with no mockup. Opt out only when `metadata.requires_screenshots: false` (non-UI changes, e.g. a pure tooling/config edit) — then no manifest is required and the gate is skipped; flag it in your return summary if the metadata says otherwise. **Never** fabricate image files or return without either the `false` flag or a real manifest — the gate re-dispatches DV until one exists.

## Manifest Row Format

Manifest row format mirrors igrsoft's `dv-screenshot-capture` output: `| name | path | source | design_ref | notes |` with `source` ∈ {`web-adapter`, `cli-fallback`}.
