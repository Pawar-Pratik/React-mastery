# Lesson 24 — Next.js App Router

> **Why this lesson exists:** Next.js is where most people actually meet RSC, and its **caching layers** are the single most confusing thing in modern React — four of them, with different scopes, different invalidation, and defaults that have changed in each of the last three major versions. Getting them wrong produces the two worst outcomes in web development: stale data users can't refresh away, and a page that never caches anything.

**Time:** ~90 minutes · **Prereq:** Lesson 23

---

## 1. The idea in one sentence

> **The App Router is file-based routing over RSC with four places a rendered value can be served from — and since Next 16, nothing reaches the persistent ones unless you mark it `"use cache"`. The skill is choosing what to cache, for how long and under which tag, and knowing which layer is stale when it goes wrong.**

---

## 2. File conventions

```
app/
├── layout.tsx              root layout — wraps everything, persists across navigation
├── page.tsx                /
├── loading.tsx             a Suspense fallback for this segment
├── error.tsx               an error boundary ("use client", receives {error, reset})
├── not-found.tsx           404 UI
├── global-error.tsx        catches errors in the root layout itself
├── template.tsx            like layout, but REMOUNTS on navigation
├── payments/
│   ├── layout.tsx          nested layout — persists while navigating within /payments
│   ├── page.tsx            /payments
│   ├── loading.tsx
│   └── [id]/
│       ├── page.tsx        /payments/:id
│       └── @modal/         a parallel route slot
├── (marketing)/            route GROUP — organisational, not in the URL
│   └── pricing/page.tsx    /pricing
└── api/webhooks/route.ts   a Route Handler (GET/POST exports)
```

| Convention | What it does |
|---|---|
| `[id]` | Dynamic segment |
| `[...slug]` | Catch-all |
| `[[...slug]]` | Optional catch-all |
| `(group)` | Organise without affecting the URL |
| `_folder` | Private — excluded from routing |
| `@slot` | Parallel route (render two pages at once) |

**`layout` vs `template`:** a layout **persists** across navigation within its segment — state survives, no remount. A template **remounts** every time. Use a layout by default; a template when you genuinely want a reset (an enter animation, or per-page state that must not leak).

**The layout gotcha worth knowing:** because layouts persist, a `useState` in a client layout survives navigation between sibling pages. That's usually what you want — and occasionally a surprising source of stale UI.

---

## 3. The four caches

**This is the part to get right, and the part that changed most in Next 16.** This section teaches Next 16 with **Cache Components** on, the model `create-next-app` generates from 16.4; the end of the section covers the model before it, which an upgraded app still runs.

### The mechanism: nothing persistent is cached unless you mark it
With Cache Components, a `fetch` or a database query runs **on every request** by default. To work out what can be served without running your code, `next build` renders each route once and every component lands in one of three places:

| What the component does | What the build does with it |
|---|---|
| No data, or data only inside `"use cache"` | Renders it into the **static shell** |
| Uncached data or request data (`cookies()`, `headers()`, `params`, `searchParams`) **inside `<Suspense>`** | Puts the fallback in the shell; the content **streams per request** |
| The same, with **no `<Suspense>` above it** | **Flags it** in the dev overlay and **fails** `next build`: the route would block |

That's **Partial Prerendering**: one response is a prerendered shell plus streamed holes. Next 15 asked "is this route static or dynamic?"; Next 16 asks "which *parts* are static?", and it refuses to guess. Uncached work that would block the page is an error until you cache it, stream it, or explicitly let the segment block (`export const instant = false`, 16.3+). `Date.now()`, `Math.random()` and `crypto.randomUUID()` count too: call `await connection()` before them inside a `<Suspense>`, or cache the result.

```
┌───────────────────────────────────────────────────────────────┐
│ 1. Request memoization     server, ONE render pass            │
│    Same GET fetch, or the same React cache() loader, runs     │
│    once per pass. Automatic. Gone when the request ends.      │
├───────────────────────────────────────────────────────────────┤
│ 2. "use cache" entries     server, ACROSS requests            │
│    Opt in per function/component. Key = build + function +    │
│    arguments. Lifetime: cacheLife. On-demand: cacheTag.       │
│    Default store: memory, per instance. Dies with the deploy. │
├───────────────────────────────────────────────────────────────┤
│ 3. Static shell (PPR)      build / revalidation time          │
│    Prerendered HTML + RSC payload: everything that completed  │
│    at build, plus Suspense fallbacks for the rest. Rebuilt    │
│    when a cache inside it revalidates, or on a new deploy.    │
├───────────────────────────────────────────────────────────────┤
│ 4. Client Cache            BROWSER, in-memory, per tab        │
│    RSC payloads for visited and prefetched routes. Fresh for  │
│    cacheLife's `stale` (min 30s). Cleared by revalidation in  │
│    a Server Action, router.refresh(), or a reload.            │
└───────────────────────────────────────────────────────────────┘
```

**The same four places as Next 15, with different controls.** 15 called layer 2 the Data Cache, layer 3 the Full Route Cache and layer 4 the Router Cache. What changed is *who decides*: 15 inferred caching from `fetch` options and route-level config; 16 caches what you mark with `"use cache"`, with its lifetime and tags written beside it. (Existing `force-cache` fetches and `unstable_cache` keep working, so a migration can be gradual.)

**Why this causes so much pain:** a mutation can update the database and the user still sees old data, because layer 2, 3 or 4 is serving a cached copy. The debugging question is always *"which layer is stale?"* — and the answer differs depending on whether the staleness survives a hard refresh (server-side) or not (client-side).

> **Version note that matters in interviews:** caching has changed in each of the last three majors. **Next 14** cached `fetch` and routes by default. **Next 15** made `fetch`, and the browser's reuse of visited pages, uncached by default, because the implicit caching surprised too many people. **Next 16** stabilised **Cache Components** (`"use cache"`, `cacheLife`, `cacheTag`, Partial Prerendering as the rendering model) behind `cacheComponents: true`, replacing the experimental `dynamicIO`, `useCache` and `ppr` flags. From **16.4**, `create-next-app` turns it on and the Next.js team recommends it for every app; **Next 17** will make it the default. So "what's the default?" is two questions: which version, and is `cacheComponents` on. Knowing that the answer changed three times, and *why*, is a strong freshness signal.

```tsx
// next.config.ts — create-next-app writes this from 16.4 on
const nextConfig: NextConfig = {
  cacheComponents: true,       // 16.0+: "use cache", cacheLife, cacheTag, PPR
  partialPrefetching: true,    // 16.3+: part of the model; leaving it unset logs a warning
};
```

```tsx
import { Suspense } from "react";
import { cacheLife, cacheTag } from "next/cache";

// Same for every merchant → cached, and part of the static shell
async function FeeSchedule() {
  "use cache";
  cacheLife("days");                            // stale 5m · revalidate 1d · expire 1w
  cacheTag("fee-schedule");
  return <FeeTable fees={await db.fees.current()} />;
}

// Per merchant → read the request OUTSIDE the cache, pass the value IN
async function RecentPaymentsSection() {
  const session = await auth();                 // reads cookies → request-time data
  return <RecentPayments merchantId={session.merchantId} />;
}

async function RecentPayments({ merchantId }: { merchantId: string }) {
  "use cache";                                  // merchantId is an argument → it's in the key
  cacheLife("minutes");
  cacheTag(`payments:${merchantId}`);
  return <PaymentsTable rows={await db.payments.recent(merchantId)} />;
}

export default function Page() {
  return (
    <>
      <FeeSchedule />                                   {/* static shell */}
      <Suspense fallback={<TableSkeleton />}>
        <RecentPaymentsSection />                       {/* streams; cached per merchant */}
      </Suspense>
      <Suspense fallback={<BalanceSkeleton />}>
        <LiveBalance />                                 {/* no directive: queried every request */}
      </Suspense>
    </>
  );
}
```

**Why `cookies()` is forbidden inside `"use cache"`.** An entry's key is the build ID, the function's identity, and its **serialized arguments and captured variables**. If `RecentPayments` could read the session itself, the merchant wouldn't be in the key, and merchant A's cached rows would be served to merchant B. So request APIs throw inside a cached scope, including in helpers it calls, and request data has to arrive as an argument, which puts it in the key. The flip side: the key is only as trustworthy as its arguments. Derive `merchantId` from the session on the server. A file-level `"use cache"` module imported into a Client Component becomes callable from the browser like a Server Function, and then that argument is client-controlled: [Lesson 23](23-rsc.md)'s BOLA problem again.

**Lifetimes are three numbers** (`cacheLife`):

| Field | Acts in | Meaning |
|---|---|---|
| `stale` | Browser | How long the Client Cache reuses the entry without asking the server (minimum 30s) |
| `revalidate` | Server | After this, a request still gets the cached value and triggers a background refresh (stale-while-revalidate) |
| `expire` | Server | After this with no traffic, the next request waits for a fresh value |

The presets are `seconds`, `minutes`, `hours`, `days`, `weeks` and `max`. With no `cacheLife` call the `default` profile applies (stale 5 min, revalidate 15 min, never expires), which is invisible at the call site, so always write one. A lifetime too short to be worth prerendering (`revalidate: 0`, or `expire` under 5 minutes; of the presets, only `seconds`) is left out of the shell and resolved per request, so it needs a `<Suspense>` boundary like uncached data.

**Where entries live is a production decision.** The default `"use cache"` store is **in-memory, per server instance**: on serverless it often doesn't survive between requests, and with several instances each holds its own copy. The build ID is in every key, so **each deploy starts cold** — unlike 15's `fetch` Data Cache, which survived deploys. `"use cache: remote"` stores entries in a shared cache handler (Redis, a platform KV), and with several instances that handler must also share tag invalidations: by default, an invalidation on instance A isn't propagated, and instance B keeps serving its stale copy.

### Invalidation
```tsx
"use server";
import { updateTag } from "next/cache";

export async function refundPayment(paymentId: string) {
  const session = await auth();                           // a public endpoint (Lesson 23)
  const payment = await db.payments.findById(paymentId, session.merchantId);
  if (!payment) return { error: "not_found" };
  await db.refunds.create({ paymentId: payment.id /* … */ });
  updateTag(`payments:${session.merchantId}`);            // ← expire now: the next read waits for fresh rows
}
```

```tsx
// app/api/webhooks/ledger/route.ts — not a Server Action, so updateTag isn't available
revalidateTag(`payments:${event.merchantId}`, "max");          // stale-while-revalidate
revalidateTag(`payments:${event.merchantId}`, { expire: 0 });  // or: gone now, the next read blocks
```

| Call | Where | What the next read gets |
|---|---|---|
| `updateTag(tag)` | Server Actions only | Fresh data; it waits. The action's response already carries the re-rendered page |
| `revalidateTag(tag, "max")` | Actions, Route Handlers | The **stale** value, while a refresh runs in the background. No re-render in the action's response |
| `revalidateTag(tag, { expire: 0 })` | Actions, Route Handlers | Fresh data; the webhook's equivalent of `updateTag` |
| `revalidatePath(path)` | Actions, Route Handlers | A fresh render of that route. Other routes reading the same data keep their copy |
| `refresh()` | Server Actions only | A re-render of the current route that touches no cache, for uncached data the action changed |
| `router.refresh()` | Client | The current route, re-requested. Server caches are untouched, so cached parts come back unchanged |

Called from a Server Action, any of the first five also **clears the whole Client Cache**. An action that calls none of them (and sets no cookie and doesn't redirect) leaves every layer alone.

**The Next 16 trap:** `revalidateTag(tag, "max")` in the refund action. The refund commits and the merchant still sees the payment as unrefunded: `revalidateTag` with `"max"` doesn't re-render in the action's response, and the next read is served the stale value *by design*. That's the bug `updateTag` exists for. **`updateTag` when the user must see their own write; `revalidateTag(tag, "max")` when eventual is fine or the caller isn't an action.** The 15-style one-argument `revalidateTag(tag)` is deprecated on 16: TypeScript rejects it, and at runtime it behaves like `{ expire: 0 }`.

**Tag-based invalidation is the one to prefer** — it's the idea behind a query-key factory ([Lesson 17](../05-data/17-server-state.md)): name the data, not the screens, so "what changed" is decoupled from "which routes display it."

### If you're on Next 15 (or on 16 without the flag)
An app upgraded to 16 keeps this model until it sets `cacheComponents`. Caching is configured on `fetch` and on the route:

```tsx
fetch(url)                                          // 15+: NOT cached (14 cached by default)
fetch(url, { cache: "force-cache" })                 // Data Cache: persists, survives deploys
fetch(url, { next: { revalidate: 60, tags: ["payments"] } })
export const dynamic = "force-dynamic";              // the whole route renders per request
export const revalidate = 60;                        // the whole route regenerates at most every 60s
// Non-fetch data: unstable_cache(fn, ["key"], { tags, revalidate })
```

| Previous model | Cache Components |
|---|---|
| Data Cache: `force-cache`, `next.revalidate`, `next.tags`, `unstable_cache` | `"use cache"` + `cacheLife` + `cacheTag` |
| Full Route Cache: a route is wholly static or wholly dynamic | A static shell per route; only what's inside `<Suspense>` is dynamic |
| One `cookies()` call silently makes the whole route dynamic | Request data outside `<Suspense>` is an error you have to resolve |
| `dynamic`, `revalidate`, `fetchCache` route config | Not allowed: exporting them errors |
| `noStore()`, `force-dynamic` | Not needed: uncached is the default |
| Router Cache | Client Cache, fresh per entry for `cacheLife`'s `stale` |

`updateTag` and the two-argument `revalidateTag` are Next 16 APIs and work in both models. On 15 itself you have `revalidateTag(tag)` and `revalidatePath`.

### The debugging procedure
```
Data is stale. Which layer?
1. Hard refresh (Cmd+Shift+R). Still stale?
   → NO  → layer 4, the Client Cache: you're inside its `stale` window.
           After your own mutation: did the action call updateTag, revalidatePath or refresh?
           Went Back to it? Cache Components restores recent routes from a hidden <Activity>
           with client state intact, so useState seeded from props keeps the old value.
   → YES → server-side. Continue:
2. Is the value produced inside "use cache" (or under a cached parent)?
   → layer 2. Its lifetime hasn't run out and its tag wasn't expired:
     · revalidateTag(tag, "max") serves stale once, by design → updateTag in actions
     · the tag you invalidate must match the cacheTag string exactly
     · several instances? the invalidation reached only the one that ran it
3. Is it baked into the shell? (`next build`: ○ static · ◐ partial prerender · ƒ dynamic)
   → layer 3. It changes when a cache inside it revalidates, or on a redeploy.
4. None of these? Next isn't caching it. Look upstream: a lagging read replica,
   or the client query cache (Lesson 17).

On 15, step 2 is the Data Cache (check cache / revalidate / tags on the fetch)
and step 3 is the Full Route Cache (○ static vs ƒ dynamic in the build output).
```
**Memorise that procedure.** It's the answer to the most common Next.js support question there is.

---

## 4. Data fetching patterns

```tsx
// Parallel — start both, then await. One round trip's worth of latency.
export default async function Page({ params }: Props) {
  const { id } = await params;
  const paymentPromise = getPayment(id);
  const refundsPromise = getRefunds(id);
  const [payment, refunds] = await Promise.all([paymentPromise, refundsPromise]);
  return <Detail payment={payment} refunds={refunds} />;
}

// ❌ Sequential — an accidental waterfall
const payment = await getPayment(id);
const refunds = await getRefunds(id);       // waits for the first, needlessly
```

```tsx
// Streaming — don't block the whole page on the slow part
export default async function Page({ params }: Props) {
  const { id } = await params;
  const payment = await getPayment(id);      // fast — render immediately

  return (
    <>
      <PaymentHeader payment={payment} />
      <Suspense fallback={<RefundsSkeleton />}>
        <RefundsList paymentId={id} />        {/* slow — streams in later */}
      </Suspense>
    </>
  );
}
```
**This is the pattern that makes RSC feel fast:** the shell and the fast data render immediately; slow sections stream in as they resolve ([Lesson 25](25-ssr-streaming-hydration.md)).

```tsx
// Preloading — start the fetch before you need it, avoiding a waterfall
export const preloadPayment = (id: string) => { void getPayment(id); };

export default async function Layout({ children, params }: Props) {
  preloadPayment((await params).id);         // kick it off; cache() dedupes the page's call
  return <>{children}</>;
}
```

```tsx
// React's cache() — dedupe a non-fetch data access across a render pass
import { cache } from "react";
export const getPayment = cache(async (id: string) => db.payments.findById(id));
// Calling it in a layout AND a page = one DB query.
```
**`cache()` is the DB equivalent of request memoization**, and it's what makes it safe to call the same loader in several components rather than prop-drilling data down.

---

## 5. Routing features worth knowing

```tsx
// Parallel routes — render two pages into one layout
// app/payments/layout.tsx
export default function Layout({ children, modal }: { children: ReactNode; modal: ReactNode }) {
  return <>{children}{modal}</>;
}
// app/payments/@modal/(.)[id]/page.tsx     ← intercepting route
```
**Intercepting routes** are the elegant version of [Lesson 16](../04-composition/16-routing-and-architecture.md)'s modal-route pattern: clicking a payment opens a modal over the list, but **opening the URL directly renders the full page.** `(.)` intercepts the same level, `(..)` one level up, `(...)` from the root.

```tsx
// Route Handlers — when you need an actual HTTP endpoint
// app/api/webhooks/ledger/route.ts
export async function POST(req: Request) {
  const raw = await req.text();                            // ← RAW body for HMAC (API L23)
  if (!verifySignature(req.headers.get("ledger-signature"), raw, SECRETS))
    return Response.json({ code: "invalid_signature" }, { status: 400 });

  const event = JSON.parse(raw);
  await queue.add("ledger-event", event);                   // ack fast, process async
  return Response.json({ received: true });
}
```
**Route Handlers are still the right tool** for webhooks, OAuth callbacks, file downloads, and any endpoint consumed by something other than your own UI. Server Actions don't replace them.

```tsx
// proxy.ts (Next.js 16; called middleware.ts before) — runs before every matched request
export function proxy(req: NextRequest) {
  const session = req.cookies.get("session");
  if (!session && req.nextUrl.pathname.startsWith("/payments")) {
    return NextResponse.redirect(new URL("/login", req.url));
  }
}
export const config = { matcher: ["/payments/:path*", "/settings/:path*"] };
```
> **Version note:** Next.js 16 renamed `middleware.ts` to **`proxy.ts`** (the exported function is `proxy`), and it runs on the **Node.js runtime**. Until Next 15.2, middleware could only run on the Edge runtime, which is why so many auth libraries couldn't use Node APIs in it. `middleware.ts` still works in 16 but is deprecated. The rename is the point: it's a proxy at the network boundary, not a place for application logic. Everything below applies under either name. Because it's an ordinary Node request path now, the [Node track's](https://github.com/Pawar-Pratik/Node-mastery) event-loop rules apply: anything slow or synchronous in it blocks every request.

**The proxy is for cheap, coarse decisions** — redirects, rewrites, setting headers, A/B bucketing. It runs on *every* matched request, so heavy work there is a latency tax on the whole site. And **it isn't your authorization layer**: the real check belongs in the Server Component or Action that touches data ([API Lesson 15](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/15-authorization-and-multitenancy.md)). That includes Server Actions: they're separate POST endpoints, so a matcher in the proxy doesn't protect them.

```tsx
// Metadata — static or generated
export const metadata: Metadata = { title: "Payments · Ledger" };

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const payment = await getPayment((await params).id);      // deduped with the page's fetch
  return { title: `Payment ${payment.id} · Ledger` };
}
```

---

## 6. Client/server composition in practice

```tsx
// app/payments/page.tsx — server
export default async function PaymentsPage({ searchParams }: Props) {
  const filters = FiltersSchema.parse(await searchParams);   // ← URL is a boundary. Parse it.
  const session = await auth();
  const initial = await db.payments.list({ ...filters, merchantId: session.merchantId });

  return (
    <PaymentsShell>                              {/* client — layout state */}
      <FilterBar />                               {/* client — inputs */}
      <Suspense fallback={<TableSkeleton />}>
        <PaymentsTable initialData={initial} filters={filters} />   {/* client — virtualized */}
      </Suspense>
    </PaymentsShell>
  );
}
```

**Hydrating a client cache from the server** — the pattern that makes RSC and TanStack Query work together instead of competing:

```tsx
// server
const qc = new QueryClient();
await qc.prefetchQuery({ queryKey: paymentKeys.list(filters), queryFn: () => listPayments(filters) });

return (
  <HydrationBoundary state={dehydrate(qc)}>
    <PaymentsTable />        {/* client: useQuery finds the data already in the cache */}
  </HydrationBoundary>
);
```
The first render has data with no loading state (server-fetched), and the client cache takes over for interaction, mutations, real-time and refetching. **That's the right architecture for an app that's both content-heavy and interactive**, and being able to describe it is a strong senior signal — it shows you don't see RSC and client caching as either/or.

---

## 7. Next.js decisions worth having opinions on

| Decision | Position |
|---|---|
| **App Router or Pages Router for a new app?** | App Router. Pages is maintained but not where the investment is |
| **Server Actions or Route Handlers?** | Actions for your own UI's mutations; Route Handlers for webhooks, OAuth callbacks, and anything a third party calls |
| **Do you still need a query library?** | **Yes**, for interactive apps — mutations with optimistic updates, real-time, polling, cross-component dedup. Prefetch on the server, hydrate the client cache |
| **Middleware for auth?** | For redirects only. Real authorization happens where data is accessed |
| **Cache Components on?** | Yes for a new app: it's `create-next-app`'s default from 16.4 and the default for everyone in 17. For an existing app, migrate route by route |
| **`updateTag`, `revalidateTag` or `revalidatePath`?** | Tags over paths — a tag decouples "what changed" from "which routes show it", like a query-key factory. Then `updateTag` in an action when the user must see their own write; `revalidateTag(tag, "max")` when eventual is fine or a webhook is calling |
| **Deploy target?** | Vercel is frictionless; self-hosting via `next start` or Docker works, but you own the cache: several instances need a shared cache handler that also shares tag invalidations |

**On vendor lock-in, honestly:** the App Router runs anywhere Node runs, but some features (ISR at scale, image optimisation, edge deployment) are materially easier on Vercel. That's a real consideration for an architecture decision, and pretending otherwise is naïve.

---

## 8. Ledger Console: would you use Next.js?

**The honest answer: probably not for the dashboard.** And this is a good interview position to hold:

> *"Ledger Console is an authenticated dashboard — there's nothing to index, no anonymous first paint to optimise, and almost every surface is interactive: a virtualized table, filters, modals, live updates. RSC's main wins are bundle reduction and server-side data access, and in an app where nearly everything is a client component, you get the ceremony without much of the benefit. I'd ship a Vite SPA with a query layer, and use Next.js for the **marketing site and docs** in front of it, where SSG and SEO genuinely matter."*

**Where Next.js *would* earn its place in this product:**
- The public marketing site and documentation (SSG, SEO, LCP)
- A public status page
- Hosted invoice/receipt pages customers open from an email (SSR, no login, must be fast)
- The reports page, if aggregation were heavy enough to want it server-side

**Choosing the framework by the constraint, rather than by default, is the skill this lesson is really teaching.**

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Know which of the four caches is serving your data** | The debugging question is always "which layer is stale?" |
| **Check the Next version, and whether `cacheComponents` is on** | 14 cached `fetch` by default, 15 didn't, 16 caches only what you mark: three answers to "why is this cached?" |
| **Write a `cacheLife` in every `"use cache"`** | The default profile is invisible at the call site, and an outer cache without one quietly takes a shorter inner lifetime |
| **Read request data outside `"use cache"`; pass it in as an argument** | Arguments are the cache key. That's what stops one merchant's cached rows from being served to another |
| **`updateTag` after a user's own write; `revalidateTag(tag, "max")` for webhooks and eventual updates** | Stale-while-revalidate shows the user their pre-mutation data on the next read |
| **Prefer tags over `revalidatePath`** | Decouples data identity from route structure |
| **More than one instance → a shared cache handler that also syncs tags** | The default store is per-instance memory, and an invalidation reaches only the instance that ran it |
| **Start independent fetches in parallel** | Sequential `await`s are an accidental waterfall |
| **Wrap slow sections in Suspense to stream** | The shell renders immediately; slow parts arrive later |
| **`cache()` your data loaders** | Call the same loader in layout and page; one query |
| **Parse `searchParams` with a schema** | The URL is a boundary and users edit it |
| **Server Actions authenticate, authorize and validate** | They're public HTTP endpoints |
| **The proxy (`proxy.ts`, formerly middleware) for redirects, not authorization** | It runs on every request and doesn't see your data layer |
| **Route Handlers for webhooks and third-party callbacks** | Actions don't replace them — and webhooks need the raw body |
| **Prefetch on the server, hydrate the client cache** | RSC and a query library complement rather than compete |
| **Layouts persist; use `template` when you need a remount** | Client state in a layout survives navigation |

---

## 10. Interview traps

**Q1. "Explain Next.js's caching."**
The weak answer recites the four cache names from the Next 14 docs. The strong one starts with the model: in Next 16 with Cache Components, **nothing persistent is cached unless you mark it `"use cache"`**. Uncached work runs per request, and the build prerenders a static shell around it. Then the four layers with scope and invalidation: request memoization (one render pass), `"use cache"` entries (keyed by arguments, lifetime from `cacheLife`, invalidated through `cacheTag`), the static shell (rebuilt when a cache inside it revalidates), and the browser's Client Cache (the `stale` window, cleared by revalidation in an action or by `router.refresh()`). **Then give the debugging procedure** — does a hard refresh fix it? That splits client from server immediately.
**Follow-up:** *"And before 16?"* The same four places under older names — Data Cache, Full Route Cache, Router Cache — but decided by inference from `fetch` options and route config, where one `cookies()` call made a whole route dynamic. Adding that an upgraded app stays on that model until it sets `cacheComponents` is the freshness signal.

**Q2. "A mutation succeeds but the page shows old data."**
Work the layers. A hard refresh fixes it → the Client Cache: the action called none of `updateTag`, `revalidateTag`, `revalidatePath` or `refresh`, so nothing cleared it. It survives a hard refresh → server-side, and on 16 the first suspect is **`revalidateTag(tag, "max")` in the action**: stale-while-revalidate serves the old value on the next read by design, so the user doesn't see their own write. `updateTag` is the read-your-own-writes call. Then check that the tag you invalidate matches the `cacheTag` string exactly, and whether you run several instances without a shared cache handler.
**Follow-up:** *"Why not just `updateTag` everywhere?"* Because it makes the next reader wait for a fresh render. For a webhook updating a page thousands of people read, serving stale for one request while it refreshes is the right trade, and `updateTag` isn't available outside actions anyway.

**Q3. "Server Actions or Route Handlers?"**
Actions for your own UI's mutations — colocated, typed, progressively enhanced, and an action that calls `updateTag` or `revalidatePath` returns the re-rendered page in the same response. Route Handlers for anything with an external caller: webhooks (which need the raw body for HMAC), OAuth callbacks, file downloads, public APIs.

**Q4. "Is Next.js middleware a good place for auth?"**
For redirecting unauthenticated users, yes — it's cheap and runs before rendering. **It is not your authorization layer**: it doesn't know which record is being accessed, and the real check belongs where data is read or written. Treating middleware as security is the same mistake as relying on a client-side route guard.
**Follow-up:** *"What changed in Next 16?"* It's `proxy.ts` now, and it runs on the Node.js runtime instead of being Edge-only. That makes Node libraries usable there, but it changes none of the reasoning above.

**Q5. "Do you still need TanStack Query with RSC?"**
For an interactive app, yes: mutations with optimistic updates, polling, real-time, cross-component deduplication and window-focus refetching all live on the client. The right pattern is **prefetch on the server, hydrate the client cache** — first render has data, and the client cache handles interaction.

**Q6. "What's an intercepting route?"**
A route that renders in a parallel slot when navigated to from within the app, but renders the full page when the URL is opened directly. `(.)` same level, `(..)` one up, `(...)` from root. It's the clean version of the "linkable modal over a list" pattern.

**Q7. "`layout` vs `template`?"**
Layouts persist across navigation within their segment — no remount, state survives. Templates remount every navigation. Default to layout; use a template when you need a reset (enter animations, per-page state that shouldn't leak between siblings).

**Q8. "Why has Next.js caching changed in every major since 14?"**
Each one fixed the previous one's surprise. **14** cached `fetch` and routes by default, so data went stale with no visible cause and the opt-out wasn't discoverable. **15** made `fetch` uncached by default, but caching was still inferred from `fetch` options, route config and whether anything read `cookies()`, and one stray call could silently make a whole route dynamic. **16** made it explicit and local: `"use cache"` where the work is, its lifetime and tags beside it, and an error instead of a silent fallback when uncached work would block the page. **The lesson generalises:** a default that's invisible at the call site, with non-obvious invalidation, is a bad default however good the performance story is, and a framework should refuse the ambiguous case rather than choose for you.

**Q9. "How do you avoid waterfalls in the App Router?"**
Start independent fetches in parallel rather than sequential `await`s, use `preload` patterns to kick off a fetch in the layout, `cache()` loaders so calling them in several places is one query, and wrap slow sections in Suspense so they don't block the rest of the page.

**Q10. "Would you use Next.js for this app?"**
Answer by constraint, not default. Authenticated, highly-interactive dashboard → a SPA with a query layer is simpler and gets most of the benefit. Public, content-heavy, SEO-sensitive → Next.js. Mixed product → both, on different subdomains. **Being able to decline the popular choice with reasoning is the signal.**

---

## 11. Build & break

### Build — the cache lab
Write a ten-line Node server on port 4000 that returns `{ now, hit }` and logs every hit ([Node Lesson 17](https://github.com/Pawar-Pratik/Node-mastery/blob/main/04-networking/17-http-from-scratch.md) is all it takes). Create the app with `create-next-app` (16.4+ has Cache Components on; on 16.0–16.3 add the config from §3) and observe through `next build && next start`, because dev mode re-renders more than production does. Then observe each layer:
1. **Uncached by default.** A component that fetches the clock, inside `<Suspense>` → every reload shows a new time and logs one hit
2. **Remove the `<Suspense>`** → `next build` fails with the blocking-route error, and `next dev` shows the same problem in its overlay. That's Cache Components refusing to guess
3. **Put the `<Suspense>` back and fetch the same URL twice in that render** → still one hit per request (request memoization)
4. **Move the fetch into a `"use cache"` function with `cacheLife("minutes")`** → the clock logs a hit *during `next build`* (the prerender filled the cache), the build table shows the route as `○` with its revalidate time, and reloads are frozen (layers 2 and 3). After a minute, the next reload still shows the old time and triggers a hit; the one after shows the new time. That's stale-while-revalidate
5. **Add `cacheTag("clock")` and a button whose Server Action calls `revalidateTag("clock", "max")`** → after the click the page still shows the old time, and so does the next reload; the reload after that shows the new one. Switch the action to `updateTag("clock")` → the click shows the new time at once. **That's §3's Next 16 trap, felt once**
6. **Link to another page and back**, with the Network panel open → note what was requested and whether the cached clock changed (layer 4). Then call `router.refresh()` → the route is re-requested, but the cached clock comes back unchanged, because `router.refresh()` doesn't touch server caches

**Write down which action cleared which layer.** That table is the debugging procedure, learned by doing. On 15 (or 16 with the flag off), redo steps 3–5 with `fetch` options instead: `cache: "force-cache"` freezes it, `next: { revalidate: 60 }` gives the same stale-while-revalidate, and `next: { tags: ["clock"] }` makes it taggable.

### Build — streaming
Build a page with fast data and a deliberately slow section (`await sleep(3000)`). Load it with the Network panel open on "Response" and **watch the HTML arrive in chunks** — the shell first, the slow section three seconds later. Then remove the Suspense boundary and watch the whole page wait.

### Build — the hybrid
Prefetch a query on the server, wrap in `HydrationBoundary`, and consume it with `useQuery` in a client component. Verify: no loading state on first render, and a mutation still updates the cache optimistically. **That's the architecture for a real app.**

### Build — attack a Server Action
As in [Lesson 23](23-rsc.md): call your action directly with another tenant's ID. Without the session check it works. **Do it once.**

### Break — four experiments
1. **Sequential awaits.** Two independent fetches, sequential vs `Promise.all`. Compare TTFB.
2. **Accidental client bundle.** Add `"use client"` to a layout and check the bundle analyzer.
3. **Stale after your own write.** Invalidate the refund with `revalidateTag(tag, "max")` and watch the merchant's own refund not appear; switch to `updateTag` and it does. Then drop the invalidation entirely and watch the old data survive a soft navigation.
4. **Proxy cost.** Do something heavy in `proxy.ts` (or `middleware.ts` before Next 16) and measure TTFB across the site — it's a tax on every request.

### Explain out loud (90 seconds)
1. The four caches, their scope, how each is invalidated, and why a refund needs `updateTag` rather than `revalidateTag(tag, "max")`.
2. The stale-data debugging procedure.
3. Server Actions vs Route Handlers.
4. How RSC and a client query library work together.
5. When you'd choose *not* to use Next.js.

---

## What's next

You've seen RSC and the framework around it. The last piece of Module 7 is the rendering mechanics underneath both: SSR, streaming, and hydration — including the hydration mismatch, which is the most confusing error in server-rendered React.

Next → **[Lesson 25: SSR, streaming & hydration](25-ssr-streaming-hydration.md)**
