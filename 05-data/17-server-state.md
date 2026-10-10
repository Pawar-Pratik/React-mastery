# Lesson 17 — Server state vs client state

> **Why this lesson exists:** the single biggest improvement most React codebases can make is recognising that **server data isn't state — it's a cache**, and treating it like `useState` means hand-rolling caching, deduplication, revalidation, retry and race-condition handling badly. This is also the lesson that retires the most common React pattern in existence: `useEffect` + `fetch`.

**Time:** ~70 minutes · **Prereq:** Lesson 08 · [API Lesson 17](../../API/04-production/17-caching-and-performance.md)

---

## 1. The idea in one sentence

> **Client state is *owned* by your app — you're the source of truth. Server state is *borrowed* — someone else owns it, it can change without you, and what you hold is a cache that's always potentially stale.**

Everything below follows from that distinction.

---

## 2. The two kinds of state

| | **Client state** | **Server state** |
|---|---|---|
| Owner | Your app | The server |
| Source of truth | In memory | Remote |
| Can change without you | No | **Yes** — other users, other tabs, background jobs |
| Needs caching | No | **Yes** |
| Needs revalidation | No | **Yes** |
| Shared between components | Via props/context/store | Via a **cache key** |
| Examples | Modal open, form draft, selected tab | Payments, customers, balance |

**The consequence:** server state needs a set of behaviours that `useState` simply doesn't have.

```
caching            — don't refetch what you already have
deduplication      — 5 components asking for the same payment = 1 request
revalidation       — refetch on window focus, reconnect, or after a mutation
stale-while-revalidate — show cached data instantly, update in the background
retry with backoff — transient failures shouldn't surface as errors
race handling      — the last response to arrive must not win
garbage collection — drop unused cache entries
pagination state   — cursors, "has more", keeping previous data while loading
offline behaviour  — what happens when the network drops
```

**That list is a caching library.** Writing it yourself, per endpoint, is why `useEffect` + `fetch` codebases are the way they are.

---

## 3. Everything wrong with `useEffect` + `fetch`

```tsx
function PaymentDetail({ id }: { id: PaymentId }) {
  const [payment, setPayment] = useState<Payment | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    setLoading(true);
    fetch(`/v1/payments/${id}`)
      .then(r => r.json())
      .then(setPayment)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [id]);

  if (loading) return <Skeleton />;
  if (error) return <Error error={error} />;
  return <View payment={payment!} />;
}
```

Ten problems, and you'll recognise most of them:

1. **Race condition** — change `id` quickly and an older response can land last ([Lesson 08](../02-state-effects/08-effects.md))
2. **No cancellation** — the request continues after unmount, wasting bandwidth and warning about state updates
3. **No cache** — navigate away and back, refetch from scratch, see a skeleton again
4. **No deduplication** — three components needing the same payment make three requests
5. **No revalidation** — the data goes stale and never updates
6. **No retry** — one flaky network blip becomes a visible error
7. **Doesn't check `res.ok`** — a 500 is parsed as JSON and set as data ([API Lesson 02](../../API/01-foundations/02-journey-of-a-request.md))
8. **No validation** — `res.json()` is `any`; `payment!` is a lie ([TS Lesson 15](../../TypeScript/04-real-code/15-the-boundary-and-parsing.md))
9. **Three separate states** — 8 representable combinations, 3 meaningful ([TS Lesson 09](../../TypeScript/03-type-level/09-illegal-states-unrepresentable.md))
10. **Every component repeats all of it**

Fixing all ten by hand, in every component, is the work a query library already did.

---

## 4. TanStack Query

```tsx
function usePayment(id: PaymentId) {
  return useQuery({
    queryKey: ["payments", id],                        // the cache key
    queryFn: ({ signal }) => api.getPayment(id, signal), // signal → automatic cancellation
    staleTime: 30_000,                                  // fresh for 30s; no refetch in that window
  });
}

function PaymentDetail({ id }: { id: PaymentId }) {
  const { data, isPending, isError, error, refetch } = usePayment(id);
  if (isPending) return <Skeleton />;
  if (isError) return <ErrorView error={error} onRetry={refetch} />;
  return <View payment={data} />;                       // `data` is narrowed — no `!`
}
```

All ten problems, solved. But the value isn't the brevity — it's the behaviours you now get for free:

### The mental model: keys, freshness, and garbage collection

```tsx
queryKey: ["payments", { status: "succeeded", cursor }]
```
**The key is the cache identity.** Same key = same cache entry = deduplicated. Different key = different entry. Keys are serialized structurally, so object order doesn't matter.

```
staleTime   — how long data is considered FRESH. While fresh: no refetch, ever.
              Default 0 (immediately stale)
gcTime      — how long UNUSED data stays in memory before being dropped.
              Default 5 minutes
```

**These two are constantly confused, and the distinction is the most important thing to get right:**
- `staleTime` controls **refetching** — "is this worth re-requesting?"
- `gcTime` controls **memory** — "can I throw this away?"

```tsx
staleTime: 0          // refetch on every mount/focus — always current, chatty
staleTime: 30_000     // typical for list data
staleTime: 5 * 60_000 // slow-changing: settings, reference data
staleTime: Infinity   // immutable: a completed payment, a historical invoice
```

**The default `staleTime: 0` surprises people** — it means every mount triggers a background refetch. That's a deliberate "always fresh" default, but on a dashboard with many components it's chatty. Setting a sensible per-query `staleTime` is usually the first real tuning you do.

### Stale-while-revalidate, which is why it *feels* fast
```
Navigate to a payment you viewed 10 seconds ago:
  → cached data renders INSTANTLY (no skeleton)
  → a background refetch starts if stale
  → the UI updates silently if anything changed
```
Same idea as HTTP's `stale-while-revalidate` ([API Lesson 17](../../API/04-production/17-caching-and-performance.md)), applied in the client. **This is the single biggest perceived-performance difference** between a query-library app and an effect-fetching one.

### The defaults worth setting deliberately
```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,
      gcTime: 5 * 60_000,
      retry: (failureCount, error) =>
        error instanceof ApiError && !error.retryable ? false : failureCount < 3,
      //   ↑ don't retry 4xx — the `retryable` flag from your API error contract (API L10)
      retryDelay: attempt => Math.min(1000 * 2 ** attempt, 30_000),   // backoff (API L18)
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
      throwOnError: error => error instanceof ApiError && error.status >= 500,
      //   ↑ 5xx goes to the error boundary; 4xx is handled in the component
    },
  },
});
```
**The `retryable` line is the payoff from the API track.** Because your error contract publishes whether an error is retryable, the client can make the right decision automatically instead of retrying a `422` three times.

---

## 5. The patterns you'll use

```tsx
// ── Dependent queries ──
const { data: payment } = useQuery({ queryKey: ["payments", id], queryFn: ... });
const { data: customer } = useQuery({
  queryKey: ["customers", payment?.customerId],
  queryFn: () => api.getCustomer(payment!.customerId!),
  enabled: !!payment?.customerId,                     // ← don't run until we have the id
});
```

```tsx
// ── Parallel queries — note this is ONE round trip's worth of latency, not three ──
const results = useQueries({
  queries: [
    { queryKey: ["balance"], queryFn: api.getBalance },
    { queryKey: ["payments", { limit: 10 }], queryFn: () => api.listPayments({ limit: 10 }) },
    { queryKey: ["webhook-endpoints"], queryFn: api.listEndpoints },
  ],
});
```

```tsx
// ── Cursor pagination (API Lesson 08) ──
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: ["payments", filters],
  queryFn: ({ pageParam, signal }) => api.listPayments({ ...filters, cursor: pageParam }, signal),
  initialPageParam: undefined as string | undefined,
  getNextPageParam: last => last.next_cursor ?? undefined,   // ← opaque cursor, never constructed
});
const rows = data?.pages.flatMap(p => p.data) ?? [];
```

```tsx
// ── Keep the previous page visible while loading the next ──
useQuery({
  queryKey: ["payments", filters],
  queryFn: ...,
  placeholderData: keepPreviousData,       // no skeleton flash when filters change
});
// Pair with `isPlaceholderData` to dim the table — the same stale-indicator idea as Lesson 11
```

```tsx
// ── Prefetch on hover — the cheapest perceived-perf win ──
<tr onMouseEnter={() => queryClient.prefetchQuery({
      queryKey: ["payments", p.id], queryFn: () => api.getPayment(p.id), staleTime: 30_000 })}>
```

```tsx
// ── Seed the detail cache from the list, so opening a row is instant ──
queryClient.setQueryData(["payments", payment.id], payment);
// Or declaratively:
useQuery({
  queryKey: ["payments", id],
  queryFn: ...,
  initialData: () => queryClient.getQueryData<Paginated<Payment>>(["payments", filters])
                        ?.data.find(p => p.id === id),
  initialDataUpdatedAt: () => queryClient.getQueryState(["payments", filters])?.dataUpdatedAt,
});
```
That `initialDataUpdatedAt` detail matters: without it the seeded data is treated as brand new and won't revalidate.

```tsx
// ── Suspense mode (Lesson 15) ──
const { data } = useSuspenseQuery({ queryKey: ["payments", id], queryFn: ... });
// `data` is always defined — no isPending branch. Loading and errors go to the boundaries.
```

---

## 6. Key design and invalidation

```tsx
// A key factory — the pattern that makes invalidation sane
export const paymentKeys = {
  all:     ["payments"] as const,
  lists:   () => [...paymentKeys.all, "list"] as const,
  list:    (f: Filters) => [...paymentKeys.lists(), f] as const,
  details: () => [...paymentKeys.all, "detail"] as const,
  detail:  (id: PaymentId) => [...paymentKeys.details(), id] as const,
};

// Invalidation is hierarchical — prefixes match
queryClient.invalidateQueries({ queryKey: paymentKeys.all });        // everything payments
queryClient.invalidateQueries({ queryKey: paymentKeys.lists() });     // all lists, not details
queryClient.invalidateQueries({ queryKey: paymentKeys.detail(id) });  // one payment
```

**The key factory is worth adopting on day one.** Without it, keys are string literals scattered across the codebase, a typo silently creates a second cache entry, and nobody can work out what to invalidate after a mutation. With it, invalidation is a one-line, type-safe, hierarchical operation.

**Keys must include everything the query depends on:**
```tsx
// ❌ Filters aren't in the key — changing them serves the wrong cached data
queryKey: ["payments"], queryFn: () => api.listPayments(filters)
// ✅
queryKey: ["payments", "list", filters], queryFn: () => api.listPayments(filters)
```
That's the most common query-library bug: a stale result served for new filters because the key didn't change.

---

## 7. The alternatives

| Library | Model | Best for |
|---|---|---|
| **TanStack Query** | Cache-first, framework-agnostic | **The default.** Richest feature set |
| **SWR** | Smaller, simpler, same core idea | Lighter apps; Next.js-adjacent |
| **RTK Query** | Built into Redux Toolkit | Teams already on Redux |
| **Apollo / urql** | GraphQL, normalised cache | GraphQL backends ([API L21](../../API/05-beyond-rest/21-graphql.md)) |
| **Router loaders** | Fetch on navigation | RR7 / TanStack Router / Remix — often *combined* with a query library |
| **RSC + `use`** | Server fetches, no client cache | Next.js App Router ([Lesson 23](../07-server-react/23-rsc.md)) |

**Normalised vs document caching, briefly:** TanStack Query caches *responses* by key (document cache) — simple, predictable, but the same payment appearing in a list and a detail view is two cache entries that must be invalidated together. Apollo normalises by entity ID, so updating one payment updates it everywhere — powerful, and much more complex. **For REST, document caching plus deliberate invalidation is the right trade**, and knowing why is a good interview answer.

**And the honest note:** with RSC and router loaders, some of this moves server-side. But a client cache is still needed for anything interactive — mutations, optimistic updates, polling, real-time, and cross-component deduplication. In practice most Next.js apps use both.

---

## 8. Ledger Console's data layer

```tsx
// features/payments/api.ts
export const paymentKeys = { /* the factory from §6 */ };

export function usePayments(filters: Filters) {
  return useInfiniteQuery({
    queryKey: paymentKeys.list(filters),
    queryFn: ({ pageParam, signal }) => api.listPayments({ ...filters, cursor: pageParam }, signal),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: last => last.next_cursor ?? undefined,
    placeholderData: keepPreviousData,
    staleTime: 30_000,
  });
}

export function usePayment(id: PaymentId) {
  const qc = useQueryClient();
  return useQuery({
    queryKey: paymentKeys.detail(id),
    queryFn: ({ signal }) => api.getPayment(id, signal),
    // Terminal states never change — cache them forever
    staleTime: q => TERMINAL.has(q.state.data?.status as PaymentStatus) ? Infinity : 30_000,
    initialData: () => findInAnyList(qc, id),          // instant open from the list
  });
}

export function useBalance() {
  return useQuery({
    queryKey: ["balance"],
    queryFn: api.getBalance,
    staleTime: 0,                    // ← money must be exact (API Lesson 17)
    refetchInterval: 30_000,
  });
}
```

**The `staleTime` decisions are the interesting part**, and each mirrors a decision from the API track:

| Query | `staleTime` | Why |
|---|---|---|
| `balance` | `0` + poll | Money must be exact — the same reason the API sends `no-store` |
| `payments` list | 30s | Changes often; stale-while-revalidate is fine |
| A **succeeded** payment | `Infinity` | Terminal state — it will never change again |
| A **pending** payment | 5s + poll | It's actively transitioning |
| `currencies`, `countries` | 24h | Reference data |
| `me`, settings | 5 min | Rarely changes |

That "terminal states are immutable" insight is a genuinely good one: a `succeeded` payment is permanently fixed, so caching it forever is *correct*, not just an optimisation.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Server data goes in a query cache, never `useState`** | It needs caching, dedup, revalidation, retry and race handling |
| **The query key includes every input the query depends on** | Otherwise stale data is served for new inputs |
| **Use a key factory from day one** | Typos create phantom cache entries; invalidation becomes guesswork |
| **Set `staleTime` deliberately per query** | The default of 0 refetches on every mount |
| **Understand `staleTime` (refetch) vs `gcTime` (memory)** | The most commonly confused pair |
| **Pass the `signal` to `fetch`** | Automatic cancellation on unmount and key change |
| **Don't retry non-retryable errors** | Use the `retryable` flag from your API error contract |
| **`keepPreviousData` for filters and pagination** | No skeleton flash on every filter change |
| **`staleTime: Infinity` for terminal/immutable resources** | Correct, not just fast |
| **Prefetch on hover and seed detail caches from lists** | Navigation feels instant |
| **`throwOnError` for 5xx; handle 4xx in-component** | Unexpected errors go to a boundary; expected ones are UI |
| **Never put server data in context or a store** | You're hand-rolling a worse cache |

---

## 10. Interview traps

**Q1. "Why is `useEffect` + `fetch` the wrong default?"**
Give the list, not a vibe: race conditions, no cancellation, no cache, no deduplication, no revalidation, no retry, unchecked `res.ok`, no validation, 8-state loading soup, and it's repeated in every component. **Then the framing:** server data isn't state, it's a cache, and caches need behaviours `useState` doesn't have.

**Q2. "Client state vs server state?"**
Ownership. Client state you own — you're the source of truth. Server state is borrowed: someone else owns it, it can change without you, and what you hold is always potentially stale. That single difference generates the whole feature list of a query library.

**Q3. "`staleTime` vs `gcTime`?"**
`staleTime` = how long data is considered fresh, so it controls **refetching**. `gcTime` = how long unused data stays in memory, so it controls **eviction**. Defaults: 0 and 5 minutes. A query can be stale but still cached (shown instantly, refetched in the background) — that's stale-while-revalidate.

**Q4. "What goes in a query key?"**
Everything the query result depends on — resource type, ID, filters, pagination, and anything else that changes the response. A missing input means stale data served for new inputs, which is the most common bug with these libraries. Use a key factory so invalidation is hierarchical and typo-proof.

**Q5. "How does a query library solve the race condition?"**
Two ways: it passes an `AbortSignal` so a superseded request is cancelled, and it only commits a response if it belongs to the currently-active query for that key. A late response for a stale key is discarded — the same "old synchronization is torn down" idea from [Lesson 08](../02-state-effects/08-effects.md).

**Q6. "How do you decide `staleTime` for a given query?"**
By how much staleness the data can tolerate. Money and balances: 0 (and poll). Lists: tens of seconds. **Terminal resources — a succeeded payment — `Infinity`, because they're immutable.** Reference data: hours. **Naming the terminal-state case is the answer that stands out.**

**Q7. "Should server data ever go in Redux/context?"**
No — you'd reimplement caching, dedup, revalidation and retry, badly. Redux is for *client* state. This is the most common state-management mistake, and modern Redux Toolkit ships RTK Query precisely because of it.

**Q8. "Normalised vs document caching?"**
Document caching stores responses by key — simple and predictable, but the same entity in a list and a detail view is two entries that must be invalidated together. Normalised caching (Apollo) stores entities by ID so one update propagates everywhere — powerful, much more complex, and a natural fit for GraphQL. **For REST, document + deliberate invalidation is the right trade.**

**Q9. "How do you make navigating to a detail page instant?"**
Three layers: prefetch on hover, seed the detail cache from the list you already have (`initialData` + `initialDataUpdatedAt`), and `staleTime` high enough that returning doesn't refetch. The user sees data immediately and a background refresh corrects it if needed.

**Q10. "Does RSC make query libraries obsolete?"**
No. RSC and loaders move *initial* fetching to the server, which is a real win. But you still need a client cache for mutations, optimistic updates, polling, real-time updates and cross-component deduplication. Most Next.js apps use both — server fetch for the first render, a client cache for interaction.

---

## 11. Build & break

### Build — the ten problems, demonstrated
Write the naïve `useEffect` version of `PaymentDetail`, then reproduce each problem deliberately:
1. Change `id` rapidly with variable latency → wrong payment displayed
2. Unmount mid-request → a state update on an unmounted component
3. Navigate away and back → skeleton again
4. Render three components with the same `id` → three requests in the Network panel
5. Return a 500 → it's parsed and set as `data`

Then replace it with `useQuery` and confirm all five are gone. **That before/after is the most persuasive thing in this module.**

### Build — Ledger's data layer
Implement the §8 hooks with the key factory, the per-query `staleTime` table, `keepPreviousData`, hover prefetching, and list→detail cache seeding. Open the React Query Devtools and watch the cache: fresh vs stale vs inactive, and what happens on focus, on reconnect, and after `gcTime` elapses.

**The Devtools are the learning tool here** — leave them open for a week and the mental model builds itself.

### Break — five experiments
1. **Missing key input.** Omit `filters` from the key, change a filter, and watch the old results appear instantly and never update.
2. **`staleTime: 0` chattiness.** Open the Network panel on a dashboard with ten queries and switch browser tabs a few times. Count the requests. Set sensible stale times and recount.
3. **Retrying a 422.** Remove the `retryable` check and submit invalid data. Watch three identical doomed requests.
4. **No `signal`.** Drop the `signal` from `queryFn`, navigate away mid-request, and watch the request complete in the Network panel.
5. **`gcTime` eviction.** Set `gcTime: 5000`, navigate away for ten seconds, come back, and watch the skeleton return — then raise it and watch instant render.

### Explain out loud (90 seconds)
1. Client vs server state, by ownership.
2. Five things `useEffect` + `fetch` doesn't do.
3. `staleTime` vs `gcTime`.
4. What goes in a query key, and why a factory.
5. How you'd decide `staleTime` for three different resources.

---

## What's next

Reading is the easy half. Next: **writes** — mutations, cache invalidation, optimistic updates with correct rollback, and the idempotency story that connects directly back to the API track.

Next → **[Lesson 18: Mutations, optimistic updates & races](18-mutations-and-optimistic.md)**
