# CSP, Trusted Types, and Secrets (Front-End / Web)

Use this when:

- You author or review a Content-Security-Policy header or `<meta http-equiv>` policy.
- You wire per-request nonces or hashes so your own scripts load under a strict policy.
- You enable Trusted Types to force every DOM-sink write through a policy.
- You add Subresource Integrity (SRI) to a third-party `<script>`/`<link>`.
- You read an environment variable that might ship into the client bundle.
- A reviewer flagged a weak CSP, a missing nonce, or a secret-in-bundle leak.

Skip this file if:

- You are escaping data into the DOM or chasing an XSS sink. Use `xss-and-injection.md`.
- You only need the rule summary. Use the parent `SKILL.md`.

Jump to:

- CSP as Defense in Depth
- The Strict, Nonce-Based Policy
- Directive Reference
- Wiring a Nonce per Framework
- Trusted Types
- Subresource Integrity (SRI)
- Secrets and the Client Bundle
- Environment-Variable Prefix Rules
- Detecting a Bundle Leak
- Checklist

## CSP as Defense in Depth

A Content-Security-Policy does not stop XSS at the source — escaping and sanitization (see `xss-and-injection.md`) do that. CSP **contains** an XSS that slips past them: even if attacker markup reaches the DOM, a strict policy refuses to execute injected inline scripts, `eval`, and unexpected external loads. Treat CSP as the seatbelt, not the brakes — you need both.

The single most important property: a CSP is only as strong as its `script-src`. `script-src 'unsafe-inline'` (or no `script-src` at all) means any injected `<script>` runs — the policy provides essentially zero XSS containment. The goal is a policy with **no `'unsafe-inline'` and no `'unsafe-eval'`** in `script-src`.

## The Strict, Nonce-Based Policy

The recommended modern policy uses a per-request nonce plus `'strict-dynamic'`, so your own scripts run and everything else is blocked:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'nonce-{RANDOM}' 'strict-dynamic' https: 'unsafe-inline';
  object-src 'none';
  base-uri 'none';
  frame-ancestors 'self';
  require-trusted-types-for 'script';
```

How it reads:
- `'nonce-{RANDOM}'` — only `<script>` tags carrying this exact, unguessable, per-response nonce execute. An injected `<script>` has no nonce, so it is blocked.
- `'strict-dynamic'` — scripts your nonced script loads are trusted transitively, so you do not need to allow-list every CDN. It also makes browsers that understand it **ignore** the trailing `https:` and `'unsafe-inline'` (those are fallbacks only for older browsers, harmless on modern ones).
- `object-src 'none'`, `base-uri 'none'` — close the `<object>`/`<embed>` and `<base>`-hijack holes.
- `frame-ancestors 'self'` — anti-clickjacking (supersedes `X-Frame-Options`).

The nonce must be **cryptographically random and unique per response** (e.g. 16+ bytes, base64). Reusing a nonce across responses, or a guessable one, defeats the whole mechanism.

## Directive Reference

| Directive | Purpose | Strict value |
|-----------|---------|--------------|
| `default-src` | Fallback for unset fetch directives | `'self'` |
| `script-src` | Where scripts may load / inline-exec | `'nonce-…' 'strict-dynamic'` — no `'unsafe-inline'`/`'unsafe-eval'` |
| `style-src` | Stylesheets / inline styles | `'self'` (+ nonce/hash if inline styles are unavoidable) |
| `connect-src` | `fetch`/XHR/WebSocket/`EventSource` targets | allow-list your API origins |
| `img-src` / `font-src` / `media-src` | Asset origins | `'self'` + your CDN |
| `frame-ancestors` | Who may frame this page (anti-clickjacking) | `'self'` (or named origins) |
| `frame-src` / `child-src` | What this page may frame | allow-list embed origins |
| `object-src` | `<object>`/`<embed>`/`<applet>` | `'none'` |
| `base-uri` | `<base href>` (stops base-tag hijack) | `'none'` or `'self'` |
| `form-action` | Where forms may POST | `'self'` |
| `upgrade-insecure-requests` | Auto-rewrite `http:`→`https:` | present |
| `require-trusted-types-for` | Force DOM-sink writes through a TT policy | `'script'` |
| `report-to` / `report-uri` | Where violation reports go | a collector endpoint |

Roll out with `Content-Security-Policy-Report-Only` first: it reports violations without breaking the page, so you can tighten the policy against real traffic before enforcing.

## Wiring a Nonce per Framework

The nonce is generated per request on the server and threaded to both the CSP header and every first-party inline script.

| Framework | Mechanism |
|-----------|-----------|
| Next.js (App Router) | Generate the nonce in `middleware.ts`, set it on the request header and the CSP header; Next propagates it to its own scripts automatically. |
| Nuxt | `nuxt-security` module, or set the header in server middleware + `useHead({ script: [{ nonce }] })`. |
| SvelteKit | `svelte.config.js` `csp: { mode: 'auto' }` — SvelteKit generates nonces/hashes for its own scripts; add a `handle` hook for custom headers. |
| Angular | `ngCspNonce` attribute on the root element (Angular 16+) so the runtime tags its injected styles/scripts. |
| Express/Node SSR | `helmet` with a per-request `res.locals.nonce`; inject into templates. |

> Requires the `ngCspNonce` attribute for Angular CSP nonce propagation (Angular 16+). Fallback: hash-based `style-src`/`script-src` or `'unsafe-inline'` with a documented justification on pre-16 apps. Canonical: _shared/version-feature-matrix.md

Never solve a "third-party widget needs inline script" problem by adding `'unsafe-inline'` — nonce it, hash it, or load it as an external file. One `'unsafe-inline'` reopens the whole XSS-containment hole.

## Trusted Types

Trusted Types turn every dangerous DOM sink (`innerHTML`, `script.src`, `eval`, …) into a type error unless the value passed through a registered policy — eliminating DOM-XSS by construction rather than by review.

```ts
// Register a single policy that sanitizes everything assigned to a sink.
if (window.trustedTypes) {
  window.trustedTypes.createPolicy("default", {
    createHTML: (s) => DOMPurify.sanitize(s),
    createScriptURL: (s) => {
      const u = new URL(s, location.origin);
      if (u.origin !== location.origin) throw new TypeError("blocked");
      return u.href;
    },
  });
}
```

Enforce it with `require-trusted-types-for 'script'` in the CSP. After that, any raw string assigned to `innerHTML`/`script.src` throws — the only way to write a sink is through the policy, which sanitizes.

> Requires Trusted Types (`require-trusted-types-for`, `window.trustedTypes`) — Chromium-based browsers only; not implemented in Firefox or Safari at time of writing. Fallback: keep the sanitize-before-sink discipline and a strict nonce-based `script-src`; treat Trusted Types as a Chromium-only hardening backstop. Canonical: _shared/version-feature-matrix.md

## Subresource Integrity (SRI)

When you load a third-party script/stylesheet from a CDN, SRI pins its content hash so a compromised CDN cannot swap in malicious code.

```html
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous"></script>
```

- Generate the hash from the exact artifact you reviewed; a mismatch blocks the load.
- Pin a specific version URL — `integrity` is meaningless against a URL whose content legitimately changes.
- Bundlers (Vite, webpack) can emit SRI hashes for your own emitted chunks; enable it for production builds.

## Secrets and the Client Bundle

The defining front-end leak: a build tool inlines a secret into JS that ships to every browser. The threat model is simple — **anything imported into client-executed code is public.** View-source, the network tab, and `curl` all expose it.

| Rule | Why |
|------|-----|
| Confidential values live in server-only modules | Server Components, route handlers, `*.server.ts`, `server/` dirs are never sent to the browser |
| Only public-prefixed env vars reach client code | The prefix is the build tool's allow-list for client exposure — and a signal the value is non-secret |
| Never store a secret to ship a "convenient" client call | If the browser needs a privileged action, proxy it through your server |
| Rotate any secret that ever entered a commit or a built bundle | Git history and CDN caches are forever |

```ts
// LEAK — no public prefix, imported into a client component → inlined in the bundle
"use client";
const apiKey = process.env.OPENAI_API_KEY; // shipped to every visitor

// SAFE — read the secret only in a server route handler; the browser calls your proxy
// app/api/complete/route.ts  (server-only)
export async function POST(req: Request) {
  const key = process.env.OPENAI_API_KEY; // stays on the server
  // …call the upstream API, return only the result
}
```

## Environment-Variable Prefix Rules

The prefix that makes a var client-visible differs per tool. Anything **without** the prefix accessed in client code is undefined (Vite/SvelteKit) or a leak risk (Next inlines `NEXT_PUBLIC_*` and treats the rest as server-only).

| Framework / tool | Client-exposed prefix | Server-only |
|------------------|-----------------------|-------------|
| Next.js | `NEXT_PUBLIC_*` | everything else |
| Vite (React/Vue/Svelte) | `VITE_*` | everything else (not in `import.meta.env` client-side) |
| SvelteKit | `PUBLIC_*` (`$env/static/public`) | `$env/static/private`, `$env/dynamic/private` |
| Nuxt | `runtimeConfig.public.*` | `runtimeConfig.*` (server only) |
| Angular CLI | `NG_APP_*` (with `@ngx-env/builder`) / `environment.ts` | no built-in server tier — keep secrets off the client entirely |

The prefix is a *contract that the value is non-confidential*, not a way to "safely" expose a secret. A publishable Stripe key, a public analytics id, a feature flag — fine. An API secret, a signing key, a DB URL — never, regardless of prefix.

## Detecting a Bundle Leak

- Grep the **production build output** (`dist/`, `.next/static/`, `build/`) for known secret shapes (`sk_live_`, `AKIA`, `-----BEGIN`, long hex/base64 blobs, your own key prefixes).
- Add a CI step with a secret scanner (gitleaks / trufflehog) over both the repo and the built artifacts; fail the build on a hit.
- Audit every `process.env.*` / `import.meta.env.*` reference in client-tagged files (`"use client"`, `.client.ts`, components) — each must use the public prefix and carry nothing confidential.
- Keep all `.env*` out of git except `.env.example` (with placeholder values); a committed `.env` is a leak even if later removed.

## Checklist

- [ ] `script-src` has no `'unsafe-inline'` and no `'unsafe-eval'`; uses `'nonce-…' 'strict-dynamic'`.
- [ ] Nonce is cryptographically random and unique per response; threaded to all first-party inline scripts.
- [ ] `object-src 'none'`, `base-uri 'none'`/`'self'`, `frame-ancestors 'self'` set.
- [ ] CSP rolled out in `Report-Only` mode before enforcing.
- [ ] Trusted Types enabled (`require-trusted-types-for 'script'`) where Chromium coverage suffices; sanitize-before-sink kept as the cross-browser fallback.
- [ ] Third-party CDN scripts carry SRI `integrity` + `crossorigin`.
- [ ] No confidential env var imported into client-tagged code; only public-prefixed, non-secret values reach the bundle.
- [ ] Production build output scanned for secret shapes in CI; scanner fails the build on a hit.
- [ ] `.env*` git-ignored except `.env.example`; any once-committed secret rotated.

## Related

- `xss-and-injection.md` — the escaping/sanitization layer CSP and Trusted Types back up
- `../SKILL.md` — non-negotiable rules, CSP/secrets/supply-chain/SSRF summary tables
- `../../version-feature-matrix.md` — browser-feature floors (Trusted Types, CSP level) and framework env-prefix versions
