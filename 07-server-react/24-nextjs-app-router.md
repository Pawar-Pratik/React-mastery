# Lesson 24 — Next.js App Router

> **Why this lesson exists:** Next.js is where most people actually meet RSC, and its **caching layers** are the single most confusing thing in modern React — four of them, with different defaults, different invalidation, and behaviour that has changed between versions. Getting them wrong produces the two worst outcomes in web development: stale data users can't refresh away, and a page that never caches anything.

**Time:** ~75 minutes · **Prereq:** Lesson 23

---

## 1. The idea in one sentence

> **The App Router is file-based routing over RSC, with a caching system that is aggressive by default — so the skill is knowing which layer is serving your stale data and how to opt out of it deliberately.**

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

**This is the part to get right.** Four layers, each with its own scope and invalidation.

```
┌───────────────────────────────────────────────────────────────┐
│ 1. Request Memoization      server, per-request                │
│    Deduplicates identical fetch() calls within ONE render pass │
│    Auto. Cleared after the request. You rarely think about it. │
├───────────────────────────────────────────────────────────────┤
│ 2. Data Cache               server, PERSISTENT across requests │
│    Caches fetch() results. Survives deploys unless purged.     │
│    Opt in per fetch; invalidate with revalidateTag/Path.       │
├───────────────────────────────────────────────────────────────┤
│ 3. Full Route Cache         server, build/request time         │
│    Caches the rendered RSC payload + HTML for static routes.   │
│    Invalidated by revalidation or a new deploy.                │
├───────────────────────────────────────────────────────────────┤
│ 4. Router Cache             CLIENT, in-memory, per session     │
│    Caches RSC payloads for visited routes so Back is instant.  │
│    Cleared by router.refresh(), a Server Action, or a reload.  │
└───────────────────────────────────────────────────────────────┘
```

**Why this causes so much pain:** a mutation can update the database and the user still sees old data, because layer 2 or 4 is serving a cached copy. The debugging question is always *"which layer is stale?"* — and the answer differs depending on whether the staleness survives a hard refresh (server-side) or not (client-side).

```tsx
// Layer 2 — the Data Cache, controlled per fetch
fetch(url)                                          // Next 15: NOT cached by default
fetch(url, { cache: "force-cache" })                 // opt IN to caching
fetch(url, { cache: "no-store" })                    // never cache
fetch(url, { next: { revalidate: 60 } })             // cache for 60s (ISR)
fetch(url, { next: { tags: ["payments"] } })         // tag it for targeted invalidation
```

> **Version note that matters in interviews:** Next.js 14 cached `fetch` by default; **Next.js 15 flipped it to uncached by default** because the aggressive default surprised too many people. Knowing the default changed — and *why* — is a strong freshness signal. Always check your version rather than assuming.

```tsx
// Layer 3 — route-level opt-outs
export const dynamic = "force-dynamic";      // render on every request
export const revalidate = 60;                 // ISR: regenerate at most every 60s
export const fetchCache = "default-no-store";
```

```tsx
// Invalidation, after a mutation
"use server";
export async function refundPayment(id: string) {
  await db.refunds.create(...);
  revalidateTag("payments");                  // ← targeted: everything tagged "payments"
  revalidatePath(`/payments/${id}`);           // ← path-based: this route's cache
}
// A Server Action also clears the CLIENT router cache automatically.
```

**Tag-based invalidation is the one to prefer** — it's the same hierarchical idea as a query-key factory ([Lesson 17](../05-data/17-server-state.md)), and it decouples "what data changed" from "which routes display it."

### The debugging procedure
```
Data is stale. Which layer?
1. Hard refresh (Cmd+Shift+R). Still stale?
   → NO  → it's the client Router Cache. router.refresh(), or a Server Action.
   → YES → it's server-side. Continue:
2. Is the route static or dynamic? (`next build` prints ○ static / ƒ dynamic)
   → static → Full Route Cache. Add revalidate, force-dynamic, or revalidatePath.
3. Is the fetch cached? Check cache/revalidate/tags options.
   → Data Cache. revalidateTag, or no-store.
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

export default async function Layout({ params }: Props) {
  preloadPayment(params.id);                 // kick it off; request memoization dedupes
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
// Middleware — runs on every matched request, at the edge
export function middleware(req: NextRequest) {
  const session = req.cookies.get("session");
  if (!session && req.nextUrl.pathname.startsWith("/payments")) {
    return NextResponse.redirect(new URL("/login", req.url));
  }
}
export const config = { matcher: ["/payments/:path*", "/settings/:path*"] };
```
**Middleware is for cheap, coarse decisions** — redirects, rewrites, setting headers, A/B bucketing. It runs on *every* matched request, so heavy work there is a latency tax on the whole site. And **it isn't your authorization layer**: the real check belongs in the Server Component or Action that touches data ([API Lesson 15](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/15-authorization-and-multitenancy.md)).

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
| **`revalidatePath` or `revalidateTag`?** | Tag — it decouples "what changed" from "which routes show it", like a query-key factory |
| **Deploy target?** | Vercel is frictionless; self-hosting via `next start` or Docker works but you own the caching/ISR infrastructure |

**On vendor lock-in, honestly:** the App Router runs anywhere Node runs, but some features (ISR at scale, image optimisation, edge middleware) are materially easier on Vercel. That's a real consideration for an architecture decision, and pretending otherwise is naïve.

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
| **Check your Next version's `fetch` default** | 14 cached by default; 15 doesn't. Don't assume |
| **Prefer `revalidateTag` over `revalidatePath`** | Decouples data identity from route structure |
| **Start independent fetches in parallel** | Sequential `await`s are an accidental waterfall |
| **Wrap slow sections in Suspense to stream** | The shell renders immediately; slow parts arrive later |
| **`cache()` your data loaders** | Call the same loader in layout and page; one query |
| **Parse `searchParams` with a schema** | The URL is a boundary and users edit it |
| **Server Actions authenticate, authorize and validate** | They're public HTTP endpoints |
| **Middleware for redirects, not authorization** | It runs on every request and doesn't see your data layer |
| **Route Handlers for webhooks and third-party callbacks** | Actions don't replace them — and webhooks need the raw body |
| **Prefetch on the server, hydrate the client cache** | RSC and a query library complement rather than compete |
| **Layouts persist; use `template` when you need a remount** | Client state in a layout survives navigation |

---

## 10. Interview traps

**Q1. "Explain Next.js's caching."**
Name all four layers with scope and invalidation: request memoization (per-request dedup), the Data Cache (persistent, per-fetch, tag/path invalidation), the Full Route Cache (rendered output for static routes), and the client Router Cache (visited routes, cleared by `router.refresh()` or a Server Action). **Then give the debugging procedure** — does a hard refresh fix it? That splits client from server immediately.

**Q2. "A mutation succeeds but the page shows old data."**
Work the layers: hard refresh → if it fixes it, the client Router Cache (call `router.refresh()`, or use a Server Action which clears it). If not, it's server-side: check whether the route is static (Full Route Cache) and whether the fetch is cached (Data Cache), then `revalidateTag`/`revalidatePath`.

**Q3. "Server Actions or Route Handlers?"**
Actions for your own UI's mutations — colocated, typed, progressively enhanced, and they clear the router cache. Route Handlers for anything with an external caller: webhooks (which need the raw body for HMAC), OAuth callbacks, file downloads, public APIs.

**Q4. "Is Next.js middleware a good place for auth?"**
For redirecting unauthenticated users, yes — it's cheap and runs before rendering. **It is not your authorization layer**: it doesn't know which record is being accessed, and the real check belongs where data is read or written. Treating middleware as security is the same mistake as relying on a client-side route guard.

**Q5. "Do you still need TanStack Query with RSC?"**
For an interactive app, yes: mutations with optimistic updates, polling, real-time, cross-component deduplication and window-focus refetching all live on the client. The right pattern is **prefetch on the server, hydrate the client cache** — first render has data, and the client cache handles interaction.

**Q6. "What's an intercepting route?"**
A route that renders in a parallel slot when navigated to from within the app, but renders the full page when the URL is opened directly. `(.)` same level, `(..)` one up, `(...)` from root. It's the clean version of the "linkable modal over a list" pattern.

**Q7. "`layout` vs `template`?"**
Layouts persist across navigation within their segment — no remount, state survives. Templates remount every navigation. Default to layout; use a template when you need a reset (enter animations, per-page state that shouldn't leak between siblings).

**Q8. "Why did `fetch` caching change in Next 15?"**
Because caching-by-default surprised too many developers — data appeared stale with no obvious cause, and the opt-out wasn't discoverable. **The lesson generalises:** an aggressive default with non-obvious invalidation is a bad default, however good the performance story is.

**Q9. "How do you avoid waterfalls in the App Router?"**
Start independent fetches in parallel rather than sequential `await`s, use `preload` patterns to kick off a fetch in the layout, `cache()` loaders so calling them in several places is one query, and wrap slow sections in Suspense so they don't block the rest of the page.

**Q10. "Would you use Next.js for this app?"**
Answer by constraint, not default. Authenticated, highly-interactive dashboard → a SPA with a query layer is simpler and gets most of the benefit. Public, content-heavy, SEO-sensitive → Next.js. Mixed product → both, on different subdomains. **Being able to decline the popular choice with reasoning is the signal.**

---

## 11. Build & break

### Build — the cache lab
Create an App Router project with a page that fetches from an endpoint returning a timestamp. Then observe each layer:
1. Call the same `fetch` twice in one render → **one** network request (request memoization)
2. `{ cache: "force-cache" }` → reload; the timestamp is frozen (Data Cache)
3. `{ next: { revalidate: 10 } }` → it updates every 10 seconds (ISR)
4. Navigate away and back → instant, unchanged (Router Cache)
5. `router.refresh()` → the client cache clears
6. A Server Action calling `revalidateTag` → both server and client caches clear

**Write down which action cleared which layer.** That table is the debugging procedure, learned by doing.

### Build — streaming
Build a page with fast data and a deliberately slow section (`await sleep(3000)`). Load it with the Network panel open on "Response" and **watch the HTML arrive in chunks** — the shell first, the slow section three seconds later. Then remove the Suspense boundary and watch the whole page wait.

### Build — the hybrid
Prefetch a query on the server, wrap in `HydrationBoundary`, and consume it with `useQuery` in a client component. Verify: no loading state on first render, and a mutation still updates the cache optimistically. **That's the architecture for a real app.**

### Build — attack a Server Action
As in [Lesson 23](23-rsc.md): call your action directly with another tenant's ID. Without the session check it works. **Do it once.**

### Break — four experiments
1. **Sequential awaits.** Two independent fetches, sequential vs `Promise.all`. Compare TTFB.
2. **Accidental client bundle.** Add `"use client"` to a layout and check the bundle analyzer.
3. **Stale after mutation.** Mutate without `revalidateTag` and watch the old data persist through a soft navigation.
4. **Middleware cost.** Do something heavy in middleware and measure TTFB across the site — it's a tax on every request.

### Explain out loud (90 seconds)
1. The four caches, their scope, and how each is invalidated.
2. The stale-data debugging procedure.
3. Server Actions vs Route Handlers.
4. How RSC and a client query library work together.
5. When you'd choose *not* to use Next.js.

---

## What's next

You've seen RSC and the framework around it. The last piece of Module 7 is the rendering mechanics underneath both: SSR, streaming, and hydration — including the hydration mismatch, which is the most confusing error in server-rendered React.

Next → **[Lesson 25: SSR, streaming & hydration](25-ssr-streaming-hydration.md)**
