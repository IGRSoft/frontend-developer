---
name: accessibility-patterns
description: >-
  Implementation patterns for accessible web UI — semantic-first markup, correct
  ARIA roles/states, keyboard interaction, focus management (traps, restoration,
  roving tabindex), live regions, and axe-core auditing. Use when building or
  auditing interactive components (dialogs, menus, tabs, comboboxes) for WCAG 2.2
  keyboard and screen-reader support.
---

# Accessibility Patterns

**Behavioral a11y: ARIA, focus, keyboard, screen-reader support.** For the domain
selection table, see the canonical [quality-skills/SKILL.md](../quality-skills/SKILL.md).
The numeric WCAG floors are canonical in
[accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md);
the CSS-layout side (contrast, reduced motion) is in
[responsive-accessible-css](${CLAUDE_SKILL_DIR}/styling/responsive-accessible-css/SKILL.md).
This leaf is the **interaction how-to** and does not duplicate those.

## When to Use

Use this skill when:
- Building interactive widgets: dialogs, menus, tabs, comboboxes, accordions, tooltips
- Adding ARIA and unsure whether it's correct (or needed at all)
- Implementing keyboard interaction and focus management
- Auditing a component with axe-core / for screen-reader support

## Rule 1: Semantic HTML first; ARIA only to fill gaps

The first rule of ARIA is **don't use ARIA** when a native element does the job. A
native `<button>`, `<a href>`, `<input>`, `<dialog>`, `<details>` ships keyboard
support, focus, and roles for free. Reach for ARIA only for patterns HTML lacks
(tabs, comboboxes, tree views) or to add state to custom widgets.

| Don't | Do |
|-------|-----|
| `<div role="button" tabindex="0" onClick>` | `<button>` |
| `<div role="link">` | `<a href="…">` |
| `<span onClick>` for navigation | `<a>` / `<button>` per intent |
| Custom checkbox from scratch | `<input type="checkbox">` styled |

**Bad ARIA is worse than no ARIA** — a wrong `role`/state misleads screen-reader
users. If you add a role, you own all its required states and keyboard behavior.

## ARIA: roles, states, properties

- **Accessible name:** every interactive control needs one — visible label,
  `aria-label`, or `aria-labelledby`. Icon-only buttons need `aria-label`.
- **State, not just style:** reflect `aria-expanded`, `aria-selected`,
  `aria-checked`, `aria-disabled`, `aria-current` so AT announces the real state.
- **Relationships:** `aria-controls`, `aria-describedby`, `aria-labelledby` wire
  parts together (a toggle to its panel, an input to its error text).
- **Hide decoration:** `aria-hidden="true"` on purely decorative icons (and never
  on focusable content).

Follow the **WAI-ARIA Authoring Practices** pattern for each widget — they specify
the exact roles, states, and key bindings. Don't invent your own.

## Keyboard interaction (WCAG 2.1.1 — everything operable without a mouse)

| Widget | Keys |
|--------|------|
| Button | Enter / Space activate |
| Link | Enter activates |
| Dialog (modal) | Esc closes; Tab cycles **within**; focus trapped |
| Menu / menubar | Arrow keys move; Esc closes; Enter selects |
| Tabs | Arrow keys switch; Tab moves to panel (roving tabindex) |
| Combobox / listbox | Arrows navigate options; Enter selects; Esc closes; type-ahead |
| Accordion | Enter/Space toggles header |

Custom widgets that take focus must handle these keys themselves. Never trap
keyboard focus except inside an intentional modal.

## Focus management

The hardest part of dynamic UIs. Three patterns:

```ts
// 1. Move focus INTO a newly opened dialog (first focusable or the dialog itself)
dialogEl.querySelector<HTMLElement>('[autofocus], button, [href], input')?.focus();

// 2. Trap focus while the modal is open (Tab from last → first, Shift+Tab first → last)
//    Native <dialog>.showModal() does this for you — prefer it.

// 3. RESTORE focus to the trigger when the dialog closes
const trigger = document.activeElement as HTMLElement;
// …on close:
trigger.focus();
```

- **Native `<dialog>` with `showModal()`** handles trapping, Esc, and the top layer
  — prefer it over hand-rolled modals.
- **Roving tabindex** for composite widgets (toolbars, menus, tabs): exactly one
  child is `tabindex="0"`, the rest `tabindex="-1"`; arrow keys move the `0`.
- **Restore focus** to the invoking element on close — never strand focus on the
  document body. (WCAG 2.4.3 Focus Order, 3.2.1 On Focus.)
- **Never `tabindex` > 0** — it breaks natural order. Use `0` (in order) or `-1`
  (programmatic only).
- **Visible focus indicator** is required (WCAG 2.4.7 / 2.4.13) — see
  [responsive-accessible-css](${CLAUDE_SKILL_DIR}/styling/responsive-accessible-css/SKILL.md).

## Live regions: announce dynamic changes

```html
<div aria-live="polite" aria-atomic="true">3 results found</div>     <!-- status updates -->
<div role="alert">Could not save — try again</div>                   <!-- assertive errors -->
```

`aria-live="polite"` waits for a pause; `role="alert"` (assertive) interrupts. Use
polite for status (search counts, autosave), assertive only for errors needing
immediate attention. The live region must exist in the DOM **before** you update it.

## Auditing with axe-core

```
npx @axe-core/cli http://localhost:5173        # CLI scan of a running route
```

```ts
import { axe } from 'vitest-axe';              // or jest-axe
expect(await axe(container)).toHaveNoViolations();   // component-level a11y assertion in tests
```

> axe catches ~30–50% of issues automatically — **it does not replace keyboard and
> screen-reader testing**. Tab through every flow; verify with a real screen reader
> (VoiceOver/NVDA) for the rest. Canonical floors: _shared/accessibility-baseline.md

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| `<div onClick>` as a button | `<button>` (free keyboard + role) |
| `role` added without its required states/keys | follow the ARIA APG pattern fully, or use native |
| Focus stranded after closing a modal | restore focus to the trigger |
| `tabindex="5"` to reorder | DOM order + `0`/`-1` only |
| Icon-only button with no name | `aria-label` |
| Updating content with no live region | `aria-live`/`role="alert"` for async changes |
| Treating an axe pass as "accessible" | also do keyboard + screen-reader testing |

## Related Skills

- [accessibility-baseline](${CLAUDE_SKILL_DIR}/_shared/accessibility-baseline.md) — canonical WCAG 2.2 success criteria
- [responsive-accessible-css](${CLAUDE_SKILL_DIR}/styling/responsive-accessible-css/SKILL.md) — contrast, focus visibility, reduced motion (CSS side)
- [quality-skills/SKILL.md](../quality-skills/SKILL.md) — canonical domain selection table
- [fe-testing/SKILL.md](../fe-testing/SKILL.md) — axe-in-tests, keyboard assertions
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — native `<dialog>` / `inert` support notes
