# Lesson 26 — Ledger Console blueprint

> **Why this lesson exists:** twenty-five lessons of technique need somewhere to land. **Ledger Console** is the merchant dashboard for the [Ledger API](../../API/README.md), typed with the [TypeScript track's](../../TypeScript/README.md) shared layer. Every feature exists because it forces a concept you've learned — and the result is one hosted product you can demo, which turns every interview answer from *"I think you'd probably…"* into *"here's how I handled that."*

**Time:** ~50 minutes to read and internalise · **Prereq:** Modules 1–7

---

## 1. What it is

A merchant-facing dashboard: view and filter payments at scale, issue refunds, manage webhook endpoints and API keys, watch a live event feed, and read revenue reports.

```
┌──────────────────────────────────────────────────────────────┐
│  Ledger Console (React + TS, Vite SPA)                        │
│  ├── Payments   virtualized table over 4M rows, URL filters   │
│  ├── Detail     panel over the list, linkable, prefetched     │
│  ├── Refunds    state machine + idempotency + optimistic list │
│  ├── Webhooks   endpoints, delivery log, retry                │
│  ├── API keys   create (show once), revoke                    │
│  ├── Reports    lazy-loaded chart bundle                      │
│  └── Live feed  one SSE connection → the query cache          │
└───────────────────────────┬──────────────────────────────────┘
                            │  typed client generated from openapi.yaml
┌───────────────────────────▼──────────────────────────────────┐
│  Ledger API  (the API track's project)                        │
└──────────────────────────────────────────────────────────────┘
```

**Why a payments dashboard and not a to-do app:** every hard React concept appears for a *real* reason, not because a tutorial needed one.

| The product forces… | …which teaches |
|---|---|
| 4 million payments per merchant | **Virtualization** + cursor pagination ([L21](../06-performance/21-render-performance.md), [API L08](../../API/02-rest-design/08-collections-and-pagination.md)) |
| Search-as-you-type over that table | **Transitions, deferred values, race handling** ([L11](../03-hooks/11-concurrent-hooks.md), [L08](../02-state-effects/08-effects.md)) |
| Filters you can send to a colleague | **URL as state** ([L16](../04-composition/16-routing-and-architecture.md)) |
| Refunds must never double-fire | **State machine + idempotency key** ([L06](../02-state-effects/06-reducers-and-state-machines.md), [API L18](../../API/04-production/18-reliability-and-idempotency.md)) |
| Refunds must feel instant, and roll back | **Optimistic updates** ([L18](../05-data/18-mutations-and-optimistic.md)) |
| Payments arriving while you watch | **SSE → cache integration** ([L19](../05-data/19-realtime.md)) |
| A chart that must not block the table | **Suspense boundaries + code splitting** ([L15](../04-composition/15-boundaries-suspense-portals.md), [L22](../06-performance/22-loading-performance.md)) |
| A broken chart must not hide payments | **Error boundary placement** ([L15](../04-composition/15-boundaries-suspense-portals.md)) |
| Tokens expiring mid-session | **Refresh-and-retry** ([API L13](../../API/03-security/13-sessions-and-jwt.md)) |
| Keyboard and screen-reader users | **A11y in a virtualized grid** ([L21](../06-performance/21-render-performance.md), [L14](../04-composition/14-forms.md)) |

---

## 2. Routes

| Route | Loader | Notes |
|---|---|---|
| `/login` | — | Public. `useActionState` form |
| `/payments` | list loader | Filters, sort, cursor in the URL |
| `/payments/:id` | detail loader | Panel over the table; list stays mounted |
| `/payments/:id/refund` | — | Modal route — linkable, refresh-proof |
| `/webhooks` | endpoints loader | |
| `/webhooks/:id/deliveries` | deliveries loader | Attempt history, manual retry |
| `/settings/api-keys` | — | Lazy chunk. Key shown **once** |
| `/reports` | — | Lazy chunk (~180KB of charting) |

```
/payments?status=succeeded&status=refunded&created[gte]=2026-01-01&sort=-amount&cursor=eyJ...
```
**Everything in that URL is state** — shareable, bookmarkable, back/forward-navigable, refresh-proof, and quotable in a support ticket.

---

## 3. The state model

The decision tree from [Lesson 05](../02-state-effects/05-state-and-snapshots.md), applied:

| State | Where | Why |
|---|---|---|
| Filters, sort, cursor, active tab | **URL** | Shareable; back/forward; no parent cascade |
| Payments, customers, balance, webhooks | **Query cache** | Server data: caching, dedup, revalidation |
| Auth session | **Context** (+ refresh token in an `HttpOnly` cookie) | Changes on login/logout only |
| Toast API `{show, dismiss}` | **Context** (stable, memoized `[]`) | Consumers never re-render |
| Toast list | **Separate context** | Only `<ToastViewport>` reads it |
| Refund flow | **`useReducer` state machine** | Illegal transitions must be impossible |
| Search draft (pre-debounce) | **Local `useState`** | Needed during typing |
| Row selection | **Local `useState`** in the table | Nothing above cares |
| Column widths, sidebar collapsed | **`localStorage`** (schema-validated) | Preferences, not view state |
| SSE connection, idempotency keys, timers | **Refs** | Never rendered |

**The three decisions worth defending in a review:**
1. **Filters in the URL, not a store** — it removes the re-render cascade *and* makes links shareable. One decision, two wins.
2. **Split toast contexts** — every component that shows a toast consumes only the stable API and never re-renders when the toast list changes.
3. **Terminal payments are `staleTime: Infinity`** — a `succeeded` payment is immutable, so caching it forever is *correct*, not merely fast.

---

## 4. Architecture

```
src/
├── routes/
│   ├── router.tsx              route tree, lazy imports, error elements
│   └── guards.tsx              ProtectedLayout (state.from), RequireScope
├── features/
│   ├── payments/  components/ hooks/ api.ts schemas.ts index.ts
│   ├── refunds/                refund-reducer.ts, refund-modal.tsx
│   ├── webhooks/
│   ├── api-keys/
│   └── auth/
├── components/ui/              button badge input skeleton
├── components/patterns/        modal data-table page async-boundary toast
├── hooks/                      use-debounced-value use-filters use-event-callback
├── lib/
│   ├── api-client.ts           typed fetch, generated from openapi.yaml
│   ├── query-client.ts         defaults: staleTime, retry-on-retryable, throwOnError
│   ├── realtime.tsx            ONE EventSource → cache updates
│   └── format.ts               hoisted Intl formatters
└── types/                      the TypeScript track's shared layer
```

**Dependencies flow downward only**, enforced by a lint rule ([Lesson 16](../04-composition/16-routing-and-architecture.md)): `components/ui` must never import from `features/`, and features import each other only through `index.ts`.

```tsx
// What a page looks like when the layers are right — thin, composed, no business logic
export function PaymentsPage() {
  const { filters, setFilter } = useFilters();          // URL
  const deferred = useDeferredValue(filters);           // keep typing responsive
  const query = usePayments(deferred);                  // query cache

  return (
    <Page title="Payments" actions={<ExportButton filters={filters} />}>
      <FilterBar value={filters} onChange={setFilter} />
      <ErrorBoundary fallback={(e, reset) => <TableError error={e} onRetry={reset} />}>
        <AsyncBoundary state={query} stale={filters !== deferred}>
          {data => <VirtualPaymentsTable pages={data.pages} />}
        </AsyncBoundary>
      </ErrorBoundary>
    </Page>
  );
}
```

---

## 5. The five hard parts

These are the features that make the project worth building. **Each is an interview answer.**

### 1. The payments table — 4M rows at 60fps
```
Server-side filter/sort/paginate     → the client holds ~100 rows, never 4M
useInfiniteQuery + cursor            → fetch the next page 10 rows from the end
Virtualized, ~20 rows in the DOM     → keyed by payment ID, positioned with transform
memo'd rows with stable props        → scrolling doesn't re-render unchanged rows
keepPreviousData                     → no skeleton flash when filters change
role="grid" + aria-rowcount          → screen readers hear 4,000,000, not 20
Keyboard nav with scrollToIndex      → arrows work past the rendered window
```
**Lead with the server.** "Virtualize it" is a good answer; "the client shouldn't have 4 million rows" is the right one.

### 2. Search-as-you-type without lag
```tsx
const [draft, setDraft] = useState("");                      // urgent — the input
const debounced = useDebouncedValue(draft, 300);              // fewer requests
useEffect(() => setSearchParams(p => …, { replace: true }), [debounced]);   // URL
const deferred = useDeferredValue(filters);                   // non-blocking render
const query = usePayments(deferred);                          // cancels superseded requests
```
**Four mechanisms, each doing a different job:** local state so the input is instant, debounce to reduce *requests*, `replace` so the back button isn't polluted, and deferral so the *render* never blocks. Knowing why you need all four — and that debouncing and deferring solve different problems — is the substance.

### 3. The refund flow
```
State machine (Lesson 06)      → `submitting` has no `submit` transition. Double-submit impossible
Idempotency key in a ref        → stable across retries, reset per operation
NOT optimistic                  → money appearing then vanishing breaks trust
Error codes branch              → already_refunded ⇒ we were stale ⇒ invalidate + refetch
Modal route                     → /payments/:id/refund is linkable and refresh-proof
Focus trap + Escape + restore    → via <dialog> or Radix
```

### 4. The live feed
```
ONE EventSource at the root     → not one per component
useEventCallback handler         → connects once, always calls the latest logic
Writes to the query cache        → no page component knows the stream exists
Version guard on apply           → out-of-order events are discarded
Dedupe + bound the list          → at-least-once delivery guarantees duplicates
Invalidate on reconnect          → the cache may have drifted
Backoff with full jitter         → a server restart doesn't cause a reconnect stampede
aria-live connection banner      → silence reads as "broken"
```

### 5. Auth with mid-request token refresh
```tsx
// Access token in memory, refresh token in an HttpOnly cookie (API Lesson 13)
// A 401 triggers ONE refresh; concurrent requests wait on the same promise.
let refreshing: Promise<string> | null = null;

async function withAuth<T>(req: () => Promise<Response>): Promise<T> {
  let res = await req();
  if (res.status === 401) {
    refreshing ??= refreshAccessToken().finally(() => { refreshing = null; });  // single-flight
    await refreshing;
    res = await req();                                    // retry once
  }
  if (!res.ok) throw await toApiError(res);
  return ResponseSchema.parse(await res.json());          // parse, don't assume
}
```
**The single-flight refresh is the detail that matters:** without it, five concurrent 401s trigger five refreshes, and with rotating refresh tokens ([API L13](../../API/03-security/13-sessions-and-jwt.md)) four of them are *reuse detections* that log the user out. That bug is subtle, common, and a great thing to have encountered.

---

## 6. Non-negotiables

```
ACCESSIBILITY
[ ] Every interaction reachable by keyboard; visible focus rings
[ ] Virtualized grid: role="grid", aria-rowcount, aria-rowindex, arrow-key nav
[ ] Forms: label htmlFor, aria-describedby, aria-invalid, role="alert"
[ ] Modals: focus trap, focus restore, Escape, inert background
[ ] Live regions for toasts and connection status
[ ] Tested with a real screen reader at least once

PERFORMANCE
[ ] Initial JS < 200KB gz · LCP < 2.5s · INP < 200ms · CLS < 0.1 (p75, throttled)
[ ] Route-level code splitting with hover prefetch
[ ] Reports chart lazy-loaded
[ ] Measured baselines recorded before and after optimisation

CORRECTNESS
[ ] Every API response parsed with a schema
[ ] Every money-moving mutation carries an idempotency key
[ ] Exhaustive switches with assertNever on every domain union
[ ] Error boundaries at root, route, and per independent widget
[ ] No `any`, no unexplained `as`, no unexplained `!`
```

---

## 7. The decisions, with justifications

**This table is your interview answer.** Every row is a choice with a reason, not a default.

| Decision | Choice | Why |
|---|---|---|
| Framework | **Vite SPA, not Next.js** | Authenticated dashboard: nothing to index, no anonymous first paint, nearly everything interactive. RSC would be ceremony without the benefit ([L23](../07-server-react/23-rsc.md)) |
| Routing | React Router 7, data mode | Loaders kill the nested-route fetch waterfall ([L16](../04-composition/16-routing-and-architecture.md)) |
| Server data | TanStack Query | Caching, dedup, revalidation, retry, race handling — none of which `useState` has ([L17](../05-data/17-server-state.md)) |
| Filter state | **URL** | Shareable + back/forward + no cascade |
| Client state | Local, lifted, or a small store | Context only for rarely-changing values ([L09](../03-hooks/09-context.md)) |
| Table | TanStack Virtual | 4M rows ([L21](../06-performance/21-render-performance.md)) |
| Forms | RHF + Zod; Actions for simple ones | Uncontrolled = no re-render per keystroke ([L14](../04-composition/14-forms.md)) |
| Components | Radix primitives, styled in-house | Focus management is weeks of work to get right ([L13](../04-composition/13-composition-patterns.md)) |
| Types | Generated from `openapi.yaml` | A backend contract change becomes a **compile error** ([API L25](../../API/06-craft/25-openapi-contract-first.md)) |
| Validation | Zod at every boundary | Types are erased; the API can lie ([TS L15](../../TypeScript/04-real-code/15-the-boundary-and-parsing.md)) |
| Real-time | SSE, one connection | One-way, auto-reconnect with resume ([L19](../05-data/19-realtime.md)) |
| Auth | Access token in memory + `HttpOnly` refresh cookie | XSS can't read it; CSRF blunted by `SameSite` ([API L13](../../API/03-security/13-sessions-and-jwt.md)) |
| Testing | RTL + MSW + Playwright | Behaviour, not implementation ([L28](28-testing-react.md)) |

### The five things you'll be able to say in an interview

1. *"I render 4 million payments at 60fps — but the first thing I'd say is the client never holds 4 million. The API does the filtering and cursor pagination; virtualization is about rendering the hundred I do hold."*
2. *"Search-as-you-type uses four mechanisms: local state for the input, a debounce to cut requests, `replace` on the URL so the back button isn't polluted, and `useDeferredValue` so rendering never blocks. Debouncing and deferring solve different problems."*
3. *"Refunds can't double-fire: the state machine has no `submit` transition from `submitting`, and there's an idempotency key on the request — because the network can duplicate it even when the UI can't."*
4. *"Real-time updates the query cache, not component state. No page component knows the SSE stream exists — which means adding a live feature didn't change a single page."*
5. *"Token refresh is single-flight. Without it, five concurrent 401s trigger five refreshes, and with rotating refresh tokens four of them are reuse detections that log the user out."*

**Each is mechanism + trade-off + evidence.** That's what "senior" sounds like.

---

## 8. What's deliberately out of scope

Knowing what you *didn't* build, and why, is as strong a signal as what you did.

| Not built | Why |
|---|---|
| A design system from scratch | Radix + a small token layer. Building an accessible combobox is weeks that teach little |
| i18n | Real work, orthogonal to React depth |
| Offline-first / PWA | Interesting, and a whole second project |
| Charts from scratch | Recharts. The interesting part is lazy-loading it, not drawing SVG |
| Micro-frontends | Solves an org problem you don't have |

**If asked "what would you add next?": offline support with a service worker and a persisted query cache** — it would force cache-versioning, conflict resolution and background sync, and it builds on everything already there.

---

## 9. Definition of done

```
FUNCTIONAL
[ ] Login with mid-session token refresh (single-flight)
[ ] Payments: virtualized, filtered, sorted, cursor-paginated, URL-synced
[ ] Detail panel over the list, linkable and prefetched on hover
[ ] Refund flow: state machine, idempotency, modal route, full a11y
[ ] Webhooks: endpoints, delivery log, manual retry
[ ] API keys: create (shown once), revoke
[ ] Reports: lazy-loaded chart
[ ] Live feed: one SSE connection updating the cache

QUALITY
[ ] tsc --noEmit and lint clean; zero `any`
[ ] Every response schema-validated
[ ] Component tests for each feature; MSW for the network
[ ] Playwright covering login → filter → refund → verify
[ ] Axe accessibility checks passing
[ ] Screen-reader tested once, by hand

EVIDENCE (for interviews)
[ ] docs/architecture.md — decisions, rejections, and triggers that would change them
[ ] docs/performance.md — measured before/after numbers
[ ] A 5-minute demo script, rehearsed
[ ] Deployed to a real URL
```

**That EVIDENCE section is what turns a project into leverage.** Anyone can say "I built a dashboard." Very few can hand over a decision document that lists what they rejected and the trigger that would change their mind.

---

## What's next

The blueprint is the *what*. Next is the *how*: eight phases, ordered so something works after every one, each mapped to the lessons it uses.

Next → **[Lesson 27: Console build log](27-console-build-log.md)**
