# Lesson 25 — SSR, streaming & hydration

> **Why this lesson exists:** hydration is the least-understood part of React, and *"Text content does not match server-rendered HTML"* is the most confusing error a React developer meets. It's also where the real trade-offs of server rendering live — you get a fast first paint and pay for it with a period where the page **looks** interactive and isn't. Understanding that gap is what lets you choose a rendering strategy rather than copy one.

**Time:** ~65 minutes · **Prereq:** Lessons 23, 24

---

## 1. The idea in one sentence

> **Server rendering produces HTML the user can see immediately; hydration attaches React to that HTML so it becomes interactive — and every server-rendering problem lives in the gap between those two moments.**

---

## 2. The timeline

```
CSR (a Vite SPA)
  HTML (empty div) → JS downloads → parses → executes → render → fetch → render
  └──────────────────── nothing visible this whole time ────────────────────┘
                                                                    ↑ LCP

SSR
  HTML (with content) → visible ✅ → JS downloads → hydrate → interactive
       ↑ LCP                                              ↑ TTI
       └────────── THE UNCANNY VALLEY: looks ready, isn't ─────────┘

SSG
  HTML from a CDN → visible ✅ (fastest possible) → JS → hydrate → interactive
```

**The uncanny valley is the honest cost of SSR**, and it's what interviewers are really probing when they ask about it: for a period, the user sees a complete page, clicks a button, and nothing happens. On a slow device that window can be seconds.

Mitigations, and they're real: stream the HTML so content arrives sooner, ship less JavaScript so hydration starts sooner, use RSC so less needs hydrating at all, and — the underrated one — **make the pre-hydration state honest**. A button that does nothing is worse than a button that isn't there yet.

---

## 3. Hydration, mechanically

```tsx
// Server
const html = renderToString(<App />);           // or renderToPipeableStream
// → <div id="root"><button>Count: 0</button></div>

// Client
hydrateRoot(document.getElementById("root")!, <App />);
```

**Hydration is not rendering — it's *matching*.** React walks the existing DOM and the component tree in parallel, and where they agree it attaches event listeners and builds the fiber tree **without creating any DOM nodes**. It's much cheaper than rendering from scratch, and that's the whole point.

Crucially: **React trusts the server's HTML.** It assumes the markup matches what the client would produce and doesn't verify attributes or text in the general case. When they disagree, you get a mismatch.

---

## 4. Hydration mismatches

```
Error: Text content does not match server-rendered HTML.
Warning: Expected server HTML to contain a matching <div> in <span>.
```

**The five causes, which cover essentially every real occurrence:**

### 1. Non-deterministic values
```tsx
// ❌ The server renders one value, the client another
<span>{new Date().toLocaleTimeString()}</span>
<span>{Math.random()}</span>
<span>{crypto.randomUUID()}</span>
```

### 2. Browser-only APIs
```tsx
// ❌ `window` doesn't exist on the server
const isMobile = window.innerWidth < 768;
const theme = localStorage.getItem("theme");
```

### 3. Locale and timezone differences
```tsx
// ❌ The server is UTC and in en-US; the user is IST and in en-GB
<span>{date.toLocaleDateString()}</span>
<span>{new Intl.NumberFormat().format(4999)}</span>
```
**This is the sneakiest one**, because it works perfectly on your machine and breaks for users in other timezones. Always pass an explicit locale and timezone, or render the raw value and format after hydration.

### 4. Invalid HTML nesting
```tsx
// ❌ The browser silently "fixes" this, so the DOM no longer matches what React rendered
<p><div>Hello</div></p>          // browsers close the <p> before the <div>
<table><div /></table>
<a><a>nested</a></a>
```
A genuinely confusing one, because your JSX is fine — the *browser* changed the DOM before React looked at it.

### 5. Extensions and third-party scripts
Password managers, translation tools and ad blockers inject attributes and elements into the DOM before hydration. Usually harmless; occasionally the cause of a mismatch you can't reproduce. `suppressHydrationWarning` on the affected element is the accepted escape hatch.

### The fixes

```tsx
// ✅ Fix 1 — render only after mount (the standard pattern)
function ClientOnly({ children, fallback = null }: Props) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted ? <>{children}</> : <>{fallback}</>;
}
<ClientOnly fallback={<Skeleton />}><LocalTime /></ClientOnly>
```

```tsx
// ✅ Fix 2 — suppress, for genuinely unavoidable single values
<time dateTime={iso} suppressHydrationWarning>{formatted}</time>
// One level deep only, and never as a blanket fix — it hides real bugs.
```

```tsx
// ✅ Fix 3 — make it deterministic (the best fix when available)
<time dateTime={iso}>{formatInTimeZone(iso, "UTC", "d MMM yyyy")}</time>
// Server and client compute the same string because the timezone is explicit.
```

```tsx
// ✅ Fix 4 — useId for generated ids (Lesson 11)
const id = useId();          // stable across server and client
```

```tsx
// ✅ Fix 5 — dynamic import with SSR disabled, for components that can't render server-side
const Chart = dynamic(() => import("./Chart"), { ssr: false, loading: () => <ChartSkeleton /> });
```

> **Why mismatches matter more than they look:** React 18+ recovers by re-rendering the mismatched subtree on the client, which is silent but expensive, and in the worst case a whole page falls back to client rendering — **losing the entire benefit of SSR** while still paying for it. A mismatch isn't a cosmetic warning; it can quietly double your LCP.

### The theme flash — the classic, and worth knowing the fix
```tsx
// The problem: the server can't know the user's theme, so it renders light and the client flips to dark.
// ✅ A blocking inline script that runs BEFORE paint
<script dangerouslySetInnerHTML={{ __html: `
  try {
    const t = localStorage.theme ?? (matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
    document.documentElement.dataset.theme = t;
  } catch {}
`}} />
```
It sets an attribute on `<html>` before React runs, so CSS applies the right theme immediately and React never renders theme-dependent markup. **This is the standard solution** (it's what `next-themes` does), and it's a good small answer to "how do you avoid a flash of wrong theme?"

---

## 5. Streaming SSR

Classic SSR blocks: the server renders everything, *then* sends one HTML string. One slow query delays the entire page.

```tsx
// Streaming — send the shell immediately, stream the rest as it resolves
export default async function Page() {
  return (
    <>
      <Header />                                       {/* flushed immediately */}
      <Suspense fallback={<TableSkeleton />}>
        <PaymentsTable />                               {/* streams when data resolves */}
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />                                {/* streams independently */}
      </Suspense>
    </>
  );
}
```

```
Time →
0ms    <html><head>…</head><body><header>…</header>
       <div id="S:0"><!--skeleton--></div>
       <div id="S:1"><!--skeleton--></div>            ← sent immediately, page is VISIBLE
200ms  <template id="P:0">…table html…</template>
       <script>$RC("S:0","P:0")</script>              ← swapped in place
900ms  <template id="P:1">…chart html…</template>
       <script>$RC("S:1","P:1")</script>
```

**React streams out-of-order.** Each suspended boundary's content is sent when *it* is ready, with a tiny inline script that moves it into place. So a fast section never waits for a slow one, regardless of document order.

### Selective hydration
The other half, and it's the part that surprises people:

> **React hydrates in priority order, and a user interaction promotes that part of the tree to the front of the queue.** Click a not-yet-hydrated button and React hydrates that subtree first, then replays the event.

So a slow chart at the bottom doesn't block the header's buttons from becoming interactive. **This is the concrete payoff of concurrent rendering** ([Lesson 11](../03-hooks/11-concurrent-hooks.md)) applied to hydration, and it's a strong thing to be able to explain — most people know streaming exists but not that hydration is also incremental and interaction-prioritised.

---

## 6. Choosing a rendering strategy

| Strategy | First paint | Data freshness | Cost | Right for |
|---|---|---|---|---|
| **CSR** | Slowest | Always fresh | Cheapest to host | Authenticated dashboards, internal tools |
| **SSR** | Fast | Fresh per request | A server per request | Personalised, SEO-sensitive pages |
| **SSG** | **Fastest** (CDN) | Build-time | Cheapest at scale | Marketing, docs, blogs |
| **ISR** | Fastest | Periodically stale | Cheap | Semi-dynamic content at scale |
| **RSC + streaming** | Fast, progressive | Fresh | A server | Mixed content/app |

```
Decision questions, in order:
1. Does it need to be indexed or shared as a link preview?     no → CSR is fine
2. Is the content the same for everyone?                        yes → SSG/ISR
3. Is it behind a login?                                        yes → CSR or SSR, SEO is irrelevant
4. Is first paint a business metric (bounce, conversion)?        yes → SSR/SSG
5. Is nearly everything interactive?                             yes → CSR; RSC buys little
```

**For Ledger:** the Console is CSR (authenticated, interactive, nothing to index); the marketing site and docs are SSG; hosted receipt pages that customers open from an email are SSR (public, must be fast, personalised). **Three products, three strategies, chosen by constraint** — and being able to give that answer rather than "we use Next.js" is the point.

### The SSR costs people don't mention
- **You now run servers** — scaling, cold starts, regional deployment, and a rendering CPU cost per request
- **Every component must be SSR-safe** — no `window` at module scope, no browser-only libraries without guards
- **Two environments to debug**, with different failure modes
- **Hydration mismatches** become a permanent class of bug
- **Caching gets harder** — personalised HTML can't be cached at the edge the way static HTML can

---

## 7. SSR-safe code

```tsx
// ❌ Runs at module load — crashes on the server
const width = window.innerWidth;
const stored = localStorage.getItem("theme");

// ✅ Inside an effect, which never runs on the server
useEffect(() => { setWidth(window.innerWidth); }, []);

// ✅ Or useSyncExternalStore with a server snapshot (Lesson 11)
const isOnline = useSyncExternalStore(subscribe, () => navigator.onLine, () => true);
//                                                                        ↑ server value
```

```tsx
// ❌ useLayoutEffect doesn't run on the server — React warns
useLayoutEffect(() => { measure(); }, []);

// ✅ The standard isomorphic alias
const useIsomorphicLayoutEffect = typeof window !== "undefined" ? useLayoutEffect : useEffect;
```

```tsx
// ❌ Module-scope randomness or time — different on each side
const sessionId = crypto.randomUUID();

// ✅ useId, or generate it in an effect
const id = useId();
```

**The checklist for any component that will be server-rendered:**
```
[ ] No window/document/localStorage/navigator at module scope or during render
[ ] No Date.now(), Math.random(), or crypto.randomUUID() in render output
[ ] Dates and numbers formatted with an explicit locale AND timezone
[ ] useId for generated ids, never a counter or random
[ ] useIsomorphicLayoutEffect instead of useLayoutEffect
[ ] Valid HTML nesting (no <div> inside <p>, no <div> inside <table>)
[ ] Third-party components checked for SSR support; dynamic({ssr:false}) if not
[ ] Environment variables: only public ones referenced in client code
```

---

## 8. Ledger's rendering decisions

```
ledger.dev                (marketing)     → SSG. Content is identical for everyone; LCP matters
docs.ledger.dev           (documentation) → SSG + search. Indexed, static
status.ledger.dev         (status page)   → ISR, 30s. Semi-dynamic, high traffic during incidents
app.ledger.dev            (Console)       → CSR (Vite SPA). Authenticated, interactive
pay.ledger.dev/r/:token   (receipt page)  → SSR. Public link from an email, personalised, must be fast
```

**The receipt page is the interesting one**, and it's a good worked example:
```tsx
// Opened from an email, by someone with no session, possibly on a poor connection.
// Requirements: fast first paint, correct link previews, works with JS disabled.
export default async function ReceiptPage({ params }: Props) {
  const { token } = await params;
  const receipt = await getReceiptByToken(token);        // a signed, single-purpose token
  if (!receipt) notFound();

  return (
    <article>
      <h1>Receipt for {formatMoney(receipt.money)}</h1>
      <time dateTime={receipt.paidAt}>
        {formatInTimeZone(receipt.paidAt, receipt.merchantTimezone, "d MMMM yyyy")}
      </time>                                              {/* ← explicit tz: no mismatch */}
      <DownloadPdfButton token={token} />                  {/* the only client component */}
    </article>
  );
}
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const receipt = await getReceiptByToken((await params).token);
  return { title: `Receipt ${receipt?.id}`, openGraph: { title: "Your receipt from Ledger" } };
}
```
**Note the token design:** a signed, single-purpose, expiring token in the path rather than a session — because the recipient has no account. That's an API-design decision ([API Lesson 12](../../API/03-security/12-authentication-landscape.md)) driving a rendering decision.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Choose a strategy per surface, not per product** | One app can be CSR, another SSG, in the same company |
| **Treat hydration mismatches as bugs, not warnings** | React may client-render the whole page, losing SSR's benefit entirely |
| **Format dates and numbers with an explicit locale and timezone** | The sneakiest mismatch — works on your machine, breaks abroad |
| **`useId` for generated ids** | Counters and randomness differ across server and client |
| **`useIsomorphicLayoutEffect`, or question the layout effect** | It doesn't run on the server and React warns |
| **No browser APIs at module scope or during render** | They crash SSR outright |
| **Validate HTML nesting** | The browser silently repairs it, and then the DOM doesn't match |
| **Stream with Suspense; don't block the page on the slowest query** | The shell is visible immediately |
| **`suppressHydrationWarning` only on a single unavoidable value** | Blanket use hides real bugs |
| **Prevent the theme flash with a blocking inline script** | React can't know the user's preference before it runs |
| **Remember the uncanny valley: visible ≠ interactive** | Make the pre-hydration state honest |

---

## 10. Interview traps

**Q1. "What is hydration?"**
Attaching React to server-rendered HTML: React walks the existing DOM and the component tree together, attaching event listeners and building the fiber tree **without creating DOM nodes**. It's matching, not rendering — which is why it's fast, and why a mismatch is a problem.

**Q2. "What causes a hydration mismatch?"**
Five causes: non-deterministic values (`Date.now()`, `Math.random()`), browser APIs during render, locale/timezone differences, invalid HTML nesting (the browser repairs it before React sees it), and extensions injecting into the DOM. **Locale/timezone is the one that passes on your machine and breaks in production.**

**Q3. "Why is a hydration mismatch more than a warning?"**
React recovers by re-rendering the mismatched subtree on the client — silent but expensive — and in the worst case the whole page falls back to client rendering. **You've paid for SSR and lost its benefit**, which shows up as a regressed LCP with no obvious cause.

**Q4. "SSR vs SSG vs CSR — how do you choose?"**
By constraint: does it need indexing or link previews? Is the content the same for everyone? Is it behind a login? Is first paint a business metric? Is nearly everything interactive? **Then give a concrete split** — marketing SSG, dashboard CSR, public receipt page SSR — because that shows you choose per surface rather than per product.

**Q5. "What's the cost of SSR that people don't mention?"**
The uncanny valley — the page looks interactive before it is. Plus servers to run and scale, every component needing to be SSR-safe, two environments to debug, hydration mismatches as a permanent bug class, and harder edge caching for personalised HTML.

**Q6. "What is streaming SSR?"**
The server sends the shell immediately and streams each Suspense boundary's content as it resolves, **out of order**, with an inline script that swaps it into place. One slow query no longer blocks the whole page.

**Q7. "What is selective hydration?"**
React hydrates in priority order, and **a user interaction promotes that subtree to the front of the queue** — click a not-yet-hydrated button and React hydrates it first, then replays the event. So a slow section doesn't block interactivity elsewhere. It's concurrent rendering applied to hydration, and most people don't know it exists.

**Q8. "How do you avoid a flash of the wrong theme?"**
A blocking inline script in `<head>` that reads `localStorage`/`prefers-color-scheme` and sets an attribute on `<html>` before paint, with CSS keyed off that attribute. React never renders theme-dependent markup, so there's nothing to mismatch. It's what `next-themes` does.

**Q9. "How do you make a component SSR-safe?"**
The §7 checklist: no browser APIs at module scope or in render, no non-deterministic values in output, explicit locale and timezone for formatting, `useId`, `useIsomorphicLayoutEffect`, valid HTML nesting, and `dynamic({ ssr: false })` for libraries that genuinely can't render server-side.

**Q10. "Your LCP got worse after adding SSR. What would you check?"**
Several things, in order: a hydration mismatch causing a full client re-render, TTFB (the server is now doing work per request), render-blocking resources, whether the LCP element is in the streamed shell or a suspended boundary, and whether you're shipping *more* JS than before (SSR doesn't reduce bundle size — only RSC does).

---

## 11. Build & break

### Build — cause every mismatch
Create an SSR page and deliberately trigger each of the five causes. **Read the exact error message for each** — they differ, and recognising them is how you diagnose quickly:
```tsx
<span>{new Date().toLocaleTimeString()}</span>                 // non-deterministic
<span>{window.innerWidth}</span>                                // browser API
<span>{new Intl.NumberFormat().format(4999)}</span>             // locale (run with TZ=Asia/Kolkata)
<p><div>invalid</div></p>                                        // nesting
```
Then fix each with the appropriate technique from §4.

### Build — watch streaming happen
A page with a fast header and two Suspense boundaries resolving at 200ms and 2000ms. Load it with the Network panel on "Response" and **watch the HTML arrive in chunks**, including the `$RC` scripts that swap content in. Then remove the boundaries and watch the entire page wait for the slowest query.

### Build — feel the uncanny valley
Throttle CPU to 6× and network to Slow 3G. Load an SSR page with a button and **click it as soon as it appears.** Nothing happens. Time the gap between visible and interactive. **That number is the cost of SSR**, and measuring it once changes how you think about shipping JavaScript.

### Build — the rendering-strategy document
Write Ledger's version of §8: each surface, its strategy, and the constraint that decided it. One page. **That document is the artefact an architecture interview is actually asking for.**

### Break — four experiments
1. **Mismatch cost.** Cause a mismatch high in the tree, then measure LCP with and without it. Watch React discard the server HTML.
2. **Blocking SSR.** Put a 3-second `await` before the return, with no Suspense. TTFB is 3 seconds and nothing is visible until then.
3. **`window` at module scope.** Reference it at the top of a module and watch the server crash — a clearer failure than the subtle ones.
4. **Theme flash.** Implement dark mode with `useEffect` only, reload a few times, and watch the white flash. Then add the inline script.

### Explain out loud (90 seconds)
1. What hydration is, and why it's matching rather than rendering.
2. Five causes of a mismatch, and why it's worse than a warning.
3. What streaming and selective hydration each buy you.
4. How you'd choose a rendering strategy for three different surfaces.
5. The costs of SSR that people skip.

---

## Module 7 complete — checkpoint

- [ ] What a Server Component is, mechanically
- [ ] RSC vs SSR, and why they're complementary
- [ ] What crosses the RSC boundary, and what can't
- [ ] The children pattern for keeping server content out of the bundle
- [ ] Why a Server Action needs the same authorization as a REST endpoint
- [ ] The four Next.js caches and how each is invalidated
- [ ] The stale-data debugging procedure
- [ ] Server Actions vs Route Handlers
- [ ] How RSC and a client query library compose
- [ ] What hydration is, and the five mismatch causes
- [ ] Streaming SSR and selective hydration
- [ ] Choosing a rendering strategy by constraint

---

## What's next

Everything so far has been technique. Module 8 is the build: **Ledger Console**, the dashboard that exercises all of it — and then how to test React so the whole thing stays correct.

Next → **[Lesson 26: Ledger Console blueprint](../08-project/26-console-blueprint.md)**
