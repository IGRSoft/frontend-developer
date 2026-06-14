---
name: accessibility-baseline
description: WCAG 2.2 success-criteria baseline for front-end work — perceivable/operable/understandable/robust, keyboard and focus order, color contrast, target size, and the definition of the accessibility gate that QA enforces. Use when implementing accessible UI, auditing a component, or defining the a11y completion gate.
---

# Accessibility Baseline (WCAG 2.2)

**The non-negotiable accessibility floor every UI change clears. An axe-detectable
or audit-confirmed Level A/AA failure is a P1 finding (P0 when it blocks the core
task) — the a11y gate is a QA completion-blocker.** Used by
`fe-accessibility-auditor`, the framework agents, and the QA gate.

## Conformance Target

The baseline is **WCAG 2.2 Level AA**. Level A is the absolute minimum (failures
block core operability); Level AA is the gate target; Level AAA criteria are
best-practice (P2–P3). WCAG 2.2 adds nine criteria over 2.1 — the new ones most
likely to surface in review are called out below.

## The Four Principles (POUR)

WCAG organizes every success criterion under four principles. The baseline picks
the Level A/AA criteria that recur in front-end review.

### Perceivable — information must be presentable in ways users can perceive

| Criterion | Level | What it means in code |
|-----------|-------|-----------------------|
| 1.1.1 Non-text Content | A | Every `<img>` has an `alt` (empty `alt=""` for decorative); icons conveying meaning have an accessible name |
| 1.3.1 Info & Relationships | A | Use semantic elements (`<button>`, `<nav>`, `<h1-6>`, `<label>`, `<table>`); structure conveyed visually is conveyed in markup |
| 1.3.5 Identify Input Purpose | AA | `autocomplete` attributes on inputs collecting user info (name, email, address) |
| 1.4.3 Contrast (Minimum) | AA | Text contrast ≥ 4.5:1 (normal), ≥ 3:1 (large/18.66px-bold or 24px); UI/graphical contrast ≥ 3:1 (1.4.11) |
| 1.4.4 Resize Text | AA | Layout survives 200% zoom without loss of content/function; use relative units |
| 1.4.10 Reflow | AA | No horizontal scroll at 320px-equivalent width; responsive, no fixed pixel traps |
| 1.4.11 Non-text Contrast | AA | Interactive component boundaries / focus indicators / meaningful graphics ≥ 3:1 |

### Operable — UI components must be operable

| Criterion | Level | What it means in code |
|-----------|-------|-----------------------|
| 2.1.1 Keyboard | A | Every interactive control reachable and operable by keyboard alone; no mouse-only handlers |
| 2.1.2 No Keyboard Trap | A | Focus can always move away (modals trap *intentionally* but `Esc`/close restores focus) |
| 2.4.3 Focus Order | A | Tab order follows a logical reading sequence; no positive `tabindex` reordering |
| 2.4.7 Focus Visible | AA | A clearly visible focus indicator — never `outline: none` without a replacement |
| 2.4.11 Focus Not Obscured (Minimum) | AA | **WCAG 2.2** — sticky headers/footers must not hide the focused element |
| 2.5.7 Dragging Movements | AA | **WCAG 2.2** — any drag operation has a single-pointer (tap/click) alternative |
| 2.5.8 Target Size (Minimum) | AA | **WCAG 2.2** — interactive targets ≥ 24×24 CSS px (or sufficient spacing) |

### Understandable — information and operation must be understandable

| Criterion | Level | What it means in code |
|-----------|-------|-----------------------|
| 3.2.1 / 3.2.2 On Focus / On Input | A | Focusing or changing a control does not trigger an unexpected context change (no auto-submit on select) |
| 3.3.1 Error Identification | A | Validation errors are described in text, programmatically associated (`aria-describedby`), and announced |
| 3.3.2 Labels or Instructions | A | Every input has a visible, programmatically-associated `<label>` (or `aria-label`/`aria-labelledby`) |
| 3.3.7 Redundant Entry | A | **WCAG 2.2** — don't ask for the same info twice in a process; allow autofill/carry-forward |
| 3.3.8 Accessible Authentication (Minimum) | AA | **WCAG 2.2** — no cognitive-function test (e.g. transcribe-this puzzle) without an alternative; allow paste/password managers |

### Robust — content must be robust enough for assistive tech

| Criterion | Level | What it means in code |
|-----------|-------|-----------------------|
| 4.1.2 Name, Role, Value | A | Custom widgets expose the correct ARIA role, accessible name, and state (`aria-expanded`, `aria-checked`, `aria-selected`); native elements preferred over re-implemented ones |
| 4.1.3 Status Messages | AA | Async status (toasts, "3 results", validation) announced via `role="status"`/`aria-live` without moving focus |

## Keyboard & Focus Management

The most common real-world failures are keyboard and focus issues, which axe
cannot fully catch — they require manual/audit verification:

- **Everything operable by keyboard.** Tab to reach, Enter/Space to activate, arrow keys for composite widgets (menus, tabs, listboxes per the ARIA Authoring Practices). A `<div onClick>` is not keyboard-operable — use `<button>`.
- **Visible focus.** Never remove the focus ring without replacing it with an equally visible custom indicator (2.4.7, 2.4.11). `:focus-visible` is the modern tool.
- **Logical focus order** that matches the visual/reading order (2.4.3); no positive `tabindex`.
- **Manage focus on route/state change.** On navigation, move focus to the new view's heading or main landmark. On opening a modal, move focus in and trap it; on close, **restore focus to the trigger**.
- **Live regions** (`aria-live="polite"`/`role="status"`) announce async changes without stealing focus (4.1.3).
- **Skip link** to bypass repeated navigation to `<main>`.

## Color & Contrast

- Text contrast ≥ **4.5:1** (normal), ≥ **3:1** (large text ≥ 24px or ≥ 18.66px bold) — 1.4.3.
- Non-text contrast (control borders, focus indicators, icons, chart strokes) ≥ **3:1** — 1.4.11.
- **Never convey meaning by color alone** (1.4.1) — pair color with text, icon, or pattern (error states, required fields, chart series).
- Verify against the *actual rendered* foreground/background, including overlays, gradients, and `:hover`/`:disabled` states.

## Semantic HTML First

The cheapest accessibility is a native element: `<button>`, `<a href>`, `<input>`,
`<select>`, `<nav>`, `<main>`, `<dialog>`, `<details>` ship roles, states, keyboard
behavior, and focus for free. **Reach for ARIA only to fill a gap a native element
cannot** — "No ARIA is better than bad ARIA." A re-implemented `<div role="button"
tabindex="0">` with hand-wired keyboard handling is a frequent source of P1 bugs;
use `<button>`.

## The Accessibility Gate (definition)

A change clears the a11y gate when **all** hold:

1. **axe-core clean** on the changed components and affected pages — zero violations (`vitest-axe`/`jest-axe` in the component layer, `@axe-core/playwright` in e2e). Automated axe catches roughly a third of issues — necessary, not sufficient.
2. **Keyboard pass** — every new/changed interactive element is reachable and operable by keyboard, with visible focus, in logical order, no trap (manual/audit verification, since axe can't confirm this).
3. **Contrast pass** — text and non-text contrast meet 1.4.3 / 1.4.11 against rendered colors and states.
4. **Names & roles** — custom widgets expose correct role/name/state (4.1.2); every input has an associated label (3.3.2).
5. **No new WCAG 2.2 Level A/AA failure** introduced (focus-obscured 2.4.11, target-size 2.5.8, dragging 2.5.7, redundant-entry 3.3.7, accessible-auth 3.3.8).

A Level A/AA failure is **P1** (P0 if it blocks completing the core task). The gate
is enforced at QA; failures route to `frontend-developer:fe-code-fixer`. The
`fe-accessibility-auditor` runs this gate review-only and returns a ≤500-token
P0–P3 summary with `file:line`.

## Reference: WCAG 2.2 Additions

The nine criteria added in WCAG 2.2 (over 2.1) most relevant to front-end review:
2.4.11 Focus Not Obscured (Minimum, AA), 2.4.13 Focus Appearance (AAA),
2.5.7 Dragging Movements (AA), 2.5.8 Target Size Minimum (AA),
3.2.6 Consistent Help (A), 3.3.7 Redundant Entry (A),
3.3.8 Accessible Authentication Minimum (AA), 3.3.9 Accessible Authentication
Enhanced (AAA). (4.1.1 Parsing was removed in 2.2.)

> Requires the WCAG 2.2 criteria (current W3C Recommendation, Oct 2023). Fallback: WCAG 2.1 AA where a 2.2 criterion's tooling support is immature — still meet 2.1 AA as the floor. Canonical: _shared/version-feature-matrix.md

## Related Skills

- `severity-matrix.md` — how WCAG levels map to P0–P3 priorities
- `testing-principles.md` — the axe assertions and Testing-Library accessible-query doctrine
- `secure-coding/SKILL.md` — the parallel security gate every UI change also clears
- `workflow-integration/SKILL.md` — the QA gate definition (axe-clean + Lighthouse budget)
