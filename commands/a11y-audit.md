---
description: Audit web UI for WCAG 2.2 conformance with axe-core and Lighthouse, triage findings by severity, and optionally route fixes to the accessibility auditor and code fixer
argument-hint: [path or URL (default: detect dev server)] [--fix] [--level A|AA|AAA]
allowed-tools: Read, Glob, Grep, Bash, WebSearch, WebFetch
estimated-cost:
  band: high
  min-tokens: 4000
  max-tokens: 30000
  model-distribution:
    haiku: 15%
    sonnet: 65%
    opus: 20%
---

# Accessibility Audit (WCAG 2.2)
<!-- Updated: June 2026 -->

Audit a web UI against WCAG 2.2 using automated tooling (axe-core, the Lighthouse accessibility category), triage every finding by severity against the success criteria, and produce a prioritized P0-P3 report. With `--fix`, route the findings to `frontend-developer:fe-accessibility-auditor` for a review-grade triage, then to `frontend-developer:fe-code-fixer` for minimal, gated patches.

[Extended thinking: Automated a11y tooling catches roughly a third of WCAG issues — the machine-detectable ones (missing names, contrast, ARIA validity) — so this command treats axe-core/Lighthouse as the *evidence collector*, not the auditor. It runs them against a rendered page (a dev server or static build, captured via the web_adapter rendering path), then triages each rule violation to its WCAG 2.2 success criterion and severity. The `--fix` path is deliberately two-staged: the accessibility auditor (review-only) confirms and contextualizes the machine findings and adds the keyboard/focus/reading-order issues the scanners miss, then the code fixer applies only the localized, mechanical fixes under a minimal-diff gate. Keyboard operability, focus order, and meaningful-sequence findings are human-judgment items the auditor owns — the command never claims a page is "accessible" on a green axe run alone.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Automated tooling is evidence, not a verdict.** A clean axe/Lighthouse run does NOT mean WCAG-conformant. Report the automated coverage honestly and flag the categories scanners cannot check (keyboard operability, focus order, meaningful sequence, error identification quality) as requiring the auditor / manual review.
2. **Render before you scan.** axe-core and Lighthouse need a rendered DOM. Resolve a target URL (running dev server, preview server, or a served static build) via the web_adapter rendering path before scanning. If nothing renders, fall back to static source analysis with a clear reduced-coverage note.
3. **Triage every finding to a WCAG 2.2 success criterion.** Each violation maps to an SC (e.g. 1.1.1, 1.4.3, 2.4.7, 4.1.2) and a severity from `skill: severity-matrix`. `--level` bounds which SCs are in scope (A / AA / AAA).
4. **`--fix` is two-staged and gated.** First route to `frontend-developer:fe-accessibility-auditor` (review-only — confirms findings, adds manual-check items, groups by SC/severity); then route its P0/P1 localized fixes to `frontend-developer:fe-code-fixer` under a minimal-diff gate. Never let the auditor patch — it is review-only.
4a. **Re-verify after fixes.** After `fe-code-fixer` applies patches, re-run the axe scan over the touched routes to confirm the violation is gone and no new violation was introduced.
5. **Single-command Bash invocations.** Use each tool's own flags (`npx @axe-core/cli <url>`, `npx lighthouse <url> --only-categories=accessibility`). Never `cd`-chain or `&&`-chain — scoped Bash patterns do not match compound commands.
6. **Tool-missing never hard-fails.** If axe-core or Lighthouse is unavailable, print the install hint, run the remaining scanner, and continue with reduced coverage. If both are missing, fall back to static source analysis. Never abort over one missing tool.
7. **Commands route, they do not orchestrate.** This command names `frontend-developer:fe-accessibility-auditor` and `frontend-developer:fe-code-fixer` so Claude routes the `--fix` work; it does not call `Task`.
8. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Audit the detected dev server (or prompt for a URL)
/frontend-developer:a11y-audit

# Audit a specific route
/frontend-developer:a11y-audit http://localhost:5173/checkout

# Audit a component directory (static source analysis + render if possible)
/frontend-developer:a11y-audit src/components/Modal

# Audit AA and route fixes through the auditor + code fixer
/frontend-developer:a11y-audit http://localhost:3000 --level AA --fix
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `path or URL` | detect dev server | A running URL to scan, or a source path (the command serves/renders it where it can). If omitted, detect a running dev server (`vite`/`next`/`ng serve` default ports) or prompt for a URL. |
| `--fix` | off | After triage, route to `frontend-developer:fe-accessibility-auditor` (review) then `frontend-developer:fe-code-fixer` (gated patches) for P0/P1 localized fixes, then re-verify. |
| `--level A\|AA\|AAA` | AA | The WCAG 2.2 conformance level to audit against. AA is the default legal/industry baseline; AAA findings are reported as advisory when requested. |

## Workflow

### Phase 1: Resolve & Render (Bash + web_adapter)

1. Resolve the target. If a URL is given, use it. If a source path is given, look for a running dev server; if none, attempt to start the project's preview/build server (`npm run preview` / serve `dist/`) and use that URL.
2. Confirm the URL renders (web_adapter rendering path: Playwright / Chrome). If it does not render, note "reduced coverage — static analysis only" and proceed to Phase 2b.
3. Enumerate the routes/states to scan: the target route plus any obvious sub-states (modal open, form invalid) the user names. Record them.

### Phase 2a: Automated Scan (Bash) — rendered target

1. Run axe-core against each route:
   ```bash
   npx @axe-core/cli "<url>" --tags wcag2a,wcag2aa,wcag22aa --save "$OUT/axe-<route>.json"
   ```
   (Scope `--tags` to `--level`.) Capture the JSON report under `.context/images/<worktask_id>/` (a11y reports are supporting evidence alongside screenshots).
2. Run the Lighthouse accessibility category:
   ```bash
   npx lighthouse "<url>" --only-categories=accessibility --output=json --output-path="$OUT/lh-<route>.json" --quiet --chrome-flags="--headless"
   ```
3. Reconcile axe and Lighthouse findings, de-duplicating by rule/element.

### Phase 2b: Static Source Analysis (Bash + Read) — fallback / supplement

When rendering is unavailable, or to supplement, analyze source for static a11y defects:
- Missing `alt` on `<img>`; unlabeled controls (`<button>`/`<a>`/icon-only with no accessible name); `<input>` without an associated `<label>`/`aria-label`.
- ARIA misuse (invalid role, `aria-*` on the wrong element, redundant roles); positive `tabindex`; `onClick` on non-interactive elements without keyboard handlers/roles; missing form `aria-describedby` for errors.
- Heading-order skips; missing `lang`; missing `:focus-visible` styles in CSS; `outline: none` without a replacement.
Note this is static-only and cannot assess runtime focus order or contrast against computed styles.

### Phase 3: Triage (synthesis)

1. Map each finding to its WCAG 2.2 success criterion and the responsible source `file:line` (resolve the rendered element back to source via component/selector mapping).
2. Assign severity from `skill: severity-matrix`: P0 = blocks a user from completing a task (no keyboard path, unlabeled critical control), P1 = serious barrier (contrast fail on body text, missing form labels), P2 = degraded experience, P3 = advisory/AAA.
3. Flag the scanner-blind categories explicitly: keyboard operability (2.1.1), focus order (2.4.3), meaningful sequence (1.3.2), error suggestion quality (3.3.3) — mark these "requires auditor / manual review."

### Phase 4: `--fix` (two-staged, gated)

If `--fix` is set:

1. Route to `frontend-developer:fe-accessibility-auditor` (review-only):
   "Review WCAG 2.2 ({level}) findings for `{target}`. Automated evidence (axe + Lighthouse) attached: {findings}. Confirm each finding against its SC, add the keyboard-operability / focus-order / reading-order issues the scanners missed, and return a P0-P3 list grouped by severity with `file:line` and a concrete patch suggestion per item. Review-only — do NOT edit."
2. Route the auditor's P0/P1 *localized* fixes to `frontend-developer:fe-code-fixer`:
   "Apply minimal, targeted accessibility fixes for these P0/P1 findings: {findings as `{file, line, sc, fix}`}. Minimal-diff gate: add the accessible name / label / role / focus style exactly as specified; do not refactor or restyle untouched markup. Report each change as `{file, line, sc, change}` and list anything you could NOT safely auto-fix (needs design judgment / structural change)."
3. **Re-verify**: re-run the Phase 2a axe scan over the touched routes. Confirm each fixed violation is gone and no new violation appeared.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect |
|--------------|--------------|--------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | no automated scan — static analysis only |
| `@axe-core/cli` | `npm install -D @axe-core/cli` | skip axe; rely on Lighthouse + static analysis; note reduced coverage |
| `lighthouse` | `npm install -D lighthouse` (or `npx lighthouse`) | skip Lighthouse; rely on axe + static analysis |
| Chrome/Chromium (Lighthouse/axe rendering) | install Chrome, or `npx playwright install chromium` | rendering unavailable → static source analysis with a reduced-coverage note |

If both scanners are missing, fall back to Phase 2b static source analysis and clearly mark the report "static-only — automated WCAG coverage unavailable." Never hard-fail on a missing optional tool: print the hint, run what remains, and report the reduced coverage.

## Output Format

```markdown
## Accessibility Audit Report (WCAG 2.2 — {level})

**Target:** {url | path}
**Rendered:** ✅ (web_adapter) / ❌ static-only
**Scanners:** {axe-core ✅ | Lighthouse ✅ | static fallback}
**Reports:** .context/images/{worktask_id}/axe-*.json, lh-*.json
**Lighthouse a11y score:** {0-100 | N/A}

### Summary
| Priority | Count | Example SC |
|----------|------:|------------|
| P0 (blocks task) | {n} | 2.1.1 Keyboard |
| P1 (serious barrier) | {n} | 1.4.3 Contrast, 4.1.2 Name/Role/Value |
| P2 (degraded) | {n} | 2.4.6 Headings & Labels |
| P3 (advisory / AAA) | {n} | 1.4.6 Contrast (Enhanced) |

### Findings
| File:Line | WCAG SC | Severity | Issue | Fix |
|-----------|---------|----------|-------|-----|
| Modal.tsx:24 | 2.4.7 Focus Visible | P1 | focus ring removed via `outline:none` | add `:focus-visible` ring |
| Icon.tsx:8 | 1.1.1 / 4.1.2 | P0 | icon-only button has no accessible name | add `aria-label` |

### Requires Manual Review (scanner-blind)
- {2.1.1 keyboard operability}: {note} — route to fe-accessibility-auditor
- {2.4.3 focus order}: {note}

<!-- --fix mode only -->
### Fixes Applied
| File:Line | SC | Change |
|-----------|----|--------|

**Re-verify (axe re-scan):** {N violations resolved, 0 new}
**Not auto-fixed (manual):** {structural/design items, or "none"}

<!-- on reduced coverage -->
### Coverage Notes
- {scanner/render missing} — {what was skipped, install hint above}
```

## Error Handling

### No render target
```
Note: No running dev server detected and no URL given.
Suggestion: Start your dev server (npm run dev) and pass its URL, e.g.
/frontend-developer:a11y-audit http://localhost:5173
Proceeding with static source analysis (reduced coverage).
```

### Rendering unavailable (no browser)
```
Warning: No Chromium available to render the page for axe/Lighthouse.
Install: npx playwright install chromium
Falling back to static source analysis — automated WCAG coverage is reduced.
```

### Both scanners missing
Fall back to static source analysis; mark the report "static-only." Print both install hints. Do not hard-fail.

## See Also

- `skill: accessibility-baseline` — WCAG 2.2 success-criteria reference the triage maps to.
- `skill: accessibility-patterns` — ARIA patterns, focus management, keyboard-nav recipes the fixes draw on.
- `skill: severity-matrix` — P0-P3 definitions used by the triage.
- `/frontend-developer:code-review` — the a11y pass there is a lighter, source-level check; this command is the full rendered audit.
- `/frontend-developer:generate-tests --type e2e` — add a Playwright + axe assertion to lock the fix in.
- `frontend-developer:fe-accessibility-auditor` — the review-only agent the `--fix` path routes to first.
