# Lesson 27 — Console build log

> **Why this lesson exists:** knowing *what* to build (Lesson 26) and knowing the techniques (Modules 1–7) still leaves the hardest question: **in what order?** Build it wrong and you spend a week on scaffolding before anything renders, lose momentum, and abandon it. This log is ordered so **something works after every phase**, and each phase only uses lessons you've already read.

**Time:** 8 phases · roughly 35–50 hours · **Prereq:** Lesson 26

---

## How to use this

- **One phase at a time. Finish it.** Each ends with a "Demonstrate" step that actually works in a browser.
- **Type the code.** Snippets here are anchors; the referenced lessons have the full versions.
- **Write the test in the same sitting as the feature.** Later doesn't come.
- **Keep `DECISIONS.md`** as you go — one line per choice, with the reason. At the end you have `docs/architecture.md` for free, and it'll be honest rather than reconstructed.
- **Commit per phase**, with a message naming what now works. Your git log becomes a narrative you can walk an interviewer through.

> **You need the Ledger API running.** Build [API Phases 0–4](../../API/07-project/29-ledger-build-log.md) first, or point at a mock (MSW with the OpenAPI spec) — but the API is more interesting, and having built both is the stronger story.

---

## Phase 0 — Foundations (3–4 h)

**Goal:** an empty but correct shell. Everything after this is features.

```bash
npm create vite@latest ledger-console -- --template react-ts
cd ledger-console && npm i
npm i react-router @tanstack/react-query @tanstack/react-virtual zod
npm i -D vitest @testing-library/react @testing-library/user-event msw
npm i -D eslint typescript-eslint eslint-plugin-react-hooks vite-plugin-checker
```

**Build:**
1. `tsconfig.json` with `strict`, **`noUncheckedIndexedAccess`**, **`exactOptionalPropertyTypes`** ([TS L02](../../TypeScript/01-foundations/02-tsconfig-deeply.md)).
2. `vite-plugin-checker` so type errors appear in the browser overlay — your dev server doesn't type-check ([TS L18](../../TypeScript/05-ecosystem/18-tooling-and-performance.md)).
3. ESLint with `react-hooks` and `@typescript-eslint/strict-type-checked`, plus the feature-boundary `no-restricted-imports` rule ([L16](../04-composition/16-routing-and-architecture.md)).
4. The folder structure from [Lesson 26 §4](26-console-blueprint.md), with a one-paragraph `README.md` in each top-level directory.
5. `src/types/` — copy the TypeScript track's shared layer: brands, `Money`, `Result`, the `Payment` union.
6. `components/ui/` primitives: `Button`, `Badge`, `Input`, `Skeleton` — extending `ComponentPropsWithoutRef` ([TS L17](../../TypeScript/05-ecosystem/17-typescript-with-react.md)).
7. Error boundary (`react-error-boundary`) + `AsyncBoundary` pattern ([L15](../04-composition/15-boundaries-suspense-portals.md)).

**Demonstrate:** a page rendering a `Button` and a `Badge`, with a deliberate type error visible in the browser overlay.

**Test:** `Button` renders, forwards native props, and is disabled while `loading`.

> **Lessons used:** 02 (props as API), 13 (composition), 15 (boundaries), 16 (architecture), TS 02/17.

---

## Phase 1 — Routing & auth (4–5 h)

**Goal:** you can log in, and protected routes work.

**Build:**
1. Route tree from [Lesson 26 §2](26-console-blueprint.md), with `lazy` imports and per-route `errorElement`.
2. `ProtectedLayout` with `<Navigate state={{ from: location }} replace />` ([L16](../04-composition/16-routing-and-architecture.md)).
3. `AuthProvider` — access token in memory, refresh in an `HttpOnly` cookie, `undefined` default + throwing `useAuth` ([L09](../03-hooks/09-context.md)).
4. Login form with `useActionState` ([L14](../04-composition/14-forms.md)).
5. `lib/api-client.ts` — typed fetch with the **single-flight refresh** from [Lesson 26 §5](26-console-blueprint.md), schema-parsed responses, and `AbortSignal` support.
6. `<ScrollRestoration />`, a `useNavigation()` progress bar, and hover prefetching on links.

```ts
// The single-flight refresh — the detail that matters
let refreshing: Promise<string> | null = null;
if (res.status === 401) {
  refreshing ??= refreshAccessToken().finally(() => { refreshing = null; });
  await refreshing;
  res = await req();
}
```

**Demonstrate:**
- Hit `/payments` logged out → redirected to `/login`
- Log in → land back on `/payments` (not the home page)
- Expire the access token in devtools → the next request refreshes and retries silently
- Fire five concurrent requests with an expired token → **one** refresh call in the Network panel

**Test:** the redirect carries `state.from`; `useAuth` throws outside its provider; concurrent 401s trigger exactly one refresh.

> **Lessons used:** 09 (context), 14 (forms), 16 (routing), API 13 (tokens), TS 15 (parsing).

---

## Phase 2 — The payments list (5–6 h)

**Goal:** real data on screen, with filters in the URL.

**Build:**
1. `features/payments/schemas.ts` — Zod schemas, types via `z.infer`.
2. `features/payments/api.ts` — the **key factory** and `usePayments` as an `useInfiniteQuery` with cursors ([L17](../05-data/17-server-state.md)).
3. `useFilters()` — URL state with schema parsing, `{ replace: true }`, cursor reset on filter change ([L16](../04-composition/16-routing-and-architecture.md)).
4. `FilterBar` — local draft state + `useDebouncedValue(300)` → URL.
5. A plain (non-virtualized) table first, with `AsyncBoundary` for loading/error/empty.
6. Query client defaults: per-query `staleTime`, retry only on `error.retryable`, `throwOnError` for 5xx.

```ts
// staleTime by resource — each mirrors an API decision (Lesson 17)
balance:                 0 + refetchInterval 30s   // money must be exact
payments list:           30s
a TERMINAL payment:      Infinity                   // immutable
a pending payment:       5s + poll
currencies/countries:    24h
```

**Demonstrate:**
- Filter by status → the URL changes → the list updates
- Copy the URL into a new tab → identical view
- Back button steps through *navigations*, not every keystroke
- Type fast → one request per pause, not per character

**Test:** filters round-trip through the URL; a malformed URL (`?limit=abc`) renders defaults instead of crashing; changing a filter clears the cursor.

> **Lessons used:** 05 (state placement), 16 (URL state), 17 (query cache), TS 15 (schemas).

---

## Phase 3 — Virtualization & scale (4–5 h)

**Goal:** 500,000 rows at 60fps.

**Build:**
1. Seed the API with 500k payments: `npm run seed -- --payments 500000`.
2. Replace the table with `useVirtualizer` — keyed by **payment ID**, positioned with `transform` ([L21](../06-performance/21-render-performance.md)).
3. Infinite loading: fetch the next page when 10 rows from the end.
4. `memo` the row; `useCallback` the select handler so it's stable per row.
5. Accessibility: `role="grid"`, `aria-rowcount={total}`, `aria-rowindex`, arrow-key navigation via `scrollToIndex`.
6. `keepPreviousData` so filter changes don't flash a skeleton.
7. `useDeferredValue` on filters + an `isStale` opacity ([L11](../03-hooks/11-concurrent-hooks.md)).

**Demonstrate — and record the numbers:**
```
DOM nodes:        ~23 (not 500,000)
Scroll:           60fps, no React renders during scroll
Commit duration:  <8ms
Typing:           FilterBar only re-renders; the table lags without blocking
```

**Test:** ticking a checkbox in row 2, then prepending a payment, keeps the tick on the **same payment** (the index-key bug, guarded); `aria-rowcount` reports the total, not the window.

**Break it first:** render 500k rows *without* virtualization and watch the tab die. **Record that.** It's the most persuasive before/after you'll have.

> **Lessons used:** 04 (keys), 10 (memo), 11 (deferred), 17 (infinite query), 21 (virtualization).

---

## Phase 4 — The refund flow (5–6 h)

**Goal:** the feature that demonstrates the most.

**Build:**
1. `refundReducer` — the state machine from [Lesson 06](../02-state-effects/06-reducers-and-state-machines.md), switching on **state first**.
2. `useIdempotencyKey(resetOn)` — a ref, stable across retries ([L07](../02-state-effects/07-refs-and-escape-hatches.md)).
3. `useRefundPayment()` — mutation with full blast-radius invalidation (payment, lists, balance), error branching on `code`, `retryable`-aware retry, `requestId` in the failure toast ([L18](../05-data/18-mutations-and-optimistic.md)).
4. `RefundModal` on a **modal route** (`/payments/:id/refund`) — linkable and refresh-proof.
5. Radix Dialog (or `<dialog>`) for focus trap, focus restore, `Escape`, inert background.
6. Form with RHF + Zod, `z.input`/`z.output` for the amount transform, server field errors mapped back to inputs.

**Demonstrate:**
- Open the refund modal → **copy the URL** → open it in a new tab → the modal is there
- Click Refund twice rapidly on a throttled connection → **one** refund, confirmed in the API's request log
- Force a `409 already_refunded` → error toast **and** a refetch
- Force a `422` → errors land on the correct form fields
- Tab through the modal → focus never escapes; `Escape` closes; focus returns to the trigger

**Test:** the reducer rejects a second `submit` from `submitting` (a pure function test, no rendering); concurrent mutations produce one request; a `409` triggers invalidation.

> **Lessons used:** 06 (state machine), 07 (refs), 13 (composition), 14 (forms), 15 (modals), 18 (mutations), API 10/18.

---

## Phase 5 — Webhooks, keys & reports (4–5 h)

**Goal:** breadth, and the code-splitting story.

**Build:**
1. **Webhook endpoints:** list, create (with `assertSafeUrl`-style client validation), toggle enabled — the **optimistic** case ([L18](../05-data/18-mutations-and-optimistic.md)).
2. **Delivery log:** attempt history, response status and truncated body, manual retry (optimistic status → `pending`).
3. **API keys:** create with the key **shown once** (never re-fetchable), copy-to-clipboard, revoke with confirmation.
4. **Reports:** `lazy()` with Suspense + an error boundary for chunk failures, Recharts in a separate chunk.
5. Hover prefetching on every route link.

**Demonstrate:**
- Toggle an endpoint with the API returning 500 half the time → it flips back, with a toast
- Open `/reports` → check the Network panel: a separate chunk, loaded on demand
- Hover `/reports` first → by the time you click, the chunk is already there
- The API key is shown once and cannot be retrieved again

**Test:** the optimistic toggle rolls back on failure; the API key value is never in a subsequent response; a failed chunk load shows a reload prompt.

> **Lessons used:** 15 (Suspense + boundaries), 18 (optimistic), 22 (code splitting).

---

## Phase 6 — Real-time (3–4 h)

**Goal:** live updates, with no page component knowing about them.

**Build:**
1. `lib/realtime.tsx` — **one** `EventSource` in a provider at the root.
2. `useEventCallback` for the handler → connects once, always runs the latest logic ([L07](../02-state-effects/07-refs-and-escape-hatches.md)).
3. Cache integration: `setQueryData` for the detail, `invalidateQueries` for lists and balance ([L19](../05-data/19-realtime.md)).
4. Version guard so out-of-order events are discarded; dedupe and bound the live list.
5. **Invalidate on reconnect** — the cache may have drifted.
6. `ConnectionBadge` with `role="status"` and `aria-live="polite"`.
7. Buffer-and-flush at 250ms for bursts.

**Demonstrate:**
- Create a payment via `curl` against the API → **it appears in the open table**, with no page code involved
- Kill the API → the badge shows "Reconnecting…", with backoff visible in the Network panel
- Restart it → reconnect + a full invalidation refetch
- Emit an unknown event type → ignored, nothing breaks

**Test:** the handler is stable (one connection across many renders); an older-version event is discarded; a duplicate event doesn't duplicate the row.

> **Lessons used:** 07 (refs), 08 (effects + cleanup), 17/19 (cache + real-time), API 23.

---

## Phase 7 — Performance, a11y & polish (4–5 h)

**Goal:** measured, accessible, deployed.

**Build:**
1. **Measure the baseline first** — LCP, INP, CLS, bundle size, at 4× CPU and Fast 4G ([L20](../06-performance/20-measuring.md)). Write it down *before* optimising.
2. Bundle analyzer → replace heavy dependencies (`Intl` over a date library, `Object.groupBy` over lodash).
3. `preconnect` to the API origin; `preload` the font; `font-display: swap`.
4. Skeletons matched to real content dimensions (CLS).
5. Route-level splitting verified; check nothing above the fold is lazy.
6. Full keyboard pass: every interaction reachable, visible focus rings, `/` focuses search.
7. **Screen-reader pass** (VoiceOver ⌘F5 / NVDA): forms announce errors, the grid announces the real row count, the connection badge is announced.
8. `axe-core` in the test suite; a `size-limit` and Lighthouse budget in CI.
9. Deploy to a real URL with HTTPS.

**Demonstrate:** the before/after performance table. `npm run build && npx serve dist` and run Lighthouse throttled.

> **Lessons used:** 20, 21, 22, 14 (a11y), 15 (boundaries).

---

## Phase 8 — Test, review & present (4–5 h)

**Goal:** turn it into interview leverage.

**Build:**
1. **Test suite** ([Lesson 28](28-testing-react.md)): MSW handlers from the OpenAPI spec, component tests per feature, the reducer tested as a pure function, Playwright for login → filter → refund → verify.
2. **Self-review** against the checklists: no `any`, no unexplained `as`/`!`, every response parsed, every union exhaustive, boundaries placed. Write findings into `docs/review-findings.md` and fix them.
3. **`docs/architecture.md`** from your `DECISIONS.md` — including the **"explicitly rejected"** and **"triggers that would change this"** sections.
4. **`docs/performance.md`** with the measured numbers.
5. **The 5-minute demo script**, written and rehearsed:

```
1. Here's the app, here's the API it talks to.                        (15s)
2. 500,000 payments. Scroll — 60fps, 23 DOM nodes in the inspector.
   And the first thing I'd say: the client never holds 500k. The API
   filters and paginates; virtualization renders the hundred I hold.  (60s)
3. Type in the filter. The input never lags — local state, debounced
   to the URL, deferred for rendering. Three different mechanisms.    (45s)
4. Copy the URL, open it in a new tab. Same view. Filters are state.  (20s)
5. Refund a payment. I'll click twice on a throttled connection —
   one refund. State machine plus an idempotency key.                 (60s)
6. Here's the API's request log confirming one request.                (15s)
7. I'll create a payment with curl. Watch it appear live — SSE
   updating the query cache. No page component knows the stream exists.(45s)
8. Here's the decision doc, including what I rejected and why.         (30s)
```

**The presentation rule: lead with the hard parts.** Interviewers have seen a thousand CRUD demos. Nobody shows them a double-click on a throttled connection producing exactly one refund, with the server log to prove it.

---

## The build order, and why

| Phase | Why here |
|---|---|
| 0 Foundations | Boundaries and primitives must exist before features, or you retrofit them into 40 components |
| 1 Routing & auth | Everything else is behind a login; the API client is used by every feature |
| 2 Payments list | The core screen; needs auth and the API client |
| 3 Virtualization | Needs real data volume to be meaningful |
| 4 Refund flow | Needs a payment to refund |
| 5 Breadth | Needs the patterns from 0–4 |
| 6 Real-time | Needs a cache to update |
| 7 Performance | Needs something to measure |
| 8 Review | Needs something to review |

**The two phases people get wrong:** they skip **Phase 3's "break it first"** (rendering 500k rows unvirtualized) and so never see why it matters, and they leave **Phase 7's baseline measurement** until after optimising, so they can't prove anything improved.

---

## If you have less time

| Available | Build phases | You still get |
|---|---|---|
| **~12 h** | 0, 1, 2, 4 | Auth, a real list, and the refund flow — **the most interviewable part** |
| **~20 h** | + 3, 6 | Virtualization at scale and live updates |
| **~35 h** | + 5, 7 | Complete and deployed |
| **~45 h** | + 8 | Interview-ready with evidence |

**If you build only one thing, build Phase 4.** The refund flow alone demonstrates state machines, idempotency, mutations, modals, forms, error contracts and accessibility — and the double-click demo is the single most persuasive thing in the project.

---

## What's next

You've built it. The last technical lesson is how to keep it correct: testing React so that refactors are safe and regressions are caught — testing **behaviour**, not implementation.

Next → **[Lesson 28: Testing React](28-testing-react.md)**
