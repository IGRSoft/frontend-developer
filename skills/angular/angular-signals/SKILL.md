---
name: angular-signals
description: >-
  Angular 18+ signals reactivity with explicit version gates — signal(),
  computed(), effect(), signal-based input()/output()/model(), standalone
  components, and the new @if/@for/@switch control flow. Use when building or
  reviewing reactive Angular state, migrating @Input/@Output to signals, or
  choosing signals over RxJS for synchronous UI state.
---

# Angular Signals (18+)

**Version-gated reactivity and standalone-component patterns.** For the domain
selection table and version snapshot, see the canonical
[angular-skills/SKILL.md](../angular-skills/SKILL.md) — this leaf carries depth and
does not duplicate it.

## When to Use

Use this skill when:
- Building synchronous UI state that should re-render on change (`signal`/`computed`)
- Migrating `@Input()`/`@Output()` decorators to signal `input()`/`output()`/`model()`
- Deciding between a `signal` and an RxJS stream (signals = synchronous values; streams = async — see [angular-rxjs](../angular-rxjs/SKILL.md))
- Authoring a new standalone component with the v17+ control flow
- Reviewing for change-detection correctness (mutation vs new reference)

## Per-Feature Version Gate

Pick the lowest version that has the feature; if the project is pinned lower, use
the fallback column. Canonical minimums live in
[version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md).

| Feature | Min version | Pre-version fallback |
|---------|-------------|----------------------|
| `signal()` / `computed()` / `effect()` | Angular 16 (preview), stable 17+ | RxJS `BehaviorSubject` + `async` pipe |
| Signal `input()` / `input.required()` | Angular 17.1+ | `@Input()` decorator + setter |
| Signal `output()` | Angular 17.3+ | `@Output()` + `EventEmitter` |
| Two-way `model()` | Angular 17.2+ | `@Input()` + `@Output('xChange')` pair |
| Signal queries (`viewChild`/`contentChild` as signals) | Angular 17.2+ | `@ViewChild`/`@ContentChild` decorators |
| New control flow (`@if`/`@for`/`@switch`) | Angular 17+ | `*ngIf`/`*ngFor`/`*ngSwitch` |
| `@defer` deferred loading | Angular 17+ | `loadChildren` / manual lazy import |
| Standalone components default | Angular 15+ (default since 19) | declare in an `NgModule` |
| Signal-based forms | Angular 22 stable (experimental 21) *(verify — newly stabilized)* | reactive forms (`FormGroup`/`FormControl`) |
| Zoneless change detection | Angular 20.2 stable; default in new apps since 21 | Zone.js (default) |

The primitives are **stable in Angular 20**; the versions above are historical floors.

> Requires Angular 20+ stable signals and standalone components (zoneless stable 20.2, default since 21). Fallback: Angular 15 RxJS `BehaviorSubject` + `async` pipe and NgModule declarations. Canonical: _shared/version-feature-matrix.md

## signal / computed / effect

`signal` holds a value; `computed` derives one lazily and memoizes; `effect` runs
a side effect when read signals change. Read a signal by calling it (`count()`).

```ts
import { signal, computed, effect } from '@angular/core';

const count = signal(0);
const doubled = computed(() => count() * 2);   // lazy + memoized; recomputes only when count changes

effect(() => console.log('count is', count())); // runs on change; auto-tracks read signals

count.set(5);                 // replace
count.update(n => n + 1);     // derive from previous
```

**Rules:**
- **Never mutate a signal's value in place** for objects/arrays — call `.set()`/`.update()`
  with a *new reference* so change detection and `computed` see the change.
- **`computed` is for derivation, not side effects;** `effect` is for side effects
  (logging, DOM sync, persisting), not for writing other signals (allowed only via
  `allowSignalWrites`, which usually signals a design smell — prefer `computed`).
- **`effect` cleans itself up** when its injection context is destroyed; do not
  hand-roll teardown.

## Signal-based component I/O (17.1+)

```ts
import { Component, input, output, model } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (label()) { <span>{{ label() }}</span> }
    <button (click)="inc()">{{ value() }}</button>
  `,
})
export class CounterComponent {
  label = input('');                      // optional input with default
  step  = input.required<number>();       // required input — compile error if omitted
  value = model(0);                        // two-way bindable: [(value)]
  changed = output<number>();             // typed event emitter

  inc() {
    this.value.update(v => v + this.step());
    this.changed.emit(this.value());
  }
}
```

> Requires signal `input()`/`output()`/`model()` (Angular 17.1+). Fallback: `@Input()`/`@Output()` decorators with `EventEmitter`. Canonical: _shared/version-feature-matrix.md

`input()` is **read-only** inside the component — Angular updates it from the
parent binding; you never assign to it. Use `model()` when the value flows both ways.

## New control flow (17+)

```html
@if (user(); as u) {
  <p>Welcome {{ u.name }}</p>
} @else {
  <p>Please sign in</p>
}

@for (item of items(); track item.id) {     <!-- track is REQUIRED -->
  <li>{{ item.label }}</li>
} @empty {
  <li>No items</li>
}

@switch (status()) {
  @case ('loading') { <spinner /> }
  @case ('error')   { <error-banner /> }
  @default          { <content /> }
}
```

`@for` **requires** a `track` expression — use a stable identity (`item.id`), not
`$index`, so the DOM diff is correct on reorder. This is the largest perf win over
`*ngFor` without `trackBy`.

> Requires new control flow (Angular 17+). Fallback: `*ngIf`/`*ngFor` with `trackBy`/`*ngSwitch`. Canonical: _shared/version-feature-matrix.md

## Deferred loading (@defer, 17+)

```html
@defer (on viewport) {
  <heavy-chart [data]="data()" />
} @placeholder { <skeleton /> } @loading (minimum 200ms) { <spinner /> }
```

`@defer` lazy-loads the block and its component dependencies, splitting them into a
separate chunk — a declarative alternative to route-level lazy loading for
below-the-fold widgets. See [bundling-optimization](${CLAUDE_SKILL_DIR}/tooling/bundling-optimization/SKILL.md).

## Standalone bootstrap + functional providers

```ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
  ],
});
```

No `AppModule`, no `NgModule`. Import dependencies directly in the component's
`imports: [...]`. For DI in functions, use `inject()` rather than constructor
parameters where it reads more clearly.

## Signals vs RxJS (the routing question)

| Use a signal when… | Use RxJS when… |
|--------------------|----------------|
| The value is synchronous and you just need the latest | The data is an async stream over time (HTTP, events, websockets) |
| You want template-driven reactivity with `computed` | You need operators: `debounceTime`, `switchMap`, `combineLatest`, retry |
| Local component/UI state | Cross-cutting event buses, cancellation, backpressure |

Bridge the two with `toSignal()`/`toObservable()` — see [angular-rxjs](../angular-rxjs/SKILL.md).

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| Mutating an object signal in place (`obj().x = 1`) | `obj.update(o => ({ ...o, x: 1 }))` |
| Writing signals inside an `effect` | Derive with `computed` instead |
| `@for` without `track` | Add `track item.id` (required in v17+) |
| `@Input()` decorator in new v17+ code | `input()` / `input.required()` |
| Keeping `NgModule` for a new feature | Standalone component + functional providers |

## Related Skills

- [angular-rxjs/SKILL.md](../angular-rxjs/SKILL.md) — when to stay on streams; interop helpers
- [angular-skills/SKILL.md](../angular-skills/SKILL.md) — canonical domain selection table
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — canonical Angular minimums
- [web-performance](${CLAUDE_SKILL_DIR}/quality/web-performance/SKILL.md) — change-detection and render-budget impact
