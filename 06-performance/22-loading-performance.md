# Lesson 22 — Loading performance & Core Web Vitals

> **Why this lesson exists:** render performance is what users feel *after* the app loads. Loading performance is whether they wait around to find out — and for most real applications it's the bigger problem. It's also measurable against public thresholds (Core Web Vitals), tied to search ranking, and the area where a single decision like "we imported all of `lodash`" costs more than a month of memoization.

**Time:** ~70 minutes · **Prereq:** Lesson 20 · [API Lesson 02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md)

---

## 1. The idea in one sentence

> **Every byte of JavaScript must be downloaded, parsed, compiled and executed before your app does anything — so the cheapest optimisation is always the code you don't ship.**

And the asymmetry that makes it urgent: **parse and execute cost scales with device class, not bandwidth.** 200KB of JS on a fast connection to a mid-range Android is still ~1 second of CPU before a single pixel changes.

---

## 2. The Core Web Vitals

| Metric | Measures | Good | Poor |
|---|---|---|---|
| **LCP** — Largest Contentful Paint | When the main content appears | ≤ 2.5s | > 4.0s |
| **INP** — Interaction to Next Paint | Responsiveness across the whole visit | ≤ 200ms | > 500ms |
| **CLS** — Cumulative Layout Shift | Visual stability | ≤ 0.1 | > 0.25 |

**Measured at the 75th percentile of real users**, not your laptop. And note **INP replaced FID in 2024** — FID only measured the *first* interaction's input delay; INP measures the full interaction-to-paint duration across the entire session, which is much harder to game and much closer to what users feel.

Supporting metrics worth knowing: **TTFB** (server responsiveness — the API track's territory), **FCP** (first pixel of anything), and **TBT** (total blocking time — the lab proxy for INP).

```tsx
import { onLCP, onINP, onCLS, onTTFB } from "web-vitals";
[onLCP, onINP, onCLS, onTTFB].forEach(fn =>
  fn(m => beacon("web_vital", { name: m.name, value: m.value, rating: m.rating,
                                route: location.pathname })));
```
**Tag by route.** An aggregate number hides the one page that's terrible.

---

## 3. Bundle size: find out what you actually ship

```bash
npx vite-bundle-visualizer          # Vite
npx source-map-explorer dist/assets/*.js
npx @next/bundle-analyzer           # Next.js
```

**Run this before optimising anything.** The results are routinely surprising, and the top offenders are usually the same handful:

| Offender | Typical cost | Fix |
|---|---|---|
| **`moment` + all locales** | ~230KB | `date-fns` (tree-shakeable) or `Intl.DateTimeFormat` (free) |
| **`lodash` (full import)** | ~70KB | `lodash-es` with named imports, or write the three functions you use |
| **A charting library** | 200–500KB | `lazy()` it; most users never scroll to the chart |
| **An icon library, fully imported** | 100KB+ | Per-icon imports, or an SVG sprite |
| **A duplicated dependency** | 2× everything | `npm ls <pkg>`, then `overrides`/`resolutions` |
| **A moment/date library in a *dependency*** | invisible | Check the analyzer, not just your own imports |

```tsx
// ❌ 70KB for one function
import _ from "lodash";
const grouped = _.groupBy(payments, "status");

// ✅ ~2KB
import groupBy from "lodash-es/groupBy";

// ✅ 0KB — and it's four lines
const grouped = Object.groupBy(payments, p => p.status);     // ES2024, widely supported
```

**The date one is worth calling out specifically:** `Intl.DateTimeFormat` and `Intl.NumberFormat` are built into every browser, handle every locale, and cost nothing. Most date libraries in most apps could be deleted. (Just **hoist the formatter** — constructing one per render is the trap from [Lesson 21](21-render-performance.md).)

```bash
# Before adding any dependency
npx bundlephobia <package>      # size, and whether it tree-shakes
```

---

## 4. Code splitting

```tsx
// 1. Route-level — the highest-value split by far
const PaymentsPage = lazy(() => import("./pages/Payments"));
const SettingsPage = lazy(() => import("./pages/Settings"));
const ReportsPage  = lazy(() => import("./pages/Reports"));
// A user who never opens Reports never downloads it.

// 2. Heavy components below the fold
const RevenueChart = lazy(() => import("./RevenueChart"));       // 300KB of charting
<Suspense fallback={<ChartSkeleton height={240} />}><RevenueChart /></Suspense>

// 3. Modals and drawers — code for UI that may never open
const RefundModal = lazy(() => import("./RefundModal"));
{isOpen && <Suspense fallback={null}><RefundModal /></Suspense>}

// 4. Conditional features — admin, dev tools, feature-flagged code
{principal.role === "owner" && <Suspense><AdminPanel /></Suspense>}
```

**Route splitting first.** It's usually a 50–70% reduction in initial JS for a multi-page app, for about ten lines of change.

### Preloading — splitting without the delay
```tsx
// Prefetch on hover/intent: by the time the click lands, the chunk is there
<Link to="/reports" onMouseEnter={() => import("./pages/Reports")}>Reports</Link>

// Or declaratively, in the route definition
{ path: "reports", lazy: () => import("./pages/Reports") }
```
**This is the technique that makes code splitting invisible.** Without it, splitting trades initial load for a delay on navigation; with it, you get both.

```html
<!-- Resource hints — tell the browser early -->
<link rel="preconnect" href="https://api.ledger.dev" crossorigin>    <!-- DNS+TCP+TLS early -->
<link rel="dns-prefetch" href="https://api.ledger.dev">
<link rel="preload" href="/fonts/Inter.woff2" as="font" type="font/woff2" crossorigin>
<link rel="modulepreload" href="/assets/vendor.js">
```
`preconnect` to your API origin removes DNS + TCP + TLS from the critical path of the first API call — 100–300ms on mobile, for one line ([API Lesson 02](https://github.com/Pawar-Pratik/API-mastery/blob/main/01-foundations/02-journey-of-a-request.md)).

### The splitting mistakes
```tsx
// ❌ Splitting something tiny — a chunk request costs more than 3KB of inline code
const Badge = lazy(() => import("./Badge"));

// ❌ Splitting above-the-fold content — you've made LCP worse
const Header = lazy(() => import("./Header"));

// ❌ A lazy component with no error boundary — chunks fail, especially after deploys
<Suspense><Chart /></Suspense>                // Lesson 15

// ❌ Waterfall splitting: a lazy component that lazily imports another
```

---

## 5. Fixing each vital

### LCP — make the main content appear sooner
```
Causes, in rough order of frequency:
1. Slow TTFB               → server/CDN/region (API Lessons 02, 17)
2. Render-blocking JS/CSS  → defer JS, inline critical CSS
3. The LCP element is an image that loads late → preload it, and don't lazy-load it
4. Client-side rendering   → the content doesn't exist until JS runs (§6)
5. Web fonts blocking text → font-display: swap
```
```tsx
// The LCP image — this is the single most common LCP fix
<img src={hero} alt="" fetchPriority="high" loading="eager" width={1200} height={600} />
// ❌ loading="lazy" on the LCP image ACTIVELY HURTS — the browser deprioritises it
```
**Lazy-loading the hero image is a real and common own-goal**: a blanket `loading="lazy"` on all images includes the one that defines your LCP.

```css
/* Fonts: show text immediately in a fallback, swap when the web font arrives */
@font-face { font-family: Inter; src: url(/fonts/Inter.woff2) format("woff2"); font-display: swap; }
/* And size-adjust to reduce the layout shift when it swaps */
```

### INP — keep interactions responsive
```
Causes:
1. Long tasks blocking the main thread (>50ms)
2. Expensive renders on interaction              → Lesson 21
3. Heavy event handlers
4. Third-party scripts (analytics, chat widgets, tag managers)
```
```tsx
// Break up long work so the browser can paint between chunks
async function processLarge(items: Item[]) {
  for (let i = 0; i < items.length; i += 100) {
    process(items.slice(i, i + 100));
    await scheduler.yield?.() ?? new Promise(r => setTimeout(r, 0));   // yield to the browser
  }
}

// Or mark the update low-priority so the click still paints (Lesson 11)
startTransition(() => setExpensiveState(next));
```
**Third-party scripts are the most common INP killer and the least examined.** A tag manager that loads four analytics scripts, a chat widget, and a session recorder can add hundreds of milliseconds of main-thread work on every interaction. Audit them; load them with `defer`, or in a worker (Partytown), or not at all.

### CLS — stop things moving
```tsx
// ❌ The image has no dimensions → content jumps when it loads
<img src={logo} />
// ✅ Reserve the space
<img src={logo} width={120} height={40} alt="" />
// Or in CSS: aspect-ratio: 16 / 9;
```
```tsx
// ❌ A skeleton that's a different size from the real content → a shift on swap
<Suspense fallback={<div>Loading…</div>}><Chart /></Suspense>
// ✅ Match the real dimensions
<Suspense fallback={<Skeleton height={240} />}><Chart /></Suspense>
```
**A skeleton whose shape doesn't match the content it replaces causes CLS** — which means "add a loading skeleton" can make your vitals *worse* if done carelessly. Other common causes: ads and embeds without reserved space, banners injected at the top of the page, and web fonts with very different metrics.

---

## 6. Rendering strategy

The biggest LCP lever, and it's an architecture decision rather than an optimisation.

| Strategy | First paint | Best for |
|---|---|---|
| **CSR** (Vite SPA) | Blank until JS runs, then fetch, then render | Dashboards behind a login |
| **SSR** | HTML immediately, hydrate after | Content that must be fast and indexable |
| **SSG** | Pre-built HTML from a CDN — fastest possible | Marketing, docs, blogs |
| **ISR** | SSG + periodic regeneration | Semi-dynamic content at scale |
| **RSC + streaming** | HTML streams as data resolves | Mixed content/app ([Lesson 23](../07-server-react/23-rsc.md)) |

```
CSR waterfall:   HTML (empty) → JS → parse+execute → fetch data → render
                 └───────────────── LCP is at the end ─────────────────┘

SSR/RSC:         HTML (with content) → LCP ✅ → JS → hydrate → interactive
```

**For Ledger Console specifically, CSR is the right call** — it's an authenticated dashboard, so there's nothing to index and no anonymous first-visit to optimise. **Being able to say "we don't need SSR here, and here's why" is a stronger answer than reflexively reaching for Next.js.** The marketing site in front of it is a different product with different constraints, and should probably be SSG.

---

## 7. Assets and caching

```html
<!-- Images: modern formats, responsive sizes, explicit dimensions -->
<img
  src="/chart.webp"
  srcSet="/chart-400.webp 400w, /chart-800.webp 800w, /chart-1600.webp 1600w"
  sizes="(max-width: 768px) 100vw, 800px"
  width={800} height={400}
  loading="lazy" decoding="async" alt="Revenue over time"
/>
```
AVIF/WebP are 30–50% smaller than JPEG. `srcSet` + `sizes` stops you sending a 1600px image to a 400px viewport — which is routinely the single largest asset on a page.

```
Caching strategy (API Lesson 17, applied to assets):
  index.html          Cache-Control: no-cache          ← must revalidate, it names the chunks
  /assets/*.[hash].js Cache-Control: max-age=31536000, immutable   ← content-hashed, cache forever
  /fonts/*.woff2      Cache-Control: max-age=31536000, immutable
```
**Content-hashed filenames plus `immutable` is the whole strategy:** the HTML is always fresh and points at hashed assets that never need revalidating. Getting this backwards — caching `index.html` aggressively — is how users get stuck on an old version.

**Compression:** Brotli beats gzip by ~15–20% on text at comparable decode speed. Pre-compress at build time for static assets; most CDNs do it automatically.

---

## 8. Ledger Console's loading budget

```
Initial bundle (gzipped)
  react + react-dom            45 KB
  @tanstack/react-query        13 KB
  react-router                 12 KB
  zod                          14 KB     ← consider valibot (~3KB) if this matters
  app code (shell + payments)  60 KB
  ─────────────────────────────────
  TOTAL                       144 KB     ✅ under the 200KB budget

Lazy chunks
  /reports  + recharts        180 KB     ← most users never open it
  /settings                    25 KB
  RefundModal                  12 KB

Measured (4G, 4× CPU throttle)
  TTFB   180ms    LCP 1.8s    INP 90ms    CLS 0.02
```

```jsonc
// Enforce it — a budget nobody enforces is a wish
// vite.config.ts
build: { rollupOptions: { output: { manualChunks: {
  vendor: ["react", "react-dom"],
  query: ["@tanstack/react-query"],
}}}}
// + a size-limit CI check
{ "size-limit": [{ "path": "dist/assets/index-*.js", "limit": "150 KB" }] }
```

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Run a bundle analyzer before optimising** | The top offenders are always surprising |
| **Check `bundlephobia` before adding a dependency** | The cheapest fix is the code you don't ship |
| **Route-level code splitting first** | 50–70% initial reduction for ~10 lines |
| **Prefetch chunks on hover/intent** | Makes splitting invisible to the user |
| **Never lazy-load the LCP image** | `loading="lazy"` on the hero actively hurts |
| **`fetchPriority="high"` + `preload` on the LCP element** | The most common single LCP fix |
| **Always set `width`/`height` or `aspect-ratio` on images** | The most common CLS cause |
| **Skeletons must match the real content's dimensions** | Otherwise your loading state *causes* CLS |
| **`font-display: swap` + `size-adjust`** | Text visible immediately, minimal shift |
| **`preconnect` to your API origin** | Removes DNS+TCP+TLS from the first request |
| **Content-hash assets + `immutable`; `no-cache` on `index.html`** | Fresh entry point, permanently cached assets |
| **Audit third-party scripts** | The most common INP killer, and the least examined |
| **Use `Intl` instead of a date/number library** | Free, complete, already in the browser |
| **Measure at p75 on throttled hardware; enforce a budget in CI** | Your laptop isn't your users |

---

## 10. Interview traps

**Q1. "Your LCP is 4.2s. Walk me through the investigation."**
Classify first: is the delay in TTFB, in resource loading, or in rendering? Check TTFB (server/CDN/region), then whether the LCP element is an image loading late or content that requires JS. Then the fixes in order: server/CDN, render-blocking resources, preload the LCP image with `fetchPriority="high"`, and — the architectural one — consider SSR/SSG if the content doesn't exist until JS runs.

**Q2. "What are the Core Web Vitals and what do they measure?"**
LCP (main content appears, ≤2.5s), INP (interaction responsiveness, ≤200ms), CLS (visual stability, ≤0.1), all at **p75 of real users**. **Note that INP replaced FID in 2024** — FID only measured the first interaction's input *delay*, INP measures full interaction-to-paint across the session. Knowing that change is a small freshness signal.

**Q3. "How do you reduce bundle size?"**
Measure first with an analyzer. Then: route-level code splitting, replace heavy dependencies (`moment`→`Intl`, full `lodash`→named imports), lazy-load below-the-fold and conditional features, check for duplicated dependencies, and verify tree-shaking actually works (`sideEffects: false`, ES modules).

**Q4. "What causes layout shift and how do you fix it?"**
Images without dimensions, ads/embeds without reserved space, web fonts with different metrics, content injected above existing content, and — the one people cause themselves — **skeletons that don't match the real content's size.** Fixes: explicit `width`/`height` or `aspect-ratio`, reserved space, `font-display: swap` with `size-adjust`.

**Q5. "What's the downside of code splitting?"**
Each chunk is a request, so over-splitting adds round trips; a chunk loading on navigation is a delay unless you prefetch; chunks can fail (especially after a deploy, when the old HTML references a hash that no longer exists), so every `lazy` needs an error boundary. And splitting above-the-fold content makes LCP worse.

**Q6. "CSR, SSR or SSG for this app?"**
Ask what it is. Authenticated dashboard → CSR is fine; there's nothing to index and no anonymous first paint to optimise. Public content → SSG or SSR for LCP and SEO. Mixed → RSC with streaming. **Being able to say "we don't need SSR here" is a stronger answer than defaulting to Next.js.**

**Q7. "Your INP is 400ms. What's likely?"**
Long tasks blocking the main thread — expensive renders on interaction, heavy event handlers, or **third-party scripts** (the most common and least audited). Fixes: break work into chunks yielding to the browser, use `startTransition` for non-urgent updates, virtualize, and audit/defer third-party code.

**Q8. "How should assets be cached?"**
Content-hashed filenames with `max-age=31536000, immutable`, and `no-cache` on `index.html` so it always revalidates and points at the current hashes. Getting it backwards strands users on an old version.

**Q9. "How do you stop bundle size regressing?"**
A `size-limit` (or Lighthouse) budget as a required CI check, plus the analyzer output on PRs that change dependencies. **An unenforced budget is a wish.**

**Q10. "The app is fast on your machine and slow for users. Why?"**
Device class (parse and execute scale with CPU, not bandwidth), network (RTT is distance-bound), data volume, and cold caches on a first visit. **Always measure at p75 on throttled hardware** — and the gap between your Lighthouse score and your field data is usually the size of that mistake.

---

## 11. Build & break

### Build — the audit
1. Run a bundle analyzer on Ledger Console. Write down the top five contributors.
2. Replace one heavy dependency with a built-in (`Intl`, `Object.groupBy`, `structuredClone`).
3. Add route-level splitting plus hover prefetching.
4. Add `preconnect` to the API origin and `preload` for the font.
5. Measure LCP/INP/CLS before and after, at 4× CPU and Fast 4G.

**Record the before/after table.** That's the artefact — "I cut initial JS from 280KB to 144KB and LCP from 3.1s to 1.8s" is a sentence with evidence behind it.

### Build — the budget
Add `size-limit` with a 150KB cap on the main chunk and a Lighthouse CI assertion for performance ≥ 0.9. Then **deliberately import `moment`** and watch CI fail. That failure is the point — it's what keeps the budget real after you stop paying attention.

### Build — the vitals pipeline
Wire `web-vitals` to your backend, tagged by route and device class. After a week, look at p75 per route. **The route you didn't expect will be the worst one** — that's why aggregate numbers are useless.

### Break — five experiments
1. **Lazy-load the hero image.** Add `loading="lazy"` to the LCP element and measure LCP before and after. It gets worse.
2. **Mismatched skeleton.** Use a 40px skeleton for a 240px chart. Measure CLS.
3. **Missing image dimensions.** Remove `width`/`height` from three images and watch CLS climb as they load.
4. **Over-splitting.** `lazy()` a 3KB component and look at the Network panel — the request overhead exceeds the code.
5. **A third-party script.** Add a tag manager or session recorder to a page and measure INP before and after. The number is usually shocking.

### Explain out loud (90 seconds)
1. The three Core Web Vitals, their thresholds, and what INP replaced.
2. The bundle-size checklist, in order.
3. Why lazy-loading the LCP image hurts.
4. Two ways a loading state can *cause* CLS.
5. How you'd choose a rendering strategy for a given app.

---

## Module 6 complete — checkpoint

- [ ] Classify load vs render before touching anything
- [ ] The four measurement tools and what each is for
- [ ] Reading a flame graph: wide bar vs many narrow bars
- [ ] Why you profile production builds, throttled, with realistic data
- [ ] The five-level fix hierarchy
- [ ] How you'd render a million rows — **starting with the server**
- [ ] Three details that make virtualization correct
- [ ] `transform` vs `top`; layout thrashing
- [ ] The three Core Web Vitals and their thresholds
- [ ] The bundle-size checklist
- [ ] Why lazy-loading the LCP image hurts
- [ ] How skeletons can cause CLS
- [ ] Choosing a rendering strategy

---

## What's next

Module 7 is server React — the biggest architectural change since hooks. It starts with the question most candidates fumble: **what is a Server Component, mechanically?**

Next → **[Lesson 23: React Server Components](../07-server-react/23-rsc.md)**
