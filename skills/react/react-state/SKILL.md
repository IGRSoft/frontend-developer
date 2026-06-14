---
name: react-state
description: >-
  React state-management architecture: separating server cache from client UI
  state, and choosing between Redux Toolkit, Zustand, and TanStack Query. Use
  when deciding where a piece of state lives, picking a state library, modeling
  async server data, lifting state, or untangling prop-drilling and Context
  overuse.
---

# React State Management

**Decide where state lives, then pick the lightest tool that fits.** The
canonical selection table lives in
[react-skills/SKILL.md](../react-skills/SKILL.md); this leaf carries depth only.

## When to Use

- Deciding where a piece of state should live (local / server / global)
- Choosing a state library for a new feature or app
- Modeling server data that needs caching, refetching, and mutation
- Replacing prop-drilling or an overgrown Context
- Migrating from a hand-rolled global store

## First question: is it server state or client state?

The biggest mistake is storing **server data** in a client store. They have
different lifecycles.

| Kind | Examples | Tool |
|------|----------|------|
| **Server state** (cache of remote data) | API responses, lists, the current user record | **TanStack Query** (or RSC fetch + `revalidate`) — handles caching, refetch, staleness, dedupe |
| **Client/UI state** (lives only in the browser) | modal open, selected tab, form draft, theme | `useState`/`useReducer` (local) or **Zustand**/**RTK** (shared) |
| **URL state** | filters, pagination, search query | the router's search params — single source of truth, shareable |
| **Form state** | field values, validation, submission | React 19 Server Actions + `useActionState`, or a form lib (React Hook Form) |

> Requires React 19 Server Actions for server-mutating forms. Fallback: a form library + API route on React 18. Canonical: _shared/version-feature-matrix.md

## Library selection

| Need | Reach for | Why |
|------|-----------|-----|
| Cache/refetch/mutate remote data | **TanStack Query** | purpose-built for async server state; query keys, stale-while-revalidate, optimistic updates |
| Small-to-mid shared client state | **Zustand** | tiny, hook-first, no provider, selector-based subscriptions; minimal boilerplate |
| Large, normalized, audited client state | **Redux Toolkit** | slices, RTK Query, devtools time-travel, entity adapters, middleware for complex flows |
| Purely local component state | `useState` / `useReducer` | no library; `useReducer` for multi-field state machines |
| A handful of values shared down one subtree | **Context** (read-mostly) | fine for theme/auth; **not** for high-frequency updates (re-renders all consumers) |

**Rule:** start local. Lift to a store only when state is genuinely shared across
distant components, and prefer Zustand unless RTK's normalization/devtools/
middleware are needed.

## TanStack Query: the server-state default

```tsx
const { data, isPending, error } = useQuery({
  queryKey: ["user", id],
  queryFn: () => fetchUser(id),
  staleTime: 30_000,
});

const qc = useQueryClient();
const mutation = useMutation({
  mutationFn: updateUser,
  onSuccess: () => qc.invalidateQueries({ queryKey: ["user", id] }),
});
```

Keep server data out of Zustand/Redux — let Query own the cache, staleness, and
refetch. In an RSC app, fetch on the server and use Query only for client-side
mutations/refetch.

## Zustand: light shared client state

```ts
import { create } from "zustand";

interface UiState { sidebarOpen: boolean; toggle: () => void; }
export const useUi = create<UiState>((set) => ({
  sidebarOpen: false,
  toggle: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}));

// subscribe to a slice only — avoids re-render on unrelated changes
const open = useUi((s) => s.sidebarOpen);
```

Select narrow slices; subscribing to the whole store re-renders on every change.

## Redux Toolkit: when the app demands it

Use RTK for large apps with normalized entities, complex async flows, or a team
that needs devtools time-travel. Use `createSlice` + RTK Query (Redux's
server-state layer) instead of hand-written thunks/reducers. Never hand-roll
action-type constants or immutable updates — RTK's Immer-backed reducers and
`createSlice` are the standard.

## Context: read-mostly only

`Context` is a dependency-injection mechanism, not a state manager. A Context
value that changes often re-renders **every** consumer. Use it for theme, auth,
locale, or a stable store reference; for frequently-updated shared state use
Zustand/RTK with selector subscriptions.

## Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| API data cached in Zustand/Redux | TanStack Query (or RSC + revalidate) |
| Global store for state used by one component | local `useState`/`useReducer` |
| Frequently-changing value in Context | Zustand/RTK with selector subscription |
| Filters/pagination duplicated in state + URL | URL search params as the single source of truth |
| Deriving state in `useEffect` (`setState` from props) | compute during render or `useMemo` |
| Subscribing to the whole Zustand store | select the specific slice you read |

## Related Skills

- [modern-react](../modern-react/SKILL.md) — RSC fetch + Server Actions reduce how much client state you need
- [react-performance](../react-performance/SKILL.md) — selector granularity and re-render cost
- [react-skills](../react-skills/SKILL.md) — canonical selection table (start here)
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical React / Next version minimums
