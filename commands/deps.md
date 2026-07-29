---
description: Audit npm dependencies for vulnerabilities and licenses, upgrade safely, or add a new package
argument-hint: [audit|upgrade|add <package>] [--manager npm|pnpm|yarn|bun] [--prod]
allowed-tools: Read, Edit, Glob, Grep, Bash, WebSearch, WebFetch
estimated-cost:
  min-tokens: 3000
  max-tokens: 18000
  model-distribution:
    haiku: 30%
    sonnet: 60%
    opus: 10%
---

# Dependency Lifecycle
<!-- Updated: June 2026 -->

Audit, upgrade, or add npm dependencies for a web project across the package managers this plugin supports: npm, pnpm, Yarn, and Bun.

Three subcommands select the operation from the first argument:

- **`deps audit [scope]`** — outdated-versions report, CVE lookup, and license inventory. Read-only assessment; changes nothing. See **Audit** below.
- **`deps upgrade <package>`** — advance exactly one dependency one step, pinning a concrete version and re-running the build and tests before touching the next. See **Upgrade** below.
- **`deps add <package>`** — introduce a new pinned dependency into the right manifest. See **Add** below.

**Dispatch**: parse the first token of `$ARGUMENTS`. If it is `audit`, run the Audit workflow with the remaining args as scope. If it is `upgrade`, run the Upgrade workflow with the next token as the target package. If it is `add`, run the Add workflow with the next token as the package to install. If the first token is none of the three (empty, a flag such as `--prod`, or a bare path), default to `audit` and treat the whole argument string as scope — never guess at a mutating mode.

> **Tool discipline:** `audit` is read-only. This command's `allowed-tools` includes `Edit` because `upgrade` and `add` need to write manifests; the audit workflow MUST NOT modify any file — it only reports findings.

[Extended thinking: Dependency changes are the highest-blast-radius edits in a web project — one transitive bump can break the build, drop a type, ship a CVE, or pull a tree-shake regression into the bundle. This command separates read-only assessment (audit) from mutation (upgrade/add) and forces upgrades through a one-dependency, pin, build-and-test-gated loop. Manager discovery is shared with `skill: language-detection`; CVE lookup uses the manager's native `audit` and cross-checks the osv.dev API; license inventory is best-effort and never blocks. Security findings are phrased in `igrsoft:security-review-process` vocabulary so they flow cleanly into an SR stage. The heavy reasoning — version-jump risk, breaking-change analysis, peer-dep resolution — is routed to `frontend-developer:fe-dependency-manager`; this command owns discovery, the gate loop, and reporting.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Audit is strictly read-only.** In `audit` mode, do NOT edit any manifest, lockfile, or source. Discovery, queries, and reporting only. If the user wants changes, they re-run with `upgrade` or `add`.
2. **Upgrade ONE dependency at a time.** Never batch upgrades. Pin the new version, then run the build+test gate (`/frontend-developer:build-test`) before proposing the next dependency. A failed gate stops the loop — report and wait.
3. **Cross a single major at a time.** A v1→v3 jump advances v1→v2 first (verified green), then v2→v3 — never two majors in one step. Pin a concrete version; never introduce `*`, `latest`, or a moving tag in the manifest (a caret range resolved by the lockfile is fine, but record the exact resolved version).
4. **Single-command Bash invocations.** Use each manager's own flags (`npm --prefix <path>`, `pnpm --dir <path>`, `yarn --cwd <path>`). Never `cd`-chain or `&&`-chain — scoped Bash patterns do not match compound commands.
5. **Tool-missing never hard-fails.** If a manager or scanner binary is absent, print the install hint, skip that pass, and continue. Report what was skipped. A missing native scanner falls back to the manager's own `audit` plus the osv.dev API — never skip the CVE pass silently.
6. **Route the reasoning, own the loop.** Hand version-jump risk, breaking-change analysis, and peer-dependency resolution to `frontend-developer:fe-dependency-manager`. This command performs discovery, runs the build+test gate, and synthesizes the report.
7. **Security findings use SR vocabulary.** Phrase every CVE/advisory in `igrsoft:security-review-process` terms (severity, advisory id, affected range, fixed-in version, remediation) so the output is consumable by an SR stage.
8. **Commands route, they do not orchestrate.** This command names `frontend-developer:fe-dependency-manager` so Claude routes the analysis; it does not call `Task`.
9. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Read-only audit of the current project (auto-detect the manager)
/frontend-developer:deps audit

# Audit only production dependencies
/frontend-developer:deps audit --prod

# Upgrade a single dependency one step, with a build+test gate
/frontend-developer:deps upgrade react --manager pnpm

# Add a new pinned dependency to the detected manifest
/frontend-developer:deps add zod
```

If no mode is given, default to `audit`.

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `audit` | (default) | Read-only: outdated report + CVE lookup + license inventory. No edits. |
| `upgrade <package>` | — | Advance one dependency one step; pin; run the build+test gate. `<package>` is required. |
| `add <package>` | — | Add a new pinned dependency to the detected (or `--manager`-selected) manifest. |
| `--manager npm\|pnpm\|yarn\|bun` | auto | Force the manager instead of detecting from the lockfile. |
| `--prod` | off | Restrict to production `dependencies` (skip `devDependencies`). |

## Manager Discovery (lockfile priority)

Scan the project root and record the manager from its lockfile. The marker → manager mapping is canonical in `skill: language-detection`; keep this in sync with it.

| Priority | Lockfile | Manager | Outdated query | Audit query |
|----------|----------|---------|----------------|-------------|
| 1 | `pnpm-lock.yaml` | pnpm | `pnpm outdated` | `pnpm audit --json` |
| 2 | `yarn.lock` | Yarn | `yarn outdated` (or `yarn upgrade-interactive --dry-run`) | `yarn npm audit --json` |
| 3 | `bun.lockb` | Bun | `bun outdated` | `bun audit` (verify against your toolchain) |
| 4 | `package-lock.json` | npm | `npm outdated --json` | `npm audit --json` |

A corepack-pinned `packageManager` field in `package.json` overrides the lockfile heuristic. `--manager` forces the choice. If no lockfile exists, default to npm and note the missing lockfile.

## Mode 1: Audit (read-only)

Produces three sections: **Outdated**, **Vulnerabilities (CVE)**, **Licenses**. No edits.

### Phase 1: Discover

Run Manager Discovery. If `--manager` is set, use it. If no `package.json`, emit the "no manifest" error and stop.

### Phase 2: Outdated Report (Bash)

Run the manager's outdated query (read-only). Capture the manager's exit status. With `--prod`, scope to production deps. Classify each as patch/minor/major.

### Phase 3: CVE Lookup

Use the manager's native audit, then cross-check osv.dev. **Never skip this pass silently.**

1. Run the manager's `audit --json` (table above). Parse advisories: severity, advisory id (GHSA/CVE), affected range, patched version.
2. **Cross-check** each flagged `(name, version)` against the osv.dev API via WebFetch when the native audit is sparse or the manager lacks audit:
   - URL: `https://api.osv.dev/v1/query`
   - Body: `{"package": {"ecosystem": "npm", "name": "<name>"}, "version": "<version>"}`
3. **Normalize into SR vocabulary** (per `igrsoft:security-review-process`): `severity` (Critical/High/Medium/Low), `advisory id`, `affected range`, `fixed-in version`, `remediation` (upgrade target). Group Critical/High at the top.

### Phase 4: License Inventory (best-effort)

Best-effort; never blocks.
- `npx license-checker --json` (or `pnpm licenses list --json` / `npx license-report`) to inventory each package's license.
- Flag any GPL/AGPL/SSPL or otherwise copyleft license against a permissive project as a review item (not a hard failure) — phrase it as a license-compatibility finding for SR.

### Phase 5: Route analysis & report

Route the raw discovery + queries to the dependency manager for risk framing:

Route to `frontend-developer:fe-dependency-manager`:
"Audit-mode dependency analysis for the project at `{path}` ({manager}). Outdated report:\n```\n{outdated_output}\n```\nCVE findings (raw):\n```\n{cve_output}\n```\nLicenses:\n```\n{license_output}\n```\nFor each outdated dependency, classify the jump (patch/minor/major), note documented breaking changes and peer-dep constraints, and assess upgrade risk. Normalize every vulnerability into `igrsoft:security-review-process` vocabulary. Produce a prioritized upgrade plan (security patches first, then patch/minor, then majors individually). Do NOT edit files — read-only audit."
Synthesize the agent's analysis into the Output Format report.

## Mode 2: Upgrade (one dependency, gated)

Advances exactly one dependency one step: prep, pin, build+test gate, then stop.

### Phase 1: Resolve target

1. Require `<package>`. Run Manager Discovery (or `--manager`).
2. Run the Phase 2/3 queries scoped to `<package>` to learn current version, latest version, peer-dep constraints, and any open CVE.

### Phase 2: Plan the single step (route)

Route to `frontend-developer:fe-dependency-manager`:
"Plan a single-step upgrade of `{package}` ({manager}) in `{path}` from `{current}` toward `{target}`. If the jump crosses a major, advance only ONE major (v1→v2, never v1→v3) and identify the exact next version to pin. Summarize documented breaking changes between `{current}` and the target, list peer-dep impacts (e.g. a `react` bump that forces `react-dom`/`@types/react`), and produce the exact manifest edit + install command. Return a concrete diff plan; do not apply yet."

### Phase 3: Apply the pinned edit

Apply the agent's edit and update the lockfile via the manager:

| Manager | Upgrade command |
|---------|-----------------|
| npm | `npm install <pkg>@<x.y.z> --prefix <path>` (updates `package-lock.json`) |
| pnpm | `pnpm --dir <path> add <pkg>@<x.y.z>` (or `pnpm update <pkg> --latest`) |
| Yarn | `yarn --cwd <path> up <pkg>@<x.y.z>` |
| Bun | `bun add <pkg>@<x.y.z> --cwd <path>` |

Bump coupled peer deps in the same step where the agent flagged them (e.g. `react` + `react-dom` together) — that is still "one logical dependency."

### Phase 4: Build + Test Gate (BINDING)

- Invoke `/frontend-developer:build-test <path>` (single dependency upgraded).
- **Gate decision:**
  - **PASS** (type-check + build + test all green) → report the step. Do NOT auto-continue; the user re-runs `upgrade` for the next.
  - **FAIL** → STOP. Report the failing stage and triage. Offer to roll back the single edit (revert the manifest + lockfile, `npm ci`-equivalent) and, if the failure is a code-level break from the new version, route the excerpt to the owning framework agent for a migration patch — then re-run the gate once.

### Phase 5: Report the step

Emit the "Upgrade Step" block with from→to, the gate result, and the next recommended dependency (but do not start it).

## Mode 3: Add (new pinned dependency)

### Phase 1: Resolve manager & manifest

Require `<package>`. Auto-detect the manager (or `--manager`). Decide `dependencies` vs `devDependencies` (build-time tooling → dev).

### Phase 2: Resolve a pinnable version

Look up the latest stable release (`npm view <pkg> version` or the registry via WebFetch) and its CVE status (Phase 3 CVE lookup, scoped). Do not add a version with an unremediated Critical/High advisory without flagging it.

### Phase 3: Add, pinned (route the edit)

Route to `frontend-developer:fe-dependency-manager`:
"Add `{package}` at version `{version}` to the {manager} manifest at `{path}` as a {dependency|devDependency}. Write the minimal entry and the install command ({npm install / pnpm add / yarn add / bun add}). Flag any peer-dep requirements and any type-package (`@types/{package}`) needed. Return the edit; do not add duplicates already present."

### Phase 4: Build + Test Gate

Run `/frontend-developer:build-test <path>`. PASS → report the addition. FAIL → roll back and report.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint |
|--------------|--------------|
| `node` / `npm` | install Node.js LTS (`https://nodejs.org` or `brew install node`) |
| `pnpm` | `npm install -g pnpm` (or `corepack enable pnpm`) |
| `yarn` | `corepack enable` or `npm install -g yarn` |
| `bun` | `curl -fsSL https://bun.sh/install \| bash` (verify against your toolchain) |
| `license-checker` | `npm install -g license-checker` (or `npx license-checker`) — without it, the license inventory is skipped with a note |

A missing native `audit` (e.g. on older Bun) falls back to the osv.dev API via WebFetch, not a skipped CVE pass. A missing license tool skips only the license section with a note. Never hard-fail on a single missing tool: print the hint, skip that pass, continue, and report the skip.

## Output Format

```markdown
## Dependency Audit Report

**Target:** {path}
**Mode:** {audit | upgrade | add}
**Manager:** {npm | pnpm | yarn | bun} ({lockfile})
**Scope:** {all | --prod}

### Outdated
| Package | Current | Latest | Jump | Risk |
|---------|---------|--------|------|------|
| react | 18.3.1 | 19.1.0 | major | review breaking changes + react-dom |
| zod | 3.23.0 | 3.24.1 | minor | low |

### Vulnerabilities (SR vocabulary)
| Severity | Advisory | Package | Affected | Fixed in | Remediation |
|----------|----------|---------|----------|----------|-------------|
| High | GHSA-xxxx-xxxx | <pkg> | <range> | <x.y.z> | upgrade to <x.y.z> |

(If none: "No known vulnerabilities in the queried versions via {npm/pnpm audit | osv.dev}.")

### Licenses
| Package | License | Note |
|---------|---------|------|
| <pkg> | MIT | compatible |
| <pkg> | GPL-3.0 | copyleft — review compatibility (SR item) |

### Prioritized Upgrade Plan
1. **Security first:** {pkg} {cur}→{fixed} (advisory {id})
2. {pkg} {cur}→{tgt} (patch/minor)
3. {pkg} major upgrades — individually, one at a time

<!-- upgrade/add modes only -->
### Upgrade Step
- **Package:** {pkg} ({manager})
- **From → To:** {current} → {target}
- **Coupled peers:** {react-dom, @types/react — or none}
- **Manifest edits:** {files changed}
- **Build + Test gate:** PASS / FAIL ({failing stage})
- **Next recommended:** {pkg} (run `/frontend-developer:deps upgrade {pkg}`)

### Skipped
- {tool}: {missing} — install hint printed above.
```

## Error Handling

### No manifest found
```
Error: No package.json detected under {path}.
Suggestion: Run from the project root, or scaffold one with `npm init`.
```

### Package not in manifest (upgrade)
```
Error: {package} is not a declared dependency.
Suggestion: Use `add {package}` to introduce it, or check the spelling against package.json.
```

### Major version jump requested
Not an error — the command advances ONE major at a time (v1→v2 verified green, then v2→v3). It will complete the first major (built, tested) before proposing the next (Rule 3).

### Build+test gate failed after upgrade
Not silent. Report the failing stage from `/frontend-developer:build-test`, offer to roll back the single manifest + lockfile edit, and (for a code-level break) route the excerpt to the owning framework agent for a migration patch before re-running the gate once.

### Tool missing
Print the install hint, skip that pass, continue. A missing native audit falls back to osv.dev. The command only reports a hard stop when *every* eligible pass was skipped.

## See Also

- `skill: language-detection` — canonical lockfile → manager → agent routing (keep discovery in sync).
- `skill: build-systems` — workspace/monorepo dependency idioms, peer-dep resolution.
- `skill: secure-coding` — npm supply-chain and dependency-trust rules that gate a diff.
- `/frontend-developer:build-test` — the build+test gate this command invokes after every upgrade/add.
- `/frontend-developer:fix-modernize` — when a major upgrade needs a framework-idiom migration (e.g. React 18→19 patterns).
- `igrsoft:security-review-process` — SR-stage vocabulary used for every vulnerability finding here.
