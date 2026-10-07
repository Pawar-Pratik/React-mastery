# Lesson 16 — Routing & app architecture

> **Why this lesson exists:** two things decide whether a React codebase is pleasant at 200 components — **where state lives** and **how files are organised** — and routing sits at the centre of both. The URL is the most under-used state container in React, and folder structure is the decision that's cheapest to make on day one and most expensive to change on day 400.

**Time:** ~60 minutes · **Prereq:** Lessons 05, 13

---

## 1. The idea in one sentence

> **The URL is application state that's shareable, bookmarkable, back/forward-navigable and refresh-proof for free — so anything a user would want to link to belongs there, not in `useState`.**

---

## 2. The URL as a state container

```
/payments?status=succeeded&status=refunded&created[gte]=2026-01-01&sort=-amount&cursor=eyJ...
```

Everything in that URL is state. Putting it in `useState` instead costs you five things a user actually notices:

| In the URL | In `useState` |
|---|---|
| Shareable — paste it in Slack, the recipient sees the same view | Gone |
| Bookmarkable | Gone |
| Back/forward works | Back leaves the page entirely |
| Survives refresh | Reset |
| Debuggable — a support ticket can include the URL | "What filters did you have?" |
| Re-render scope is the route, not a shared parent | A cascade from wherever the state lives |

**What belongs in the URL:** the current route, resource IDs, filters, sort order, pagination cursor, the active tab, search queries, and open-modal state *if the modal is linkable* (a payment detail panel, yes; a confirm dialog, no).

**What doesn't:** in-progress form drafts, hover state, transient UI (a dropdown being open), and anything containing PII or secrets — **URLs are logged at every hop** ([API Lesson 02](../../API/01-foundations/02-journey-of-a-request.md)).

```tsx
// The idiomatic hook — a typed wrapper over URLSearchParams
export function useFilters() {
  const [params, setParams] = useSearchParams();

  const filters = useMemo<Filters>(() => FiltersSchema.parse({
    status: params.getAll("status"),
    createdGte: params.get("created[gte]") ?? undefined,
    sort: params.get("sort") ?? "-created_at",
  }), [params]);                               // ← PARSE it: the URL is a boundary (TS Lesson 15)

  const setFilter = useCallback(<K extends keyof Filters>(key: K, value: Filters[K]) => {
    setParams(prev => {
      const next = new URLSearchParams(prev);
      next.delete(key);
      if (Array.isArray(value)) value.forEach(v => next.append(key, String(v)));
      else if (value != null && value !== "") next.set(key, String(value));
      next.delete("cursor");                   // ← changing a filter resets pagination
      return next;
    }, { replace: true });                     // ← replace, not push: don't fill history per keystroke
  }, [setParams]);

  return { filters, setFilter };
}
```

Three details that matter:
- **Parse the URL with a schema.** Users edit URLs, links go stale, and `?limit=abc` shouldn't crash the page.
- **`{ replace: true }` for filter changes**, so the back button doesn't walk through every filter tweak. Use `push` for genuine navigations.
- **Reset dependent params** — changing a filter must clear the cursor, or you request page 5 of a different result set.

> `nuqs` is worth knowing: typed URL state with parsers, defaults, and transitions, used widely with Next.js. It's the library form of the hook above.

---

## 3. Routing libraries

| Library | Model | Best for |
|---|---|---|
| **React Router 7** | Component or data routes, loaders/actions | SPAs; the most common choice |
| **TanStack Router** | File or code routes, **fully type-safe** params/search | TypeScript-heavy apps that want typed routes |
| **Next.js App Router** | File-based, RSC-first | Full-stack, SSR ([Lesson 24](../07-server-react/24-nextjs-app-router.md)) |
| **Remix** *(now merged into RR7)* | Loaders/actions, progressive enhancement | Same lineage as RR7's data mode |

```tsx
// React Router data mode — the important shift
const router = createBrowserRouter([
  {
    path: "/",
    element: <Layout />,
    errorElement: <RootError />,
    children: [
      {
        path: "payments",
        loader: paymentsLoader,               // ← fetch STARTS before the component renders
        element: <PaymentsPage />,
        errorElement: <RouteError />,
        children: [
          { path: ":id", loader: paymentLoader, element: <PaymentDetail /> },
        ],
      },
    ],
  },
]);
```

**Why loaders matter — the waterfall:**
```
❌ Component-fetches-in-effect:
   navigate → render → mount → effect → fetch → render again
   (and a nested route repeats the whole chain, serially)

✅ Loader:
   navigate → fetch STARTS immediately, in parallel for all matched routes
           → render once, with data
```
A three-level nested route with effect-based fetching produces **three sequential round trips**. Loaders fetch all matched levels in parallel. That's often a 2–3× improvement on navigation, and it's the main architectural reason data-mode routing exists.

```tsx
// Type-safe routing (TanStack Router) — params and search validated at the route
const route = createRoute({
  path: "/payments/$paymentId",
  validateSearch: (s) => FiltersSchema.parse(s),     // ← search params are typed AND validated
  loader: ({ params }) => fetchPayment(params.paymentId),
});
const { paymentId } = route.useParams();              // typed as string, checked at build time
```
**Typed route params are a genuine quality-of-life win** in a large app — a renamed route becomes a compile error instead of a 404 in production.

---

## 4. Routing patterns you'll need

```tsx
// Protected routes — guard at the layout, not in every page
function ProtectedLayout() {
  const { principal } = useAuth();
  const location = useLocation();
  if (!principal) return <Navigate to="/login" state={{ from: location }} replace />;
  return <Outlet />;
}
// `state.from` so login can send them back where they were — the detail people forget
```

```tsx
// Modal routes — a detail panel that's linkable, over a list that stays mounted
<Route path="payments" element={<PaymentsPage />}>
  <Route path=":id" element={<PaymentDetailPanel />} />     {/* renders in an <Outlet /> */}
</Route>
// /payments/pi_123 shows the table AND the panel. Refreshing works. The URL is shareable.
```

```tsx
// Pending navigation UI — essential once loaders exist
const navigation = useNavigation();
{navigation.state === "loading" && <TopProgressBar />}
```

```tsx
// Scroll restoration — browsers do this for MPAs; SPAs must do it themselves
<ScrollRestoration />
```

```tsx
// Prefetch on hover/intent — the cheapest perceived-performance win there is
<Link to="/payments/pi_1" prefetch="intent">View</Link>
// By the time the click lands, the data is often already there.
```

```tsx
// Route-level code splitting (Lesson 15)
{ path: "settings", lazy: () => import("./pages/Settings") }
```

**Prefetch-on-intent deserves emphasis:** hovering a link for 200ms before clicking is enough to start the fetch, so navigation feels instant with zero extra complexity. It's one of the highest ratio-of-benefit-to-effort techniques in front-end performance.

---

## 5. Folder architecture

### The two structures, and when each breaks

```
❌ By type — breaks at scale
src/
├── components/      # 200 files
├── hooks/           # 60 files
├── utils/           # 40 files
└── pages/
```
Every feature is smeared across four folders. Adding "webhooks" means touching four places; deleting it means hunting through four. **Fine up to ~30 components, painful after.**

```
✅ By feature — the default for anything real
src/
├── features/
│   ├── payments/
│   │   ├── components/      # PaymentsTable, PaymentDetail, RefundModal
│   │   ├── hooks/           # usePayments, useRefundPayment
│   │   ├── api.ts           # the query/mutation layer for this feature
│   │   ├── schemas.ts       # Zod schemas + inferred types
│   │   └── index.ts         # the feature's PUBLIC surface
│   ├── webhooks/
│   └── auth/
├── components/
│   ├── ui/                  # Button, Badge, Input — primitives
│   └── patterns/            # Modal, DataTable, Page — composed
├── hooks/                   # genuinely cross-cutting only
├── lib/                     # api client, query client, logger, utils
├── routes/                  # route definitions + loaders
└── types/                   # shared domain types (from the TS track)
```

**The rules that make it work:**
1. **A feature owns everything specific to it** — components, hooks, schemas, API calls.
2. **`index.ts` is the feature's public API.** Other features import only from there.
3. **Features don't import each other's internals.** If two features need the same thing, it moves to `components/`, `hooks/` or `lib/`.
4. **Shared means used by 2+ features**, not "might be shared one day."

```jsonc
// Enforce rule 3 mechanically, or it decays within a sprint
// .eslintrc — no-restricted-imports
"patterns": [{
  "group": ["**/features/*/!(index)*", "**/features/*/**"],
  "message": "Import from the feature's index.ts, not its internals."
}]
```
**A rule nobody enforces is decoration.** This one lint rule is what keeps feature boundaries real after you've left.

### Where things go — the decision tree
```
Used by one feature?              → features/<name>/
Used by 2+ features?              → components/ | hooks/ | lib/
A design-system primitive?         → components/ui/
A composed pattern (modal, table)? → components/patterns/
Domain types/schemas shared everywhere? → types/ (the TS track's shared layer)
Framework/infra (query client, api, logger)? → lib/
```

---

## 6. The layers

```
┌──────────────────────────────────────────────┐
│  routes/          URL → page, loaders, guards │
├──────────────────────────────────────────────┤
│  features/        domain UI + domain hooks     │
├──────────────────────────────────────────────┤
│  components/      ui primitives + patterns     │
├──────────────────────────────────────────────┤
│  lib/             api client, query client,    │
│                   logger, formatting           │
├──────────────────────────────────────────────┤
│  types/           domain types & schemas       │
└──────────────────────────────────────────────┘
        dependencies flow DOWNWARD only
```

**One-directional dependencies** is the rule that keeps this useful: `components/ui` must not import from `features/`. If a `Button` needs to know about payments, it isn't a primitive.

```tsx
// The shape a page ends up as — thin, composed, no business logic
export function PaymentsPage() {
  const { filters, setFilter } = useFilters();              // URL state
  const query = usePayments(filters);                        // feature hook → query layer

  return (
    <Page title="Payments" actions={<ExportButton filters={filters} />}>
      <FilterBar value={filters} onChange={setFilter} />
      <AsyncBoundary state={query}>
        {data => <PaymentsTable rows={data.items} />}
      </AsyncBoundary>
    </Page>
  );
}
```
**If a page component contains business logic, it's in the wrong layer.** Pages compose; features implement.

---

## 7. Ledger Console's structure

```
src/
├── routes/
│   ├── router.tsx                 # route tree, lazy imports, error elements
│   └── guards.tsx                 # ProtectedLayout, RequireScope
├── features/
│   ├── payments/  components/ hooks/ api.ts schemas.ts index.ts
│   ├── refunds/
│   ├── webhooks/
│   ├── api-keys/
│   └── auth/
├── components/ui/        button badge input skeleton table
├── components/patterns/  modal data-table page async-boundary toast
├── hooks/                use-debounced-value use-media-query use-filters
├── lib/
│   ├── api-client.ts      # typed fetch (TS Lesson 08/15)
│   ├── query-client.ts    # TanStack Query config (Lesson 17)
│   ├── format.ts          # formatMoney, formatDate
│   └── logger.ts
└── types/                 # the TypeScript track's shared layer: brands, Money, Payment, Result
```

| Route | Loader | Notes |
|---|---|---|
| `/` | — | redirect to `/payments` |
| `/login` | — | public |
| `/payments` | `paymentsLoader` | filters + cursor in the URL |
| `/payments/:id` | `paymentLoader` | detail panel over the table, linkable |
| `/payments/:id/refund` | — | modal route; the panel stays mounted behind it |
| `/webhooks` | `endpointsLoader` | |
| `/webhooks/:id/deliveries` | `deliveriesLoader` | |
| `/settings/api-keys` | — | lazy-loaded chunk |

**The `/payments/:id/refund` modal route is worth calling out:** the refund dialog has its own URL, so it's linkable from a support ticket, survives a refresh, and closes correctly with the back button. That's three real product wins from a routing decision.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Anything a user would link to goes in the URL** | Shareable, bookmarkable, back/forward, refresh-proof |
| **Parse URL params with a schema** | Users edit URLs; stale links exist; `?limit=abc` shouldn't crash |
| **`{ replace: true }` for filter changes, push for navigations** | Or the back button walks through every keystroke |
| **Reset dependent params (cursor) when a filter changes** | Otherwise you request page 5 of a different result set |
| **Never put PII or tokens in a URL** | Logged at every hop |
| **Use route loaders to kill the fetch waterfall** | Nested routes fetch in parallel instead of serially |
| **Guard at the layout, and remember `state.from`** | So login returns them where they were |
| **Prefetch on hover/intent** | Highest benefit-to-effort ratio in perceived performance |
| **Route-level code splitting + an error boundary for chunk failures** | Chunks fail after deploys |
| **Organise by feature; `index.ts` is the public surface** | Type-based folders smear every feature across four places |
| **Enforce feature boundaries with a lint rule** | Unenforced structure decays within a sprint |
| **Dependencies flow downward only** | `components/ui` must never import from `features/` |
| **Pages compose; features implement** | Business logic in a page means it's in the wrong layer |

---

## 9. Interview traps

**Q1. "Where should filter state live?"**
The URL. Then name the five concrete benefits — shareable, bookmarkable, back/forward, refresh-proof, debuggable from a support ticket — **and the re-render benefit**: it removes the cascade from a shared parent ([Lesson 03](../01-mental-model/03-rendering-and-commit.md)). Most candidates say `useState` or a store.

**Q2. "What's the fetch waterfall and how do routing loaders fix it?"**
Component-level fetching means navigate → render → mount → effect → fetch, and a nested route repeats it serially — three levels is three sequential round trips. Loaders start fetching as soon as the URL matches, in parallel for every matched route, so you render once with data. Often 2–3× faster navigation.

**Q3. "How do you structure a React project?"**
By feature, with a public `index.ts` per feature, shared primitives in `components/ui`, composed patterns in `components/patterns`, infra in `lib`. **Then the two rules that make it survive:** dependencies flow downward only, and a lint rule enforces that features import each other only through `index.ts`.

**Q4. "When does folder-by-type break down?"**
Around 30 components. After that, every feature is smeared across four folders — adding a feature touches four places and deleting one means hunting. The signal is that you can't tell what the app *does* from its folder tree.

**Q5. "How do you make a modal linkable?"**
Give it a route: `/payments/:id/refund` rendered in an `<Outlet />` so the underlying page stays mounted. It becomes shareable, refresh-proof, and closable with the back button. **The three product wins from one routing decision.**

**Q6. "How do you handle auth-protected routes?"**
A layout route that checks the principal and `<Navigate to="/login" state={{ from: location }} replace />`. `replace` so the protected URL doesn't sit in history; `state.from` so login can return them. And the crucial caveat: **client-side guards are UX, not security** — the API must authorize every request regardless ([API Lesson 15](../../API/03-security/15-authorization-and-multitenancy.md)).

**Q7. "React Router or TanStack Router or Next.js?"**
Next.js for full-stack and SSR. TanStack Router for TypeScript-heavy SPAs wanting typed params and search validation. React Router 7 for everything else — the most common, and its data mode gives you loaders. Decide by whether you need a server, and how much you value typed routes.

**Q8. "Where would you NOT put state in the URL?"**
Form drafts, transient UI (hover, an open dropdown), anything with PII or tokens, and anything that would make the URL change on every keystroke without `replace`. The test: *"would a user want to send someone this link?"*

**Q9. "How do you prevent the back button from walking through every filter change?"**
`{ replace: true }` on filter updates, `push` only for genuine navigations. A search input additionally debounces before committing to the URL, so you don't push 12 history entries for one word.

**Q10. "How do you keep architecture from decaying?"**
Enforce it mechanically: a lint rule for feature-boundary imports, a `README.md` in each top-level folder saying what belongs there, and review findings when something lands in the wrong layer. **Documentation without enforcement decays in a sprint.**

---

## 10. Build & break

### Build — Ledger Console's routing
Implement the §7 route table with: data loaders for each list route, `ProtectedLayout` with `state.from`, the `/payments/:id` detail panel as a child route, the `/payments/:id/refund` modal route, `<ScrollRestoration />`, a top progress bar driven by `useNavigation()`, route-level `lazy` imports, and per-route error elements.

Then verify each product property by hand:
- Copy `/payments?status=succeeded&sort=-amount` into a new tab — same view?
- Apply three filters, press back three times — does it step back one filter at a time, or leave the page?
- Open the refund modal, refresh — does it reopen?
- Log out, hit a protected URL, log in — do you land where you were?

### Build — `useFilters` with schema parsing
Implement §2's hook with Zod parsing, `replace: true`, and cursor reset. Then **deliberately break the URL** (`?status=nonsense&limit=abc`) and confirm the page renders with defaults instead of crashing.

### Build — the architecture, with enforcement
Set up the feature folders, write a one-paragraph `README.md` in each top-level directory, and add the `no-restricted-imports` rule. Then **try to violate it** — import `features/payments/components/PaymentsTable` directly from `features/webhooks` and confirm the lint fails.

### Break — four experiments
1. **The waterfall.** Build a 3-level nested route fetching in `useEffect` at each level. Open the Network panel and look at the request waterfall. Convert to loaders and compare.
2. **State in the wrong place.** Put filters in `useState` in a shared parent. Type in the search box with "highlight updates" on and watch the table and chart flash. Move it to the URL.
3. **History pollution.** Update the URL with `push` on every keystroke, type a word, then hold the back button. Count the entries.
4. **Unvalidated params.** Read `?limit` with `Number(params.get("limit"))` and pass `NaN` to your query. Then add the schema.

### Explain out loud (60 seconds)
1. Why the URL is a state container, and what belongs there.
2. The fetch waterfall, and how loaders fix it.
3. Feature-based structure and the two rules that keep it alive.
4. How you'd make a modal linkable, and what that buys.

---

## Module 4 complete — checkpoint

- [ ] The configuration trap, and the test before adding a prop
- [ ] Compound vs slots vs render props
- [ ] The controlled/uncontrolled pattern and its subtleties
- [ ] What a correct Modal needs beyond rendering
- [ ] Controlled vs uncontrolled inputs, and the modern default
- [ ] Where validation runs, and which layer is security
- [ ] Five accessibility requirements for a form field
- [ ] Three layers of double-submit prevention
- [ ] What error boundaries catch and don't
- [ ] Boundary placement rules for errors and Suspense
- [ ] Why transitions prevent fallback flashes
- [ ] What portals preserve and escape
- [ ] Why the URL is the best state container
- [ ] The fetch waterfall and loaders
- [ ] Feature-based architecture and how to enforce it

---

## What's next

Module 5 is data — the biggest practical shift in how modern React apps are written. It starts by separating two things most codebases conflate: **server state and client state**, and why `useEffect` + `fetch` is the wrong default.

Next → **[Lesson 17: Server state vs client state](../05-data/17-server-state.md)**
