---
name: angular-rxjs
description: >-
  RxJS interop and subscription discipline for Angular 18+ — toSignal()/
  toObservable() bridges, the async pipe, takeUntilDestroyed() lifecycle, and
  choosing streams vs signals. Use when wiring HTTP/event streams, bridging RxJS
  to signals, fixing subscription leaks, or deciding which reactivity primitive
  models async data over time.
---

# Angular RxJS Interop (18+)

**Stream patterns and the signal bridge.** For the domain selection table and
version snapshot, see the canonical [angular-skills/SKILL.md](../angular-skills/SKILL.md) —
this leaf carries depth and does not duplicate it.

## When to Use

Use this skill when:
- The data is an **async stream over time** — HTTP responses, DOM/router events,
  websockets, debounced input — where operators (`debounceTime`, `switchMap`,
  `combineLatest`) earn their keep
- You need to bridge an observable into a signal (or vice versa) for the template
- You are fixing a subscription leak or unsubscribe bug
- You are deciding between a `signal` (synchronous latest value) and a stream

## Streams vs signals (the routing rule)

Signals model **the latest synchronous value**; observables model **values over
time with cancellation/operators**. Default to a signal for component/UI state;
reach for RxJS when the data is asynchronous and benefits from operators.

| Reach for RxJS | Reach for a signal |
|----------------|--------------------|
| HTTP with retry/cancel (`switchMap`, `retry`) | Local UI flags, form-derived values |
| Debounced search input (`debounceTime` + `distinctUntilChanged`) | A `computed` of other signals |
| Combining several async sources (`combineLatest`, `forkJoin`) | The current value the template renders |
| Event buses, websockets, backpressure | Anything you would otherwise `BehaviorSubject` just to read `.value` |

See [angular-signals](../angular-signals/SKILL.md) for the signal side.

## toSignal / toObservable bridge

```ts
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { switchMap, debounceTime, distinctUntilChanged } from 'rxjs';

@Component({ standalone: true /* ... */ })
export class SearchComponent {
  private http = inject(HttpClient);

  query = signal('');                                   // synchronous input value

  // signal -> observable -> async pipeline -> signal
  results = toSignal(
    toObservable(this.query).pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(q => this.http.get<Result[]>(`/api/search?q=${encodeURIComponent(q)}`)),
    ),
    { initialValue: [] as Result[] },
  );
}
```

> Requires `toSignal`/`toObservable` from `@angular/core/rxjs-interop` (Angular 16+). Fallback: subscribe in the component with manual teardown and assign to a field. Canonical: _shared/version-feature-matrix.md

**`toSignal` auto-unsubscribes** when the injection context is destroyed — no manual
teardown. Always give it an `initialValue` (or it returns `undefined` until the
first emission). Call it in an injection context (field initializer / `inject()` scope).

## Subscription lifecycle: never leak

Prefer the **`async` pipe** in the template (it subscribes and unsubscribes for
you) or **`toSignal`**. When you must subscribe imperatively, tie teardown to the
component lifecycle:

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

constructor() {
  this.events$.pipe(takeUntilDestroyed()).subscribe(e => this.handle(e)); // auto-teardown
}
```

> Requires `takeUntilDestroyed()` (Angular 16+). Fallback: a `Subject` + `takeUntil(this.destroy$)` completed in `ngOnDestroy`. Canonical: _shared/version-feature-matrix.md

Outside the constructor injection context, pass an explicit `DestroyRef`:
`takeUntilDestroyed(this.destroyRef)`.

## Flattening operators: pick the right one

| Operator | Use when a new source emits and… |
|----------|----------------------------------|
| `switchMap` | …you want to **cancel** the previous inner request (typeahead, latest-wins) |
| `concatMap` | …order matters; queue inner requests sequentially |
| `mergeMap` | …run concurrently, order does not matter (independent writes) |
| `exhaustMap` | …ignore new triggers while one is in flight (submit-button guard) |

Default to **`switchMap` for reads** (cancel stale) and **`exhaustMap`/`concatMap`
for writes** (avoid duplicate/ordered side effects).

## HTTP with functional providers

```ts
provideHttpClient(withInterceptors([authInterceptor]));   // standalone bootstrap

const data$ = this.http.get<Dto[]>('/api/items').pipe(
  retry({ count: 2, delay: 1000 }),
  catchError(err => { this.log(err); return of([]); }),   // degrade, don't crash the stream
);
```

> Requires `provideHttpClient()` functional providers (Angular 15+). Fallback: import `HttpClientModule`. Canonical: _shared/version-feature-matrix.md

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| `subscribe()` with no teardown | `async` pipe, `toSignal`, or `takeUntilDestroyed()` |
| Nested `subscribe()` inside `subscribe()` | flatten with `switchMap`/`mergeMap` |
| `mergeMap` for a typeahead (stale results win) | `switchMap` (cancel previous) |
| `BehaviorSubject` used only to read `.value` | a `signal` |
| Manual `ngOnDestroy` + `Subject` boilerplate | `takeUntilDestroyed()` |
| Error from one source kills the whole stream | `catchError` returning a fallback `of(...)` |

## Related Skills

- [angular-signals/SKILL.md](../angular-signals/SKILL.md) — the synchronous side; when to prefer signals
- [angular-skills/SKILL.md](../angular-skills/SKILL.md) — canonical domain selection table
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — canonical Angular minimums
- [secure-coding](${CLAUDE_SKILL_DIR}/_shared/secure-coding/SKILL.md) — encode query params; avoid `bypassSecurityTrust*`
