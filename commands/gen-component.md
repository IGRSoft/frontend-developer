---
description: Scaffold a component (props, state, test, and story) in the project's detected framework and conventions
argument-hint: [ComponentName] [path (default: detected components dir)] [--with-story] [--no-test]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
estimated-cost:
  min-tokens: 500
  max-tokens: 6000
  model-distribution:
    haiku: 85%
    sonnet: 15%
---

# Component Scaffold
<!-- Updated: June 2026 -->

Generate a new component in the project's framework — the component file (typed props + minimal state), a colocated test, and (optionally) a Storybook story — following the conventions the project already uses (file naming, directory layout, styling approach, test runner). Fast and deterministic: this is a boilerplate generator, not a feature implementation. Real behavior is implemented afterward by the framework agent.

[Extended thinking: A scaffold is only useful if it matches what the project already does — the wrong file extension, a CSS approach the repo abandoned, or a test in a runner the project does not use creates cleanup work instead of saving it. So this command's first job is detection: it reads `package.json` and a sample existing component to learn the framework, the styling convention (CSS Modules vs Tailwind vs styled-components vs plain CSS), the test runner/renderer, and the directory pattern (`ComponentName/index.tsx` vs `ComponentName.tsx`). Only then does it emit files that look like they belong. It writes a typed, prop-driven shell with sensible defaults and an accessible root element, a test that renders and asserts on a role, and — with `--with-story` — a Storybook story. It deliberately stops at boilerplate: the actual logic, data fetching, and styling are the framework agent's job, and the command points the user there. When detection fails (empty/greenfield project), it emits a clearly-labeled framework-default stub rather than guessing wrong.]

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Match the project's conventions, do not impose.** Detect the framework, styling approach, test runner/renderer, and directory layout from `package.json` and a sample existing component before writing anything. Emit files that match the existing pattern (extension, naming, import style, styling). When in doubt, read a sibling component and mirror it.
2. **Scaffold boilerplate only — no behavior.** Generate a typed, prop-driven shell with an accessible root element, minimal local state, and TODO markers for logic. Do NOT implement data fetching, business logic, or real styling. Point the user to the framework agent for the implementation.
3. **Do not overwrite an existing component.** If `ComponentName` already exists at the target path, STOP and report — never clobber. The user renames or removes first.
4. **Typed props, accessible root.** The generated component declares a typed props interface and renders a semantically appropriate, labelable root element (not a bare `<div>` for an interactive component). The test asserts on an accessible role/name, not a test id.
5. **Single-command Bash invocations.** Use the framework CLI's own flags where one exists (`ng generate component`); otherwise write files directly. Never `cd`-chain or `&&`-chain — scoped Bash patterns do not match compound commands.
6. **Tool-missing never hard-fails.** If a framework CLI (`ng`) is unavailable, print the install hint and fall back to writing the files from the built-in template. Never abort over a missing optional CLI.
7. **This is a generator, not a developer.** Do NOT route to an agent to *implement* the component — the scaffold is intentionally inert. (The user invokes the framework agent next for behavior.)
8. **Never enter plan mode.** This command IS the procedure — execute it.

## Usage

```bash
# Scaffold a component in the detected components directory
/frontend-developer:gen-component UserCard

# Scaffold into a specific path
/frontend-developer:gen-component PriceTag src/features/pricing

# Scaffold with a Storybook story
/frontend-developer:gen-component Badge --with-story

# Scaffold without a test (rare; prefer keeping the test)
/frontend-developer:gen-component Spinner --no-test
```

## Options

| Option | Default | Effect |
|--------|---------|--------|
| `ComponentName` | required | PascalCase component name. The command derives file names from it per the project's convention. |
| `path` | detected components dir | Where to create the component. Defaults to the directory where existing components live (e.g. `src/components`). |
| `--with-story` | off | Also emit a Storybook story (`*.stories.tsx`) when Storybook is detected (or the framework default if requested explicitly). |
| `--no-test` | off | Skip the colocated test file (not recommended — the test is part of the deliverable by default). |

## Detection

Read `package.json` and one sample existing component to learn the conventions. Resolve framework via `skill: language-detection`.

| Aspect | Detect by | Drives |
|--------|-----------|--------|
| Framework | `react`/`vue`/`svelte`/`@angular/core` in `package.json` | file extension + component syntax |
| Styling | `*.module.css` siblings (CSS Modules), `tailwind.config` (Tailwind), `styled-components`/`@emotion` in deps, plain `*.css` imports | how the root element is styled in the stub |
| Test runner/renderer | `vitest`/`jest` + `@testing-library/*` (see `/frontend-developer:gen-tests` detection) | the test file's imports + render call |
| Directory layout | sample component path: `Comp/index.tsx` vs `Comp.tsx` vs `comp.vue` | the generated file path |
| Storybook | `@storybook/*` in deps; `.storybook/` dir | whether `--with-story` uses the project config |

If detection finds no framework (greenfield), pick the framework-default stub (React + TypeScript + CSS Modules + Vitest/Testing Library) and **label it clearly** in the report as a default the user can change.

## Generated Files (per framework)

The exact extension/syntax follows detection; these are the shapes.

### React (`.tsx`)

```tsx
// {Component}.tsx
import styles from './{Component}.module.css'; // or Tailwind classes / styled-components per detection

export interface {Component}Props {
  /** TODO: document props */
  label: string;
}

/**
 * {Component} — TODO: describe.
 */
export function {Component}({ label }: {Component}Props) {
  // TODO: state and behavior — implement via frontend-developer:react-developer
  return (
    <section className={styles.root} aria-label={label}>
      {label}
    </section>
  );
}
```

### Vue (`.vue`, `<script setup>`)

```vue
<script setup lang="ts">
interface Props {
  /** TODO: document props */
  label: string
}
const props = defineProps<Props>()
// TODO: state and behavior — implement via frontend-developer:vue-developer
</script>

<template>
  <section class="root" :aria-label="props.label">{{ props.label }}</section>
</template>
```

### Svelte (`.svelte`, Svelte 5 runes)

```svelte
<script lang="ts">
  // TODO: implement via frontend-developer:svelte-developer
  let { label }: { label: string } = $props();
</script>

<section class="root" aria-label={label}>{label}</section>
```

> Requires Svelte 5 runes (`$props`) (Svelte 5+). Fallback: `export let label: string;` on Svelte 4. Canonical: _shared/version-feature-matrix.md

### Angular (standalone)

Prefer the CLI when present: `npx ng generate component <name> --standalone --inline-style=false`. Otherwise write a standalone component (`standalone: true`, typed `@Input()`/`input()` per version) + template + spec. The Angular CLI emits the spec and (with config) the story.

> Requires standalone components (Angular 15+; standalone-by-default 19+). Fallback: declare in an NgModule on older Angular. Canonical: _shared/version-feature-matrix.md

### Test (Vitest + Testing Library shape)

```tsx
// {Component}.test.tsx
import { render, screen } from '@testing-library/react'; // framework renderer per detection
import { {Component} } from './{Component}';

test('renders with its label', () => {
  render(<{Component} label="Example" />);
  expect(screen.getByRole('region', { name: 'Example' })).toBeInTheDocument();
});
```

### Story (`--with-story`, Storybook CSF3)

```tsx
// {Component}.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { {Component} } from './{Component}';

const meta: Meta<typeof {Component}> = { component: {Component} };
export default meta;
export const Default: StoryObj<typeof {Component}> = { args: { label: 'Example' } };
```

## Workflow

### Phase 1: Detect (Bash + Read)

1. Validate `ComponentName` is PascalCase; if not, normalize and note the normalization.
2. Read `package.json`; resolve the framework, styling, test runner/renderer, Storybook presence, and directory layout (Detection). If no framework, select the labeled default.
3. Resolve the target path (explicit `path` or the detected components dir). Compute the file names per the project's naming convention.

### Phase 2: Collision Check (Bash)

1. Check whether the component file already exists at the target path. If it does, STOP and emit the Error Handling "already exists" message — never overwrite (Rule 3).

### Phase 3: Generate (Write, or framework CLI)

1. For Angular, prefer `npx ng generate component` when the CLI is present (it wires the spec + standalone metadata). For React/Vue/Svelte, write the files directly from the matching template, substituting `ComponentName`, the detected styling approach, and the detected test renderer.
2. Emit: the component file, the colocated test (unless `--no-test`), and the story (with `--with-story` and Storybook detected). Place each at the convention path.

### Phase 4: Report (Bash)

Emit the Output Format. List the files written, the conventions detected, and the next step (implement behavior via the framework agent).

## Graceful Degradation

If a tool is not installed: print the install hint and skip this step. Never exit non-zero from a missing optional tool.

| Missing tool | Install hint | Fallback |
|--------------|--------------|----------|
| `node` / `npx` | install Node.js LTS (`https://nodejs.org` or `brew install node`) | write files from the built-in template directly (no CLI needed) |
| `@angular/cli` (`ng generate`) | `npm install -D @angular/cli` then `npx ng ...` | write the standalone component + spec from the built-in template |
| `@storybook/*` (for `--with-story`) | `npx storybook@latest init` | emit a `*.stories.tsx` stub anyway, with a note that Storybook is not yet installed |

If the framework CLI is missing, print the hint and **always** fall back to writing the files from the built-in template — scaffolding never requires the CLI. With `--with-story` but no Storybook, emit the story stub and note the missing setup. Never hard-fail on a missing optional tool: print the hint, fall back, continue, and report.

## Output Format

```markdown
## Component Scaffold Report

**Component:** {ComponentName}
**Framework:** {React | Vue | Svelte | Angular} {(detected | framework default — change as needed)}
**Conventions:** styling: {CSS Modules | Tailwind | styled-components | plain CSS} · test: {Vitest + Testing Library | Jest} · layout: {Comp/index.tsx | Comp.tsx}

### Files Created
| File | Purpose |
|------|---------|
| src/components/{Comp}.tsx | component shell (typed props, accessible root, TODO behavior) |
| src/components/{Comp}.test.tsx | renders + asserts on role/name |
| src/components/{Comp}.stories.tsx | Storybook story (--with-story) |
| src/components/{Comp}.module.css | styling stub (per detected approach) |

### Next Step
Implement behavior and styling:
- Logic/state → route to frontend-developer:{framework}-developer
- Tests → /frontend-developer:gen-tests src/components/{Comp} --type component
- Verify it builds → /frontend-developer:build-test .

<!-- on a degraded run -->
### Notes
- {framework CLI missing — wrote from template | Storybook not installed — story stub emitted}
```

## Error Handling

### Component already exists
```
Error: {ComponentName} already exists at {path}.
Refusing to overwrite. Suggestion: pick a different name, or remove/rename the existing
component first.
```

### Invalid component name
```
Note: '{input}' normalized to PascalCase '{ComponentName}'.
(Component names are PascalCase by convention.)
```

### No framework detected (greenfield)
```
Note: No framework detected under {path}. Scaffolding a React + TypeScript + CSS Modules
default (Vitest + Testing Library). Change the template by running inside an existing
framework project, or edit the generated files.
```

### Tool missing
Print the install hint, fall back to the built-in template, continue. Scaffolding never hard-fails on a missing CLI — the templates are self-contained.

## See Also

- `skill: language-detection` — framework detection that drives the file extension and syntax.
- `/frontend-developer:gen-tests` — flesh out the colocated test into a full suite once the component has behavior.
- `/frontend-developer:build-test` — confirm the scaffold builds and type-checks.
- `/frontend-developer:review-code` — review the component once behavior is implemented.
- `frontend-developer:react-developer` / `vue-developer` / `svelte-developer` / `angular-developer` — implement the component's behavior after scaffolding.
- `skill: accessibility-patterns` — make the accessible root element correct for the component's interaction model.
