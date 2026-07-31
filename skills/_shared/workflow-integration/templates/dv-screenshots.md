# DV Screenshot Manifest Template (company-workflow v3.12.0)

Format for the screenshot manifest a DV agent MUST write at
`.context/images/<worktask_id>/screenshots.md` when `metadata.requires_screenshots != false`
(default **TRUE** for web UI changes). If this file is absent at `SubagentStop`, company-workflow's
`dv-screenshot-gate.sh` blocks the stop and returns `hookSpecificOutput.additionalContext`
telling the run to capture screenshots (`dv-screenshot-capture`) — DV is re-dispatched until
the manifest exists.

This mirrors company-workflow's `dv-screenshot-capture` skill output so the gate finds it, and the
`design_ref` column is the join key for QA's RMSE design-diff (company-workflow 3.11.4).

## How web DV produces it

1. Build the app and serve it: `npm run build` then a static preview (`npx vite preview`,
   `npx serve dist`, framework equivalent), or run the dev server `npm run dev`. One command
   per scoped-Bash call — no `&&` chains.
2. Capture each meaningful **route/state** via company-workflow's `web_adapter`:
   - Playwright headless: `npx playwright screenshot <url> .context/images/<worktask_id>/<name>.png`
     (per viewport when responsive states matter) → `source: web-adapter`.
   - Or the Chrome MCP screenshot tool against the same URL → `source: web-adapter`.
3. If a route cannot render headlessly (auth-walled flow, native bridge, hardware-gated state),
   fall back to a CLI/textual capture → `source: cli-fallback`, and put the reason in `notes`.
4. (Supporting, optional) Save Lighthouse / axe reports under
   `.context/images/<worktask_id>/` and add a row with `source: web-adapter`,
   `notes: lighthouse` / `notes: axe`, `path` → the JSON/HTML report. These corroborate the
   captures; they do not replace a rendered screenshot of the changed UI.
5. Write the manifest below (one row per route/state). The `path` is relative to the repo /
   worktree root.

## Manifest file format

Copy this to `.context/images/<worktask_id>/screenshots.md`:

```markdown
---
handoff:
  stage: DV
  artifact: screenshots-manifest
  worktask_id: "<worktask_id>"
  screenshot_count: 3
---

# DV Screenshots — <worktask_id>

| name | path | source | design_ref | notes |
|------|------|--------|------------|-------|
| Login (empty) | .context/images/<worktask_id>/login-empty.png | web-adapter | designs/login.png | |
| Login (error) | .context/images/<worktask_id>/login-error.png | web-adapter | | new error state, no mockup |
| Login Lighthouse | .context/images/<worktask_id>/login-lighthouse.json | web-adapter | | lighthouse, supporting evidence |
```

## Column contract

| Column | Required | Values / format |
|--------|----------|-----------------|
| `name` | yes | Human-readable route/state name (unique within the manifest). |
| `path` | yes | Path to the captured image, under `.context/images/<worktask_id>/`. For `cli-fallback`, a `.txt`/`.md` capture is acceptable; supporting Lighthouse/axe rows point at the `.json`/`.html` report. |
| `source` | yes | `web-adapter` (Playwright `npx playwright screenshot` or Chrome MCP rendered DOM) or `cli-fallback` (textual/CLI capture when a headless render is impossible). |
| `design_ref` | optional | Path to the matching `designs/` mockup; populated when a designer mockup exists. QA uses it for the RMSE pixel-diff join — leave blank for net-new screens with no mockup. |
| `notes` | optional | Why a fallback was used, the supporting-evidence kind (`lighthouse`/`axe`), or any caveat (e.g. "auth-walled flow"). |

## Notes

- One row per *meaningful* route/state — not every transient frame. Capture the states a
  reviewer or QA needs to verify the change, plus any new responsive breakpoints that changed.
- The frontmatter `screenshot_count` MUST equal the number of table rows (the gate / QA may
  assert this) — supporting Lighthouse/axe rows count too.
- Opt out only with `metadata.requires_screenshots: false` (non-UI changes, e.g. a pure
  tooling/config edit) — then this file is not required and the gate is skipped.
- For a `ui_visual_check` row, state live-driven provenance in the `notes` column — which
  interaction path reached the state (e.g. "typed invalid email → submitted → error") — or,
  for a `cli-fallback`, the reason it could not be live-driven. Columns and frontmatter keys
  are unchanged; provenance rides in `notes`.
- See `workflow-integration/SKILL.md § DV Screenshot Gate` and the framework DV agents'
  "DV completion — screenshot manifest" notes.
