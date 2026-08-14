---
name: web-performance
description: >-
  Core Web Vitals budgets and how to hit them — LCP < 2.5s, INP < 200ms,
  CLS < 0.1 — with the lab/field tooling (Lighthouse, PageSpeed, RUM) and the
  concrete levers for each metric. Use when setting a performance budget,
  diagnosing a poor Lighthouse score, profiling LCP/INP/CLS, or defining the
  performance completion gate.
---

# Web Performance

**Core Web Vitals budgets and the levers to meet them.** For the domain selection
table, see the canonical [quality-skills/SKILL.md](../quality-skills/SKILL.md) —
this leaf is the **canonical source for front-end performance budgets** and does
not duplicate the domain table. Bundle-size work lives in
[bundling-optimization](${CLAUDE_SKILL_DIR}/tooling/bundling-optimization/SKILL.md).

## When to Use

Use this skill when:
- Setting or enforcing a performance budget (the QA performance gate)
- A Lighthouse / PageSpeed score is poor and you need the right lever
- Diagnosing slow LCP, janky INP, or layout shift (CLS)
- Deciding lab vs field measurement

## The budget (the gate)

| Metric | Good (gate target) | Needs work | Poor |
|--------|--------------------|------------|------|
| **LCP** (Largest Contentful Paint) — loading | **< 2.5s** | 2.5–4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) — responsiveness | **< 200ms** | 200–500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) — visual stability | **< 0.1** | 0.1–0.25 | > 0.25 |

Targets are measured at the **75th percentile** of real users. The QA performance
gate passes when lab numbers clear these and no regression vs baseline is recorded
in DV Build Evidence. (INP replaced FID as the responsiveness Core Web Vital.)

> Core Web Vitals thresholds and INP-as-a-Core-Vital reflect the current standard; confirm current thresholds at web.dev/vitals before pinning a CI budget. Canonical: _shared/version-feature-matrix.md

## Lab vs field — measure both

| Source | Tool | Use for |
|--------|------|---------|
| **Lab** (synthetic, reproducible) | Lighthouse (`npx lighthouse <url>`), DevTools, WebPageTest | CI gate, before/after deltas, debugging |
| **Field** (real users, RUM) | Chrome UX Report (CrUX), `web-vitals` JS lib, PageSpeed Insights | the 75th-pct truth users actually experience |

Lab numbers are for catching regressions and debugging; **field data is the source
of truth** for whether real users are within budget. A great Lighthouse score with
poor CrUX means your lab profile doesn't match real conditions (devices, network).

```
npx lighthouse https://example.com --output=json --output-path=./.context/images/<id>/lighthouse.json
```

Attach the report as a supporting evidence row (`source: web-adapter`,
`notes: lighthouse`) per [CORPFLOW.md](../../../CORPFLOW.md).

## Levers per metric

### LCP < 2.5s (how fast the main content paints)

- **Prioritize the LCP resource:** `preload` the hero image/font; `fetchpriority="high"`
  on the LCP `<img>`; avoid lazy-loading the LCP element.
- **Cut render-blocking work:** inline critical CSS, defer non-critical CSS/JS,
  reduce the critical request chain.
- **Server/CDN:** fast TTFB, cache HTML/assets, use SSR/SSG/streaming so content
  arrives early.
- **Right-size images:** modern formats (AVIF/WebP), responsive `srcset`.

### INP < 200ms (how snappy interactions feel)

- **Reduce main-thread work:** break up long tasks (> 50ms); `scheduler.yield()`
  or chunk heavy work; move computation off the critical path (Web Workers).
- **Ship less JS:** the biggest INP lever is less JS to parse/execute — see
  [bundling-optimization](${CLAUDE_SKILL_DIR}/tooling/bundling-optimization/SKILL.md).
- **Avoid layout thrash:** batch DOM reads/writes; don't read layout in a loop.
- **Defer non-urgent updates:** `useTransition`/`useDeferredValue` (React), debounce
  expensive handlers.

### CLS < 0.1 (don't move content as it loads)

- **Reserve space:** explicit `width`/`height` or `aspect-ratio` on images, video,
  ads, embeds, iframes.
- **Fonts:** `font-display: swap` with a metrics-matched fallback (`size-adjust`)
  to limit reflow; preload the critical face.
- **Never insert content above existing content** after load (banners, late
  components) without reserved space.
- **Animate transform/opacity**, not properties that trigger layout.

## Profiling workflow

1. **Field first:** check CrUX / PageSpeed for the real 75th-pct numbers.
2. **Reproduce in lab:** Lighthouse + DevTools Performance trace under throttling
   matching your audience (mid-tier mobile, 4G).
3. **Find the metric's bottleneck:** LCP element in the trace; long tasks for INP;
   the Layout Shift entries for CLS.
4. **Apply the targeted lever**, re-measure, record the delta in Build Evidence.
5. **Stop at budget** — don't micro-optimize past the gate.

## Common Anti-Patterns

| Anti-pattern | Fix |
|--------------|-----|
| Lazy-loading the LCP hero image | eager + `fetchpriority="high"` + preload |
| Shipping a huge JS bundle (poor INP) | code-split / tree-shake to budget |
| Images/embeds with no dimensions (CLS) | reserve space (`aspect-ratio`) |
| Trusting Lighthouse over field data | validate against CrUX/RUM |
| One long main-thread task | chunk it / yield / Web Worker |
| Optimizing forever past the budget | stop at LCP<2.5/INP<200/CLS<0.1 |

## Related Skills

- [bundling-optimization](${CLAUDE_SKILL_DIR}/tooling/bundling-optimization/SKILL.md) — bundle size is the main INP/LCP lever
- [responsive-accessible-css](${CLAUDE_SKILL_DIR}/styling/responsive-accessible-css/SKILL.md) — CLS from late styles/fonts
- [quality-skills/SKILL.md](../quality-skills/SKILL.md) — canonical domain selection table
- [CORPFLOW.md](../../../CORPFLOW.md) — attaching Lighthouse as supporting evidence
- [version-feature-matrix](${CLAUDE_SKILL_DIR}/_shared/version-feature-matrix.md) — `scheduler.yield`/`fetchpriority` support notes
