# XSS and Injection (Front-End / Web)

Use this when:

- You render user-influenced data into the DOM — user content, URL params, API responses, `postMessage` payloads, cookies, headers.
- You reach for a raw-HTML sink: `dangerouslySetInnerHTML`, `v-html`, `{@html}`, `innerHTML`, `outerHTML`, `[innerHTML]`, `insertAdjacentHTML`, `document.write`.
- You build an `href`/`src`/`style`/event-handler attribute from data.
- A reviewer flagged an XSS or unsafe-rendering finding.

Skip this file if:

- You are configuring CSP, Trusted Types, SRI, or chasing a secret leak. Use `csp-and-secrets.md`.
- You only need the rule summary. Use the parent `SKILL.md`.

Jump to:

- The Core Doctrine
- The Five Output Contexts
- Reflected, Stored, and DOM-Based XSS
- The Raw-HTML Escape Hatches (per framework)
- Sanitization with an Allow-List
- URL and `href` Injection
- Attribute and Event-Handler Injection
- `postMessage`, JSON, and Template Injection
- Why Dynamic Code Execution Is Banned
- XSS Checklist

## The Core Doctrine

There are two ways to put data into a page:

1. **As text** — the value is HTML-escaped (`<` → `&lt;`, `&` → `&amp;`, `"` → `&quot;`) and the browser renders it as literal characters. A value containing `<script>alert(1)</script>` shows up as visible text. This is the framework default for `{value}`, `{{ value }}`, `[textContent]`.
2. **As markup** — the value is parsed as HTML and can introduce elements, attributes, and script execution. Any raw-HTML sink does this. If the value came from untrusted input, the attacker controls the markup. This is **cross-site scripting (XSS)**.

**Always render untrusted data as text.** Text bindings are the security boundary: the framework escapes for the output context automatically. You forfeit that protection the instant you opt into a raw-HTML sink, a dynamic `href`, or a dynamic event handler — those are the only places XSS can enter a well-built component.

The context matters: HTML-escaping is correct inside element bodies, but an attribute, a URL, a `<style>`, or a `<script>` block needs *context-specific* encoding. Frameworks handle the common cases; the dangerous APIs below bypass them.

## The Five Output Contexts

The same value needs different encoding depending on where it lands. Frameworks escape HTML-body and attribute contexts for you; the others require care.

| Context | Example | Who escapes it | Your risk |
|---------|---------|----------------|-----------|
| HTML element body | `<div>{value}</div>` | Framework (auto) | Only if you use a raw-HTML sink |
| HTML attribute | `<img alt={value}>` | Framework (auto) | Quote-breaking only via raw sinks; event-handler attrs are different |
| URL (`href`/`src`/`action`) | `<a href={value}>` | **Not auto** — `javascript:` schemes execute | High — validate the scheme |
| JavaScript | inline handler built from data | **Never** — this is code | Critical — never build JS from data |
| CSS | `style={value}` / `<style>` | Partial — `url()`/`expression()` risks | Medium — avoid dynamic `<style>` from data |

## Reflected, Stored, and DOM-Based XSS

Same root cause (untrusted data → execution context), three delivery paths:

- **Reflected.** The server or SSR layer echoes a request value straight back. `searchParams.get("q")` interpolated into raw HTML, an error page that reflects the URL, an SSR template built from a header. *Defense:* never interpolate `req`/`searchParams`/headers into a raw-HTML sink; let the template engine escape.
- **Stored.** Hostile markup is persisted (a comment, a profile bio, a filename) and rendered later to other users. The blast radius is everyone who views it. *Defense:* the same render-time escaping plus sanitization on any path that must show rich text. Treat your own database as untrusted input.
- **DOM-based.** No server round-trip — client JS reads from `location`, `document.referrer`, `window.name`, or `postMessage` and writes into a sink. *Defense:* avoid the sink; use `textContent`/text bindings; sanitize before `innerHTML`; Trusted Types as a backstop.

```ts
// Reflected — DON'T (SSR builds raw HTML from a query param)
return `<h1>Results for ${searchParams.q}</h1>`; // XSS
// DO — render through a text binding
return <h1>Results for {searchParams.q}</h1>;

// DOM-based — DON'T
el.innerHTML = decodeURIComponent(location.hash.slice(1)); // XSS
// DO
el.textContent = decodeURIComponent(location.hash.slice(1));
```

## The Raw-HTML Escape Hatches (per framework)

Every framework auto-escapes text. Each also offers exactly one (or few) opt-out that parses a string as HTML. **These are the only XSS sinks in idiomatic component code — audit every use.**

| Framework | Auto-escaped (safe) | Raw-HTML sink (dangerous) |
|-----------|---------------------|---------------------------|
| React | `{value}` (children, attributes) | `dangerouslySetInnerHTML={{ __html: value }}` |
| Vue | `{{ value }}`, `:prop="value"` | `v-html="value"` |
| Svelte | `{value}` | `{@html value}` |
| Angular | `{{ value }}`, `[textContent]` (built-in sanitizer on `[innerHTML]`) | `[innerHTML]` with `bypassSecurityTrustHtml()`, `bypassSecurityTrust*` |
| SolidJS | `{value}` | `innerHTML={value}` prop |
| Vanilla DOM | `el.textContent`, `el.setAttribute` (mostly) | `el.innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write` |

Angular is the outlier: `[innerHTML]` runs the built-in DomSanitizer (strips scripts) — but `bypassSecurityTrustHtml()` disables it entirely. A `bypassSecurityTrust*` call on non-constant data is a P0 finding.

```tsx
// React — every one of these is an XSS vector if `body` is untrusted:
<div dangerouslySetInnerHTML={{ __html: body }} />
```
```vue
<!-- Vue -->
<div v-html="body"></div>
```
```svelte
<!-- Svelte -->
<div>{@html body}</div>
```
```html
<!-- Angular — bypass disables the sanitizer; never on user data -->
<div [innerHTML]="sanitizer.bypassSecurityTrustHtml(body)"></div>
```

Rule: a raw-HTML sink is acceptable only when the value is a **compile-time constant** or has passed through an **allow-list sanitizer** in the same expression. Anything else is P0.

## Sanitization with an Allow-List

When you genuinely must render rich HTML (a CMS body, a Markdown render), sanitize with a vetted allow-list library — never a hand-rolled regex (regex HTML stripping is bypassable and a perennial CVE source).

```ts
import DOMPurify from "dompurify";

// Allow-list: only these tags/attrs survive; scripts, event handlers,
// javascript: URLs, and unknown tags are removed.
const clean = DOMPurify.sanitize(dirtyHtml, {
  ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "p", "ul", "ol", "li", "code"],
  ALLOWED_ATTR: ["href", "title"],
});
```

- Sanitize as close to the sink as possible, ideally in the same expression, so no later code re-introduces unsafe content.
- For Markdown, render to HTML then sanitize, or use a renderer with a strict HTML mode; never trust raw Markdown to be HTML-safe (it can embed `<script>` and `<img onerror=…>`).
- On the server (SSR / Server Component), use `isomorphic-dompurify` or sanitize before serialization — DOM-based sanitizers need a DOM (jsdom) on the server.
- Sanitization complements, never replaces, a strict CSP (see `csp-and-secrets.md`). Defense in depth.

## URL and `href` Injection

Frameworks do **not** sanitize URL schemes. A `javascript:` (or `data:text/html`) URL in an `href`, `src`, `formaction`, or `xlink:href` executes when clicked or loaded.

```tsx
// DON'T — `javascript:alert(1)` runs on click
<a href={user.website}>site</a>
// DO — allow-list the scheme
function safeHref(url: string): string {
  try {
    const u = new URL(url, window.location.origin);
    return ["http:", "https:", "mailto:", "tel:"].includes(u.protocol) ? u.href : "#";
  } catch {
    return "#";
  }
}
<a href={safeHref(user.website)} rel="noopener noreferrer">site</a>
```

- Allow-list schemes (`http`, `https`, `mailto`, `tel`); reject `javascript:`, `data:`, `vbscript:`, `blob:` unless you specifically need them and have sanitized the contents.
- External links opened with `target="_blank"` need `rel="noopener noreferrer"` to block reverse-tabnabbing (the opened page getting `window.opener` access).
- Angular auto-sanitizes `[href]`/`[src]` bindings; Vue/React/Svelte do **not** — you own the scheme check.

## Attribute and Event-Handler Injection

- **Never build an event-handler attribute from data.** `onclick={userValue}` (as a string), `setAttribute("onclick", data)`, or `el.setAttribute("on" + name, code)` are code-execution sinks. Wire handlers through the framework's binding (`onClick={fn}`), never as string attributes.
- **`ref`/DOM-escape hatches** (`useRef` + manual DOM, Vue `ref`, Angular `ElementRef.nativeElement`) bypass the framework — if you touch `.innerHTML`/`.setAttribute` there, you own the encoding.
- **`style` from data** can smuggle `url(javascript:…)` (legacy) or break out of the attribute; prefer object-style bindings (`style={{ color }}`) over string concatenation, and never inject a full `<style>` block from user data.
- **`data-*` and `aria-*` attributes** built from data are text-escaped by the framework — safe — but if read back into a sink later they are untrusted again.

## `postMessage`, JSON, and Template Injection

- **`postMessage` listeners** must verify `event.origin` against an allow-list before trusting `event.data`. An unchecked listener that writes `event.data` into the DOM is DOM-XSS reachable by any frame.
- **JSON embedded in HTML** (SSR hydration state, `<script type="application/json">`) must be serialized with `<`/`>`/`&`/` `/` ` escaped, or an attacker-controlled string closes the `</script>` tag. Use the framework's serializer (Next/Nuxt/SvelteKit do this) or a library like `serialize-javascript`; never `JSON.stringify` raw into a `<script>` body.
- **Client-side template injection**: never `eval`/`new Function` a template string, and never feed user data into a templating engine that executes expressions.

## Why Dynamic Code Execution Is Banned

Constructing code from data and executing it collapses the data/code boundary — the most powerful primitive an attacker can reach. It is a non-negotiable rule: **no dynamic code construction or execution at runtime.**

| Banned construct | Why |
|------------------|-----|
| `eval(str)`, `new Function(str)` | Runs attacker-controlled JS |
| `setTimeout("code", n)`, `setInterval("code", n)` (string form) | Implicit `eval` |
| `javascript:` URL built from data | Executes on navigation |
| `el.innerHTML` / raw-HTML sink with `<script>` or `onerror=` | Markup injection → script execution |
| `import(/* dynamic attacker path */)` | Loads attacker-chosen module |
| building SQL/GraphQL by string concat on the server tier | Injection — use parameterized queries |

The replacement is always the same shape: **structured APIs over string interpolation.** Text bindings instead of raw HTML. Allow-list sanitization when rich HTML is unavoidable. A dispatch table (`Record<string, () => void>`) instead of `eval`-ing a function name. Parameterized queries instead of concatenated SQL on any backend boundary you touch.

If you believe you have a legitimate need to disable this rule, that requires a documented, reviewed justification recorded inline at the call site — and it must never accept data that crosses a trust boundary.

## XSS Checklist

- [ ] No untrusted data in `dangerouslySetInnerHTML` / `v-html` / `{@html}` / `[innerHTML]` / `innerHTML` / `insertAdjacentHTML` / `document.write`.
- [ ] Every raw-HTML sink value is either a compile-time constant or DOMPurify-sanitized in the same expression.
- [ ] No `bypassSecurityTrust*` (Angular) on non-constant data.
- [ ] `href`/`src`/`formaction` schemes allow-listed (`http(s)`/`mailto`/`tel`); `javascript:`/`data:` rejected.
- [ ] `target="_blank"` links carry `rel="noopener noreferrer"`.
- [ ] No event-handler attribute built from a string; handlers wired through framework bindings.
- [ ] `postMessage` listeners verify `event.origin` before using `event.data`.
- [ ] SSR/hydration JSON serialized with HTML-special chars escaped (no raw `JSON.stringify` into `<script>`).
- [ ] No `eval` / `new Function` / string-form `setTimeout`/`setInterval`.
- [ ] Rich-HTML rendering uses an allow-list sanitizer, never a regex strip.

## Related

- `csp-and-secrets.md` — CSP/nonces and Trusted Types that contain an XSS that slips through, plus secret-leak detection
- `../SKILL.md` — non-negotiable rules and the per-framework sink table
- `../../accessibility-baseline.md` — the parallel non-security UI gate
