---
name: modern-react
description: >-
  Modern React 19 + Next.js 15 features with explicit version gates: React
  Server Components, Server Actions (`"use server"`), the `use(promise/context)`
  hook, `useActionState`/`useFormStatus`/`useOptimistic`, `ref` as a prop,
  document metadata, and the React Compiler. Use when adopting a React 19
  feature, deciding the Server/Client boundary, wiring a form mutation, finding
  a React 18 fallback, or fixing Rules-of-Hooks violations.
---

# Modern React (19 + Next.js 15)

**Version-gated feature adoption with explicit fallbacks.** The canonical
selection table lives in [react-skills/SKILL.md](../react-skills/SKILL.md); this
leaf carries depth only.

## When to Use

- Adopting a React 19 feature and you need its minimum version + fallback
- Deciding whether a component is a Server or Client Component
- Replacing a client `fetch`/`useEffect` data flow with RSC or `use`
- Wiring a form mutation with a Server Action
- Removing manual memoization once the React Compiler is on
- Fixing a Rules-of-Hooks / `exhaustive-deps` violation

## Per-Feature Version Gate

Canonical minimums live in the
[version-feature-matrix](../../_shared/version-feature-matrix.md).

| Feature | Min version | Pre-version fallback |
|---------|-------------|----------------------|
| React Server Components | React 19 + framework (Next 15 App Router) | client components + route loaders; fetch in a loader |
| Server Actions (`"use server"`, `<form action={fn}>`) | React 19 (+ Next 15) | API route + client `fetch` |
| `use(promise)` / `use(context)` | React 19 | `useContext`; thread promises via a data lib |
| `useActionState` / `useFormStatus` | React 19 | `useState` + manual pending/error tracking |
| `useOptimistic` | React 19 | local optimistic `useState`, reconcile on response |
| `ref` as a prop (no `forwardRef`) | React 19 | `forwardRef(...)` |
| Document metadata (`<title>`/`<meta>` in JSX) | React 19 | `next/head` / `react-helmet` |
| React Compiler (auto-memoization) | React Compiler 1.0 (stable, opt-in) | manual `useMemo`/`useCallback`/`React.memo` |
| Concurrent (`useTransition`/`useDeferredValue`/`<Suspense>`) | React 18+ | synchronous renders; manual debounce |

> Requires the `use` hook and Server Actions (React 19). Fallback: `useContext` + API routes on React 18. Canonical: _shared/version-feature-matrix.md

## Server vs Client Components

In the App Router every component is a **Server Component** unless the file (or an
ancestor it imports) starts with `"use client"`.

```tsx
// app/users/page.tsx — Server Component (no "use client")
import { db } from "@/lib/db";
export default async function UsersPage() {
  const users = await db.user.findMany();   // runs on the server, no client JS shipped
  return <UserList users={users} />;
}

// components/SearchBox.tsx — needs state/handlers → Client Component
"use client";
import { useState } from "react";
export function SearchBox() {
  const [q, setQ] = useState("");
  return <input value={q} onChange={(e) => setQ(e.target.value)} />;
}
```

**Rules:** keep `"use client"` as low in the tree as possible (push interactivity
to leaves). Only **serializable** props cross the boundary — no functions
(except Server Actions), class instances, or Dates-as-objects. Data fetching
belongs in Server Components or a server cache, not `useEffect`.

## Server Actions + `useActionState`

```tsx
// app/actions.ts
"use server";
export async function createTodo(_prev: State, formData: FormData): Promise<State> {
  const title = String(formData.get("title") ?? "");
  if (!title) return { error: "Title required" };
  await db.todo.create({ data: { title } });
  return { ok: true };
}

// components/TodoForm.tsx
"use client";
import { useActionState } from "react";
import { createTodo } from "@/app/actions";

export function TodoForm() {
  const [state, action, pending] = useActionState(createTodo, {});
  return (
    <form action={action}>
      <input name="title" aria-invalid={!!state.error} />
      {state.error && <p role="alert">{state.error}</p>}
      <button disabled={pending}>{pending ? "Saving…" : "Add"}</button>
    </form>
  );
}
```

Pre-React-19 fallback: an API route + client `fetch`, with `useState` tracking
pending/error. Validate every Server Action input on the server — it is a public
endpoint (see [secure-coding](../../_shared/secure-coding/SKILL.md)).

## The `use` hook

`use(promise)` suspends until the promise resolves; `use(context)` reads context
and (unlike `useContext`) may be called conditionally.

```tsx
function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise);   // suspends; wrap in <Suspense>
  return <ul>{comments.map((c) => <li key={c.id}>{c.body}</li>)}</ul>;
}
```

Create the promise in a Server Component and pass it down; render `<Comments>`
inside `<Suspense fallback={...}>`. Pre-19 fallback: a data library (TanStack
Query) or manual state.

## React Compiler: stop hand-memoizing

With the React Compiler enabled (Babel/SWC plugin), the compiler inserts
memoization automatically — remove most manual `useMemo`/`useCallback`/`memo`.

> Requires the React Compiler (1.0 stable, opt-in; React 17+, best on React 19). Fallback: keep manual `useMemo`/`useCallback`/`React.memo`. Canonical: _shared/version-feature-matrix.md

**Rule:** don't add manual memoization "just in case" in a Compiler project; keep
it only for measured hot paths the compiler can't reach (see
[react-performance](../react-performance/SKILL.md)). Code must still obey the
Rules of Hooks for the compiler to bail in safely.

## Rules of Hooks (always)

- Call hooks at the **top level** of a component or custom hook — never inside
  conditions, loops, or nested functions. (`use` is the only conditional-capable
  exception.)
- Custom hooks start with `use`.
- `exhaustive-deps` must be clean: list every reactive value the effect reads, or
  restructure so it doesn't depend on it. Don't silence with a disable comment.
- Effects are for **synchronizing with external systems**, not for deriving state
  — derive during render or with `useMemo`.

## Diagnostics

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Functions cannot be passed directly to Client Components" | non-serializable prop across RSC boundary | pass a Server Action, or move the boundary |
| `useState`/`useEffect` is not defined in a server file | missing `"use client"` | add the directive (push it to a leaf) |
| Hydration mismatch | server/client render differ (Date.now, random, `window`) | gate browser-only reads behind an effect or `useId` |
| Server Action returns but UI doesn't update | not using the action's state, or no revalidation | use `useActionState` result; `revalidatePath`/`revalidateTag` |
| `forwardRef` deprecation warning | React 19 accepts `ref` as a prop | type `ref` in props, drop `forwardRef` |

## Related Skills

- [react-state](../react-state/SKILL.md) — where mutated/fetched data lives
- [react-performance](../react-performance/SKILL.md) — Suspense streaming, virtualization, measured memoization
- [secure-coding](../../_shared/secure-coding/SKILL.md) — Server Action input validation, `dangerouslySetInnerHTML`
- [react-skills](../react-skills/SKILL.md) — canonical selection table (start here)
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical React / Next version minimums
