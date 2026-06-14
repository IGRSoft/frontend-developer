---
name: sveltekit
description: >-
  SvelteKit application architecture: file-based routing and layouts, `load`
  functions (server `+page.server.ts` vs universal `+page.ts`), form actions
  with progressive enhancement (`use:enhance`), the server/universal code split,
  and safe environment-variable modules (`$env/static|dynamic/private|public`).
  Use when building SvelteKit routes, loading page data, handling form
  mutations, or separating server-only code and secrets from the client.
---

# SvelteKit

**Routing, data loading, and form actions for Svelte apps.** The canonical
selection table lives in [svelte-skills/SKILL.md](../svelte-skills/SKILL.md);
this leaf carries depth only. Component reactivity lives in
[svelte-runes](../svelte-runes/SKILL.md).

## When to Use

- Structuring routes, layouts, and nested data
- Loading page data on the server vs universally
- Handling a form mutation with progressive enhancement
- Keeping secrets and server-only logic off the client
- Choosing a rendering mode (SSR / prerender / CSR) per route

## File-based routing

| File | Role |
|------|------|
| `+page.svelte` | the page component (uses runes) |
| `+page.ts` | **universal** `load` — runs on server then client |
| `+page.server.ts` | **server-only** `load` + form `actions` (DB, secrets) |
| `+layout.svelte` / `+layout(.server).ts` | shared shell + data for child routes |
| `+server.ts` | a route handler (REST/JSON endpoint) |
| `+error.svelte` | error boundary for the subtree |

Routes are directories; `[param]` is dynamic, `[...rest]` is a catch-all,
`(group)` groups without affecting the URL.

## `load`: server vs universal

```ts
// +page.server.ts — runs only on the server; safe for DB and secrets
import type { PageServerLoad } from "./$types";
export const load: PageServerLoad = async ({ params, locals }) => {
  const post = await db.post.find(params.slug);   // never reaches the client bundle
  return { post };
};
```

```ts
// +page.ts — runs on server (SSR) and client (navigation); only public data/fetch
import type { PageLoad } from "./$types";
export const load: PageLoad = async ({ fetch, params }) => {
  const res = await fetch(`/api/posts/${params.slug}`); // use the provided fetch
  return { post: await res.json() };
};
```

**Rule:** anything touching a database, a secret, or server-only modules goes in
`+page.server.ts`. Use the `fetch` passed into `load` (it forwards cookies and
dedupes during SSR), not the global `fetch`. Typed via the generated `./$types`.

## Form actions + progressive enhancement

```ts
// +page.server.ts
import { fail } from "@sveltejs/kit";
import type { Actions } from "./$types";
export const actions: Actions = {
  create: async ({ request }) => {
    const data = await request.formData();
    const title = String(data.get("title") ?? "");
    if (!title) return fail(400, { error: "Title required" });   // validate on the server
    await db.post.create({ title });
    return { success: true };
  },
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import { enhance } from "$app/forms";
  let { form } = $props();        // action result
</script>
<form method="POST" action="?/create" use:enhance>
  <input name="title" aria-invalid={!!form?.error} />
  {#if form?.error}<p role="alert">{form.error}</p>{/if}
  <button>Create</button>
</form>
```

Forms work **without JS** (native POST); `use:enhance` upgrades them to no-reload
AJAX. Always validate action input on the server — it is a public endpoint (see
[secure-coding](../../_shared/secure-coding/SKILL.md)).

## Environment variables — keep secrets server-side

| Module | Contents | Where usable |
|--------|----------|--------------|
| `$env/static/private` | build-time secrets (`DATABASE_URL`) | **server only** |
| `$env/dynamic/private` | runtime secrets | **server only** |
| `$env/static/public` | build-time public (`PUBLIC_*`) | server + client |
| `$env/dynamic/public` | runtime public (`PUBLIC_*`) | server + client |

> Requires SvelteKit `$env` modules. Fallback: `process.env` in server hooks on older Kit. Canonical: _shared/version-feature-matrix.md

Importing a `private` env module into client-reachable code is a **build error** —
that guard is the point. Only `PUBLIC_`-prefixed vars are exposed to the browser.

## Rendering mode per route

```ts
export const prerender = true;   // static HTML at build (marketing pages)
export const ssr = false;        // client-only render (dashboard behind auth)
export const csr = true;         // ship client JS (default)
```

Default is SSR + hydration. `prerender` for static content, `ssr = false` for
purely client routes. Choose per route based on data freshness and SEO needs.

## Diagnostics

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Cannot import `$env/static/private`" in a component | secret module reached the client | move the read into `+page.server.ts`/`+server.ts` |
| `load` data is stale after a mutation | no invalidation | `invalidate()`/`invalidateAll()` or return fresh data from the action |
| Form posts reload the whole page | missing `use:enhance` | add `use:enhance` (and keep the no-JS fallback working) |
| Global `fetch` loses cookies during SSR | not using the provided `fetch` | use the `fetch` from `load`'s event |
| Hydration mismatch | server/client `load` diverged or browser API in universal `load` | move browser-only reads to an effect or server load |

## Related Skills

- [svelte-runes](../svelte-runes/SKILL.md) — component reactivity consuming `load` data
- [secure-coding](../../_shared/secure-coding/SKILL.md) — action validation, env-secret boundaries, `{@html}` XSS
- [web-performance](../../quality/web-performance/SKILL.md) — SSR/prerender choices and Core Web Vitals
- [svelte-skills](../svelte-skills/SKILL.md) — canonical selection table (start here)
- [version-feature-matrix](../../_shared/version-feature-matrix.md) — canonical SvelteKit version minimums
