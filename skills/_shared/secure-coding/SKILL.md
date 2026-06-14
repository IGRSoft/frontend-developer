---
name: secure-coding
description: Non-negotiable web security rules and bug-class defenses for front-end work — XSS-safe rendering, CSP and Trusted Types, CSRF and clickjacking defense, secrets-in-bundle hygiene, npm supply-chain integrity, and SSRF-resistant SSR fetch. Use when writing or reviewing UI code that renders untrusted data, injects HTML, configures headers, ships environment values to the client, adds a dependency, or fetches from the server during SSR.
---

# Secure Coding (Front-End / Web)

**Cross-framework web security rules that gate every diff. Violations are P0 review findings.**

## When to Use

Use this skill when:
- Code renders untrusted data into the DOM (user content, URL params, API responses, `postMessage` payloads).
- Code injects raw HTML (`dangerouslySetInnerHTML`, `v-html`, `{@html}`, `innerHTML`, `[innerHTML]`, `Element.insertAdjacentHTML`).
- Code configures a Content-Security-Policy, response headers, or framing controls.
- Code reads an environment variable that could ship into the client bundle.
- Code adds, upgrades, or installs an npm dependency (supply-chain surface).
- Code fetches a URL during SSR / a Server Component / a route handler (SSRF surface).
- Code stores tokens or session state in the browser (cookies, `localStorage`, `sessionStorage`).

## Non-Negotiable Rules

These mirror the global security rules and never have exceptions without a documented, reviewed justification:

1. **No untrusted data in an HTML sink** — never feed user-influenced strings to `innerHTML` / `dangerouslySetInnerHTML` / `v-html` / `{@html}` / `[innerHTML]`. Let the framework escape via text bindings (`{value}`, `{{ value }}`, `[textContent]`); if raw HTML is unavoidable, sanitize with an allow-list sanitizer (DOMPurify) **before** it reaches the sink.
2. **No dynamic code construction or execution** — no `eval`, `new Function(str)`, `setTimeout("code")`, `setInterval("code")`, or building a `<script>`/`javascript:` URL from data. Strict CSP must forbid `unsafe-eval` and `unsafe-inline`.
3. **Validate all external API responses** — assume every byte from a remote service (even "ours") is hostile; check status, content type, shape, and bounds before use. Parsing is not validation.
4. **No secret in the client bundle** — API keys, signing secrets, and service tokens never reach browser-shipped code. Only `NEXT_PUBLIC_*` / `VITE_*` / `PUBLIC_*` prefixed values are client-safe by design, and those must contain nothing confidential.
5. **Never disable a security control without documented justification** — a removed CSP directive, a `// eslint-disable-next-line no-danger`, a relaxed `sanitize: false`, a dropped `SameSite`, or `dangerouslyAllowBrowser`: each needs an inline comment with the reason and a tracking reference.

> A change that breaks any of these does not pass DR/SR review. See [workflow-integration](../workflow-integration/SKILL.md) for stage gates.

## XSS — the dominant web bug class → which sink, which defense

Cross-site scripting is the front-end equivalent of injection. Three delivery classes, one root cause: untrusted data reaching an execution context (HTML, attribute, URL, JS, or CSS) without context-correct escaping.

| Class | What it is | Caught/prevented by |
|-------|-----------|---------------------|
| **Reflected** | Server/SSR echoes a request value (query, header) straight into the response | Framework auto-escaping of template bindings; never interpolate `req`/`searchParams` into raw HTML |
| **Stored** | Hostile markup persisted server-side, later rendered to other users | Same escaping on render + sanitize on the dangerous-HTML path; treat the database as untrusted |
| **DOM-based** | Client JS writes attacker-controlled data into a sink with no server round-trip | Avoid the sink; use `textContent`/text bindings; sanitize before `innerHTML`; Trusted Types as a backstop |

The framework escapes text bindings for you — `{value}` (React), `{{ value }}` / `:prop` (Vue), `{value}` (Svelte), `{{ value }}` / `[prop]` (Angular) all HTML-escape by default. **You lose that protection the moment you opt into a raw-HTML sink.** The per-framework escape rules and the full sink catalog live in [references/xss-and-injection.md](references/xss-and-injection.md).

```tsx
// DON'T — attacker-controlled comment becomes live markup/script
<div dangerouslySetInnerHTML={{ __html: comment.body }} />
// DO — sanitize with an allow-list before the sink
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(comment.body) }} />
// BEST — render as text; no HTML sink at all
<div>{comment.body}</div>
```

Full doctrine — every framework's raw-HTML escape hatch, URL/`href` sanitization (`javascript:` schemes), attribute/event-handler injection, and `dangerouslySetInnerHTML`/`v-html`/`{@html}`/`[innerHTML]` discipline: [references/xss-and-injection.md](references/xss-and-injection.md).

## Content-Security-Policy & Trusted Types

CSP is the defense-in-depth layer that contains an XSS that slips past escaping. A strict, nonce-based policy neutralizes inline-script injection even when a sink is abused.

| Goal | Directive / mechanism |
|------|-----------------------|
| Block inline `<script>` and event-handler attributes | `script-src 'nonce-<random>' 'strict-dynamic'` — no `'unsafe-inline'` |
| Block `eval`/`new Function` | omit `'unsafe-eval'` |
| Block data exfiltration / unexpected fetch targets | `connect-src`, `default-src` allow-lists |
| Stop the page being framed (clickjacking) | `frame-ancestors 'self'` (preferred over the legacy `X-Frame-Options`) |
| Force DOM-sink injection through a policy | `require-trusted-types-for 'script'` + `trusted-types` (Baseline 2026 — verify Firefox/Safari minimums) |

A per-request **nonce** is the modern way to allow your own scripts under a strict CSP — frameworks expose it (`next.config` + middleware nonce, Nuxt `useHead`, Angular `ngCspNonce`). Never fall back to `'unsafe-inline'` to make a third-party widget work; nonce or hash it.

> Requires `require-trusted-types-for` / `trusted-types` (Baseline 2026 — newly cross-browser; verify Firefox/Safari minimums). Fallback: rely on sanitization + strict `script-src` nonce policy; treat Trusted Types as a progressive-enhancement backstop, not the only control. Canonical: _shared/version-feature-matrix.md

Full CSP directive reference, nonce wiring per framework, Subresource Integrity (SRI) for third-party scripts, and Trusted Types policy authoring: [references/csp-and-secrets.md](references/csp-and-secrets.md).

## CSRF & Clickjacking

| Risk | What it is | Defense |
|------|-----------|---------|
| **CSRF** | A cross-origin page triggers a state-changing request that rides the user's cookie | `SameSite=Lax` (default) or `Strict` cookies; double-submit or synchronizer token for cross-site flows; verify `Origin`/`Sec-Fetch-Site` on mutations |
| **Clickjacking** | Your UI is framed transparently and the user is tricked into clicking | `Content-Security-Policy: frame-ancestors 'self'`; legacy `X-Frame-Options: DENY` for old browsers |
| **Login CSRF / session fixation** | Attacker forces a known session onto the victim | Rotate session id on auth; `__Host-` cookie prefix, `Secure`, `HttpOnly` |

- Cookies holding auth state: `HttpOnly` (no JS access — blocks token theft via XSS), `Secure` (HTTPS only), `SameSite=Lax`/`Strict`. Prefer the `__Host-` prefix for the strongest binding.
- **Critical auth/permission decisions must not live in client code.** The client may hide a button, but the server/route-handler must re-check authorization on every mutation — a hidden control is not an access control.

## Secrets in the Bundle

The single most common front-end leak: a secret that the build inlines into JS that ships to the browser.

| Rule | How |
|------|-----|
| Only public-prefixed env vars reach the client | `NEXT_PUBLIC_*` (Next.js), `VITE_*` (Vite), `PUBLIC_*` (SvelteKit/Nuxt), `NG_APP_*` (Angular). Anything else accessed in client code is a bug or a leak. |
| Confidential values stay server-only | API keys, DB URLs, signing secrets live in server-only modules (`server/`, route handlers, `*.server.ts`, Server Components) — never imported into a client component |
| Scan the built bundle | grep the production build output for known secret shapes; gate in CI with a secret scanner |
| `.env*` never committed | `.gitignore` all `.env*` except `.env.example`; rotate any secret that ever touched a commit |

```ts
// DON'T — STRIPE_SECRET_KEY has no public prefix; importing it into a client
// component inlines the literal value into the browser bundle.
"use client";
const key = process.env.STRIPE_SECRET_KEY; // leaked at build time

// DO — read it only in a server module; expose just a publishable key.
const publishable = process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY;
```

Bundle-leak detection, env-var prefix rules per framework, and SRI: [references/csp-and-secrets.md](references/csp-and-secrets.md).

## npm Supply-Chain Integrity

Your dependency tree is attacker surface — typosquats, compromised maintainers, post-install scripts, and transitive CVEs.

| Risk | Defense |
|------|---------|
| Malicious / compromised package | Pin exact versions; commit the lockfile; `npm ci` (not `npm install`) in CI; review new deps and their transitive additions |
| Typosquatting | Verify the exact package name and weekly downloads/maintainer before adding |
| Malicious `postinstall` scripts | `npm config set ignore-scripts true` for untrusted installs; audit lifecycle scripts of new deps |
| Known CVEs | `npm audit` / `pnpm audit` in CI; fail on high/critical; route fixes to `frontend-developer:fe-dependency-manager` |
| Tampered registry artifact | Lockfile integrity hashes (`integrity:` `sha512-…`); enable npm provenance/attestation where available |

- One upgrade at a time, gated on a green build + test run (the deps-audit flow). Never bulk-bump across a major boundary blindly.
- Treat a transitive dependency the same as a direct one — `npm audit` reports the whole tree; a CVE three levels deep still ships to the user.

## SSRF via SSR / Server-Component Fetch

When SSR, a Server Component, a route handler, or a loader fetches a URL **derived from user input**, the request originates from your server — inside your network perimeter. That is Server-Side Request Forgery.

| Risk | Defense |
|------|---------|
| User controls the fetch target host | Allow-list permitted hosts/origins; reject anything not on it — never fetch an arbitrary user-supplied URL |
| Access to cloud metadata / internal services | Block private/link-local ranges (`169.254.169.254`, `127.0.0.0/8`, `10/8`, `192.168/16`, `::1`); resolve the hostname and re-check the resolved IP (defeats DNS rebinding) |
| Redirect to an internal target | Disable or bound redirect following; re-validate the post-redirect URL against the allow-list |
| Protocol smuggling | Allow only `https:` (and `http:` if required); reject `file:`, `gopher:`, `ftp:` |

```ts
// DON'T — Server Component fetches whatever host the user supplies.
const data = await fetch(`https://${searchParams.host}/api`); // SSRF

// DO — constrain to an allow-list; the host is never attacker-chosen.
const ALLOWED = new Set(["api.example.com", "cdn.example.com"]);
const url = new URL(userUrl);
if (url.protocol !== "https:" || !ALLOWED.has(url.hostname)) {
  throw new Error("host not allowed");
}
const data = await fetch(url);
```

## Diagnostic Table

| Symptom / finding | Likely cause | Fix | Reference |
|-------------------|-------------|-----|-----------|
| `dangerouslySetInnerHTML` / `v-html` / `{@html}` / `[innerHTML]` with non-static data | XSS sink | text binding, or DOMPurify before the sink | [xss-and-injection](references/xss-and-injection.md) |
| `href={userUrl}` or `:href` with no scheme check | `javascript:` URL XSS | allow-list `http(s):`/`mailto:`; strip `javascript:` | [xss-and-injection](references/xss-and-injection.md) |
| `eval` / `new Function` / `setTimeout("…")` | arbitrary code execution | dispatch table / parse, never eval; forbid `unsafe-eval` in CSP | [xss-and-injection](references/xss-and-injection.md) |
| `script-src 'unsafe-inline'` / `'unsafe-eval'` in CSP | XSS containment defeated | nonce + `strict-dynamic`; drop `unsafe-*` | [csp-and-secrets](references/csp-and-secrets.md) |
| no `frame-ancestors` / `X-Frame-Options` | clickjacking | `frame-ancestors 'self'` | this file, CSRF & Clickjacking |
| confidential value read in a client component | secret-in-bundle leak | move to server module; public prefix only for non-secrets | [csp-and-secrets](references/csp-and-secrets.md) |
| `process.env.SECRET` without public prefix in client code | secret inlined into bundle | server-only import; rotate the leaked secret | [csp-and-secrets](references/csp-and-secrets.md) |
| token in `localStorage`/`sessionStorage` | XSS-stealable credential | `HttpOnly` `Secure` `SameSite` cookie instead | this file, CSRF & Clickjacking |
| `npm install` in CI / no lockfile commit | supply-chain drift / unpinned deps | `npm ci` + committed lockfile + `npm audit` gate | this file, npm Supply-Chain |
| `npm audit` high/critical on a transitive dep | known CVE in the tree | one-at-a-time upgrade, build+test gate → `fe-dependency-manager` | this file, npm Supply-Chain |
| `fetch(userUrl)` in SSR / Server Component / route handler | SSRF | host allow-list + private-range block + protocol allow-list | this file, SSRF |
| missing `SameSite`/`HttpOnly` on auth cookie | CSRF / token theft | `__Host-` prefix, `HttpOnly`, `Secure`, `SameSite=Lax` | this file, CSRF & Clickjacking |

## Related Skills

- [xss-and-injection.md](references/xss-and-injection.md) — DOM/stored/reflected XSS sinks, framework escaping, URL/attribute injection, the raw-HTML escape hatches
- [csp-and-secrets.md](references/csp-and-secrets.md) — CSP directives and nonces, Trusted Types, SRI, env-var leakage, bundle-secret detection
- [accessibility-baseline.md](../accessibility-baseline.md) — the parallel non-security gate every UI change clears
- [severity-matrix.md](../severity-matrix.md) — P0–P3 mapping (XSS → P0, missing CSP → P1)
- [workflow-integration/SKILL.md](../workflow-integration/SKILL.md) — SR/DR security gates and handoff contract
- [version-feature-matrix.md](../version-feature-matrix.md) — browser-feature floors (Trusted Types, CSP level) and framework versions
