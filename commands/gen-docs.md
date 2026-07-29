---
description: Generate or update TSDoc comments, typedoc API reference, Storybook docs, and README API sections
argument-hint: [path or scope (default: src/)] [--check] [--api] [--stories] [--readme]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
estimated-cost:
  min-tokens: 3000
  max-tokens: 18000
  model-distribution:
    haiku: 20%
    sonnet: 70%
    opus: 10%
---

# Code Documentation Generator
<!-- Updated: July 2026 -->

Document a web codebase where it actually pays: TSDoc comments on the exported public API, prop-interface docs on components, explanation on the non-obvious hooks and composables — then wire those comments into the generated artifacts (typedoc API reference, Storybook autodocs, README API section). Documentation work routes to `frontend-developer:typescript-developer`, which owns the TSDoc surface and the exported type contracts.

[Extended thinking: The failure mode of doc generation is volume — a `@param name The name` on every symbol in the repo, which costs review time and teaches nothing. So this command is priority-ordered and evidence-driven: it first resolves what is actually *exported* (the package entry points, `index.ts` barrels, `package.json` `exports`), documents that surface completely, then component props interfaces (because those become Storybook autodocs and typedoc tables for free), then the hooks/composables whose behavior is non-obvious — subscription lifetime, cache invalidation, SSR safety. Internal one-liners are left alone. The second job is making the comments *load-bearing*: typedoc reads TSDoc directly, Storybook autodocs reads the props interface's TSDoc, and the README API section is generated from the same source, so one well-written comment feeds three artifacts. `--check` is the CI mode — it reports the undocumented exported surface without writing, so a coverage gate can fail a PR without this command mutating anything.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Document in priority order.** Exported public API first, then component props interfaces, then complex hooks/composables. Do NOT document private internals, trivial one-line helpers, or re-exports before the exported surface is complete. Report the priority tier each documented symbol belonged to.
2. **Explain the contract, never restate the name.** `@param userId The user id` is a failure. Every `@param`/`@returns`/`@throws` must add information the signature does not already carry — units, ranges, ownership, null semantics, failure modes, side effects.
3. **`--check` never writes.** In `--check` mode, list the undocumented exported symbols and the coverage number, and exit without editing a single file. This is the CI mode.
4. **Comments only — never change behavior.** This command adds and edits TSDoc blocks, doc pages, and README sections. It MUST NOT rename symbols, change signatures, reorder exports, or "clean up" code it is documenting. If a symbol is undocumentable because its behavior is unclear, list it under "Needs author input" rather than guessing.
5. **Every code example must be real.** `@example` blocks use the actual imported symbol with real argument shapes taken from the code or its tests. Do NOT invent APIs, options, or return shapes. If no honest example exists, omit `@example`.
6. **Detect the doc stack, do not impose one.** Use typedoc only if it is present or the user passes `--api`; use Storybook CSF3/autodocs only if Storybook is detected or `--stories` is passed. Never add a new toolchain to a project as a side effect of documenting it.
7. **Tool-missing never hard-fails.** If `typedoc`, `storybook`, or `npx` is unavailable, print the install hint, skip that artifact, and continue with the TSDoc pass. Never abort the whole run over one missing optional tool.
8. **Verify the project still builds.** After writing, run `/frontend-developer:build-test` (or at minimum `npx tsc --noEmit`) so a malformed TSDoc block or a broken doc import cannot land silently.
9. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Document the default scope (src/) — TSDoc pass + README API section
/frontend-developer:gen-docs

# Document a specific directory or file
/frontend-developer:gen-docs src/lib
/frontend-developer:gen-docs src/hooks/useCart.ts

# CI coverage gate — report undocumented exports, write nothing
/frontend-developer:gen-docs src/ --check

# Regenerate the typedoc API reference after the TSDoc pass
/frontend-developer:gen-docs src/ --api

# Document components and refresh their Storybook stories/autodocs
/frontend-developer:gen-docs src/components --stories

# Full pass: TSDoc + typedoc + Storybook + README
/frontend-developer:gen-docs src/ --api --stories --readme
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `scope` | `src/` | File, directory, or `all`. A file documents that file; a directory documents the exported surface reachable from it; `all` documents every package entry point. |
| `--check` | off | Report-only coverage gate. Lists undocumented exported symbols and the coverage percentage. Writes nothing (Rule 3). |
| `--api` | auto | Generate/refresh the typedoc API reference. Auto-enabled when `typedoc` is in `devDependencies` or a `typedoc.json` exists. |
| `--stories` | auto | Add or refresh Storybook CSF3 stories and `autodocs` for documented components; write a `.mdx` docs page for a component that needs prose. Auto-enabled when Storybook is detected. |
| `--readme` | on | Update the README / `docs/api.md` API-reference section from the documented exports. Pass `--readme` explicitly to force it when the scope is a single file. |

## Detection

Resolve the documentation stack from `package.json` and the repo layout before writing anything. Record what was found in the report.

| Signal | Meaning | Artifact produced |
|--------|---------|-------------------|
| `exports` / `main` / `types` in `package.json`, `src/index.ts` barrel | package public API surface | TSDoc on every exported symbol (tier 1) |
| `typedoc` in devDependencies, `typedoc.json`, `typedoc` script | typedoc is the API-reference generator | `npx typedoc` output under the configured `out` dir |
| `.storybook/` dir, `@storybook/*` in devDependencies | Storybook documents components | CSF3 stories + `tags: ['autodocs']`, optional `*.mdx` |
| `*.tsx`/`*.vue`/`*.svelte` with an exported props type | component props interface | TSDoc on props → autodocs table + typedoc table |
| `use*.ts` / `composables/` / `*.store.ts` | hooks, composables, stores | TSDoc with lifetime/SSR/invalidation semantics (tier 3) |
| `README.md`, `docs/api.md` | prose entry point | regenerated API-reference section |

Framework and package-manager detection follow `skill: language-detection` — do not fork that routing logic here.

## Priority Order

Document top-down; finish a tier before starting the next. Report per-tier coverage.

1. **Exported public API** — everything reachable from the package entry points (`exports`, `main`, `types`, `src/index.ts`). Functions, classes, types, constants. This tier gets the full treatment: summary, `@param`, `@returns`, `@throws`, `@example`, `@deprecated` where applicable.
2. **Component props interfaces** — every exported component's props type. Each prop gets a one-line contract (what it controls, its default, whether it is required for accessibility). These comments become the Storybook autodocs table and the typedoc props table, so they are read far more often than they are written.
3. **Complex hooks, composables, and stores** — anything with non-obvious behavior: subscription lifetime and cleanup, cache/invalidation semantics, SSR/hydration safety, re-render triggers, cancellation. Document the *behavior contract*, not the implementation.

Everything below tier 3 (internal helpers, trivial wrappers, obvious getters) is deliberately out of scope. Comment the non-obvious WHY and the contract — never the WHAT.

## TSDoc Conventions

Use `/** ... */` block comments directly above the symbol. Supported tags, in this order:

| Tag | Use for |
|-----|---------|
| summary (untagged first line) | One sentence, imperative, ends with a period. |
| `@remarks` | Extended discussion that does not belong in the one-line summary. |
| `@param name` | Each parameter: meaning, units/range, null semantics, ownership. |
| `@returns` | What comes back, including the empty/absent case. |
| `@throws` | Each error type and the condition that raises it. Include rejected-promise cases. |
| `@example` | A real, copy-pasteable call in a fenced ` ```ts ` block. |
| `@deprecated` | Why, since when, and the replacement symbol — always name the replacement. |
| `@see` | Cross-reference with `{@link OtherSymbol}`. |

```ts
/**
 * Fetch a cart and subscribe to its server-side updates.
 *
 * @remarks
 * The subscription is torn down on unmount; calling this from a Server
 * Component throws, because it opens an `EventSource`.
 *
 * @param cartId - Stable cart identifier. Changing it resubscribes.
 * @param options - Polling fallback used when `EventSource` is unavailable.
 * @returns The cart, plus `isStale` while a refetch is in flight. `cart` is
 * `null` until the first response resolves.
 * @throws {CartNotFoundError} When the id resolves to no cart (HTTP 404).
 * @throws {TypeError} When called outside a browser environment.
 *
 * @example
 * ```ts
 * const { cart, isStale } = useCart('cart_123', { pollMs: 5000 });
 * ```
 *
 * @deprecated Since 4.2 — use {@link useCartQuery} instead; this hook will be
 * removed in 5.0.
 */
```

Notes:
- TSDoc uses `@param name - description` (hyphen separator). Do not restate the TypeScript type in prose — the type is already in the signature and typedoc renders it.
- For Vue, document the `defineProps` type or the exported props interface; for Svelte, document the `$props()` type. The comment still lives in a `/** */` block above the declaration.
- Public-facing `@example` blocks must type-check. A malformed example that breaks `tsc` is caught by the Rule 8 build gate.

## Workflow

### Phase 1: Resolve scope and stack (Bash)

1. Confirm `scope` exists; if not, emit the Error Handling "path not found" message and stop.
2. Read `package.json`: entry points, `typedoc`/Storybook presence, package manager, framework.
3. Enumerate the **exported** symbols in scope (barrel files, `export` statements, `package.json` `exports`). This list is tier 1 and is the denominator for the coverage number.
4. Classify the remaining in-scope symbols into tier 2 (component props) and tier 3 (hooks/composables/stores). Everything else is out of scope.
5. Print the resolved file list, the tier counts, and the detected doc stack before writing.

### Phase 2a: `--check` (report-only)

1. For each tier, count symbols with a usable TSDoc block versus the total. A block that only restates the name counts as **undocumented** (Rule 2).
2. Emit the coverage table and the undocumented-symbol list. Write nothing. Result is FAIL if tier 1 coverage is below 100%, otherwise PASS with the tier 2/3 numbers reported.

### Phase 2b: TSDoc pass (default)

Route to `frontend-developer:typescript-developer` with the resolved tiers:

"Write TSDoc for these symbols, tier 1 first: {tier1_symbols}, then {tier2_props}, then {tier3_hooks}. Use `/** */` blocks with `@param name - …`, `@returns`, `@throws`, `@example`, `@deprecated` per the project's TSDoc conventions. Every description must add information the signature does not carry — units, ranges, null semantics, failure modes, lifetime, SSR safety. Do NOT restate names, and do NOT change any code: comments only. `@example` blocks must use the real API and type-check. If a symbol's behavior cannot be determined from the code or its tests, list it under 'Needs author input' instead of guessing."

For framework-specific props and template semantics, the framework agent that owns the file (`frontend-developer:react-developer` / `vue-developer` / `svelte-developer` / `angular-developer`) reviews the tier 2 comments; `frontend-developer:css-developer` is not involved.

### Phase 3: Generated artifacts

Run only the artifacts that detection (or an explicit flag) enabled.

**typedoc (`--api`)**
- Generate with `npx typedoc` (honoring `typedoc.json` / the `typedoc` key in `package.json`). With no config, `npx typedoc --out docs/api src/index.ts` is the documented default.
- Treat typedoc warnings about unresolved `{@link}` targets and undocumented exports as findings — they are the mechanical check on the Phase 2b pass.

**Storybook (`--stories`)**
- For each documented component, ensure a CSF3 story file exists: a default export with `component`, `title`, and `tags: ['autodocs']`, plus named `export const` stories with `args`.
- Autodocs renders the props table straight from the tier 2 TSDoc — do not duplicate prop descriptions into `argTypes` unless a control needs configuring.
- Add a `*.mdx` docs page only when a component needs prose that does not fit in autodocs (usage guidance, do/don't, composition patterns), using `<Meta of={...} />` to attach it to the story.

**README / API reference (`--readme`)**
- Keep the existing README structure; update in place. Add or refresh the API-reference section from the tier 1 exports (signature + one-line summary + link to the typedoc page).
- Record breaking changes and every `@deprecated` symbol added in this pass.
- Do not restructure prose the author wrote.

### Phase 4: Verify

Run `/frontend-developer:build-test` over the touched package (or `npx tsc --noEmit` when only comments changed) so a malformed TSDoc block, a broken story import, or a bad `{@link}` cannot land silently. Report the result in the output.

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Effect |
|--------------|--------------|--------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | TSDoc pass still runs (source-only); every generated artifact is skipped |
| `typedoc` | `npm install -D typedoc` | skip the API-reference generation; TSDoc comments are still written |
| `storybook` / `@storybook/*` | `npx storybook@latest init` | skip stories and `.mdx`; props TSDoc is still written |
| `tsc` (no `tsconfig.json`) | `npm install -D typescript && npx tsc --init` | skip the type-check gate; note the reduced verification in the report |
| `prettier` | `npm install -D prettier` | comment blocks are written unformatted; note it |

Tool-missing is always a skip-with-note, never a hard failure. The command only reports "could not document" when *no* documentable symbols are in scope (not when a tool is missing).

## Output Format

```markdown
## Documentation Report

**Scope:** {resolved paths}
**Stack:** {TSDoc | + typedoc | + Storybook autodocs | + README}
**Mode:** {write | --check}

### Coverage

| Tier | Symbols | Documented | Coverage |
|------|--------:|-----------:|---------:|
| 1 — exported public API | {n} | {n} | {%} |
| 2 — component props | {n} | {n} | {%} |
| 3 — hooks / composables / stores | {n} | {n} | {%} |

**Result:** PASS / FAIL / PARTIAL

### Documented ({total})
| File:Line | Symbol | Tier | Tags added |
|-----------|--------|-----:|------------|
| src/lib/cart.ts:14 | `useCart` | 3 | summary, @param ×2, @returns, @throws ×2, @example |

### Artifacts
| Artifact | Result | Location |
|----------|--------|----------|
| typedoc API reference | ✅ / ⏭ skipped | docs/api/ |
| Storybook stories / autodocs | ✅ / ⏭ skipped | {N stories, M .mdx} |
| README API section | ✅ / ⏭ skipped | README.md |

### Verification
`/frontend-developer:build-test` → PASS / FAIL ({tsc errors, if any})

<!-- when a symbol could not be documented honestly -->
### Needs author input ({count})
- {file}:{line} `{symbol}` — {what is ambiguous: error conditions, ownership, SSR safety}

<!-- --check mode only -->
### Undocumented exports
- {file}:{line} `{symbol}` (tier {n})

<!-- when a tool was unavailable -->
### Skipped
- {tool}: unavailable — install hint printed above; {artifact} not generated.
```

## Error Handling

### Path not found
```
Error: Path not found: {scope}
Suggestion: Pass a file or directory that exists, e.g. /frontend-developer:gen-docs src/
```

### No documentable symbols
```
Note: No exported symbols, component props, or hooks found under {scope}.
Suggestion: Point at the package entry point (src/index.ts) or a components directory.
```

### No package.json
```
Error: No package.json found — cannot resolve the public API surface.
Suggestion: Run from the package root, or pass the workspace package path.
```

### Existing comments conflict with the code
Not an error. Keep the author's prose, correct only the factually wrong parts (renamed params, changed return shape, stale `@deprecated` targets), and list each correction in the Documented table.

## See Also

- `skill: typescript-skills` — canonical TypeScript selection table; public-API typing that the TSDoc describes.
- `skill: ts-typing` — discriminated unions, generics, and narrowing patterns worth documenting explicitly.
- `skill: fe-testing` — the test suite is where honest `@example` argument shapes come from.
- `/frontend-developer:gen-component` — scaffolds a component with a story; document it here afterward.
- `/frontend-developer:gen-tests` — tests are the honest source for `@example` blocks; generate them first when there are none.
- `/frontend-developer:build-test` — the Rule 8 verification gate after writing.
- `/frontend-developer:review-code` — escalate a symbol whose behavior is unclear enough that documenting it exposed a defect.

If the exported surface is already documented to standard, say that directly instead of padding the pass with restatement comments.
