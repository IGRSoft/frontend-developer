# Decisions Log — frontend-developer plugin

Cross-stage decision record for the `frontend-developer` Claude Code plugin (v1.0.0).
Captures PL/AR/TL/DV decisions, the AR architecture decisions D1–D6, the evidence-model
choice, open-question resolutions, and the conventions every future change must preserve.

## Origin & scope

- **Apply-mode origin.** This plugin was authored in *apply mode*: it is a copy-and-reskin of
  two read-only reference plugins, not a green-field design. `system-developer@1.0.0` governs
  structure (`_base`, agent frontmatter, hooks, `validate.sh`, manifest shape, handoff contract,
  skill router); `apple-developer@1.11.0` governs the UI evidence model (screenshots). The two
  reference plugins are **never edited** — they are copied and reskinned for the web domain.
- **Target domain.** Web front-end: React/Next.js, Vue/Nuxt, Svelte/SvelteKit, Angular,
  TypeScript, modern CSS/Tailwind. Deliverables are plugin authoring artifacts (md/json/sh),
  not running web code.

## Complexity & stage list

- **Complexity = 30/50 (Moderate).** Set by PL, validated by AR with no change.
- **Stage list = PL→AR→TL→DV→DR→QA→DC→FN→ST.** AR confirmed this is correct.
- **SR (security review) skipped — rationale:** the artifacts are docs/JSON/shell authoring,
  not executable web code with a runtime attack surface. Web *secure-coding guidance* still ships
  as a skill (`skills/_shared/secure-coding/` + `fe-security-auditor`), but no SR *stage* runs over
  the plugin's own authored files. (The plugin still *participates* in the igrsoft 11-stage SR when
  invoked on a real web codebase — see `_base` §8 SR context note.)
- **DR gates code quality before QA** — `technical-lead` reviews the authored tree; `fe-code-fixer`
  applies minimal-diff remediation; `frontend-architector` consults on structural questions.

## AR architecture decisions (D1–D6)

- **D1 — `_base/frontend-agent.md` inheritance surface.** Re-skin of system-developer's
  `language-agent.md`. Plain markdown, **no YAML frontmatter** (mirrors apple-developer's
  `platform-agent.md`); `validate.sh` skips `agents/_base/*` from frontmatter checks and the
  manifest. Exactly 8 sections: Constraints; Mandatory Requirements (skills table); Code Comment
  Policy (TSDoc); Tool Priority; Delegation Routing; Standard Response Format; Workflow Stage
  Participation → Handoff Contract; Workflow Stage Participation → per-stage recipes. Every leaf
  agent opens with a one-line "Inherits `_base/frontend-agent.md` …; do not restate the base" and
  then only delta sections.
- **D2 — Agent frontmatter schema.** Exact key order `name, description, model, effort, maxTurns,
  color, [disallowed-tools,] tools, inherits`. No invented keys; no top-level `requires_screenshots`
  (that is runtime `task.metadata.*`). Three canonical blocks: (a) normal agent, (b) review-only
  agent (adds `disallowed-tools: Write, Edit`), (c) architector (`model: opus`, `effort: xhigh`,
  `maxTurns: 60`). `xhigh` is honored only on opus, so the architector must be opus.
- **D3 — Evidence-model reconciliation (the one divergence from system-developer).** Keep
  system-developer's hooks / `validate.sh` / `_base` shape / handoff contract unchanged; swap the
  evidence model to apple-developer's UI/screenshot path routed through igrsoft's `web_adapter`.
  Concretely: `requires_screenshots` default flips to **true**; DV writes
  `.context/images/<worktask_id>/screenshots.md` with columns `| name | path | source | design_ref |
  notes |`; `source` value is **`web-adapter`** (where apple uses `apple-canvas`, system uses
  `cli-fallback`); Lighthouse + axe are *supporting* rows, not substitutes; `screenshot_count`
  frontmatter equals the row count; an absent manifest at `SubagentStop` re-dispatches DV.
  **Do not copy system-developer's `cli-fallback`-default evidence prose.**
- **D4 — Skill router topology.** `skills/SKILL.md` is the router ("I need help with…" table +
  8-domain quick-nav, with `name`+`description` frontmatter); `skills/_index.md` is the nav index;
  each `skills/<domain>/<domain>-skills/SKILL.md` is *the* canonical selection table for that domain;
  leaves link back and never duplicate it. `skills/_shared/version-feature-matrix.md` is the
  canonical version source — every version-specific claim carries the marker
  `> Requires <feature> (<fw> <version>+). Fallback: <pre-version>. Canonical:
  _shared/version-feature-matrix.md`. SKILL.md >8KB without a `references/` sibling earns a WARN.
- **D5 — Qualified-name + handoff invariants.** Every `Task(...)` reference uses the fully-qualified
  `frontend-developer:<agent>` form. **Backend and Apple handoffs are forward-references only:**
  routing to `backend-developer:*` / `apple-developer:*` is *documented* in the `_base` Delegation
  Routing table but **never appears in any `tools:` `Task(...)` list** (an unresolvable Task scope
  would break the agent). Same treatment for the optional `react-native-developer`. Every stage
  artifact carries unconditional `handoff:` frontmatter; atomic state.json write
  (read→merge→temp→fsync→rename); review-only agents never patch state.json or write the stage report.
  Hooks are advisory and never merge state.json.
- **D6 — Web/native precedence rule.** When a task carries both web markers
  (`.ts`/`.tsx`/`.jsx`/`package.json`/`tsconfig.json`/framework configs) and native markers
  (`.swift`/`.xcodeproj`/`Package.swift`/native module dirs), route the **app/UI layer to
  `frontend-developer:frontend-developer`** and the **native-module layer to `apple-developer:*`**.
  The deciding question is which layer the change targets. Default to `frontend-developer` for
  ambiguous pure-JS/TS web work. The verbatim paragraph lives in the `_base` Delegation Routing note
  and in `docs/companion-patch-developer.md`.

## Evidence-model choice (summary)

Front-end work is UI work → DV defaults `requires_screenshots: true` (apple-developer path), captured
via the igrsoft `web_adapter` (Playwright / Chrome MCP), `source: web-adapter`, with Lighthouse and
axe attached as supporting evidence. This is the single intentional divergence from system-developer
(which defaults `false`, CLI). Risk R3 is closed by sourcing screenshots from apple-developer and
hooks/validator/`_base`/handoff from system-developer.

## Open-question resolutions

- **q1 — companion-patch artifact path.** The canonical editable company-workflow `developer.md`
  source is a read-only cache; DV does **not** edit it. Resolution: DV emits a self-contained patch
  artifact at `docs/companion-patch-developer.md` (TL-confirmed path) describing the exact additions
  and apply instructions. The cache is never mutated.
- **q2 — DV write-root.** Confirmed: WORKSPACE_ROOT
  (`/Users/korich/conductor/workspaces/frontend-developer/dhaka-v1`) is the editable plugin repo
  root; all authored paths are relative to it.

## Conventions to preserve (binding for all future changes)

1. **Qualified Task names** — every `Task(...)` is `frontend-developer:<agent>`; bare names are
   deprecated. `backend-developer:*`/`apple-developer:*` appear only as documented forward-reference
   handoffs, never in a `tools:` `Task(...)` list.
2. **Version markers** — every version-specific claim links `_shared/version-feature-matrix.md` with
   the canonical "Requires … Fallback … Canonical" pattern.
3. **Review-only markers** — `fe-performance-engineer`, `fe-accessibility-auditor`,
   `fe-security-auditor` carry `disallowed-tools: Write, Edit`, never list Write/Edit in `tools:`,
   and route fixes to `frontend-developer:fe-code-fixer`.
4. **Single-command Bash scopes** — `Bash(npm:*)`, `Bash(npx:*)`, etc. No bare `Bash`, no `&&`
   chains, no `;`/`|` joins (a scoped grant cannot match a compound line).
5. **Agent frontmatter key order** — `name, description, model, effort, maxTurns, color,
   [disallowed-tools,] tools, inherits`. No invented keys.
6. **`Use PROACTIVELY` / `Use when`** in every agent and skill description (trigger lint).
7. **`_base/frontend-agent.md` has no YAML frontmatter** and is excluded from the manifest.
8. **Evidence default `requires_screenshots: true`**, `source: web-adapter`, manifest at
   `.context/images/<worktask_id>/screenshots.md`.
9. **`scripts/validate.sh`** is the release gate: `ALLOWED_PREFIX_RE` includes `frontend-developer`
   (plus `backend-developer`/`apple-developer` forward-refs); the own-plugin existence branch keys on
   `frontend-developer`. Run before any release; `--strict` for CI.
