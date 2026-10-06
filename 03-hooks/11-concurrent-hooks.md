# Lesson 11 — The concurrent hooks

> **Why this lesson exists:** concurrent React is the biggest change since hooks, and most developers know only that `useTransition` "makes things not block." The actual mechanism — **interruptible rendering with priorities** — is what makes a search box responsive over 50,000 rows without debouncing, virtualizing, or a web worker. These hooks are also where React's answer to external state (`useSyncExternalStore`) and SSR-safe IDs (`useId`) live, and both come up in interviews.

**Time:** ~65 minutes · **Prereq:** Lessons 03, 10

---

## 1. The idea in one sentence

> **Concurrent React can start rendering, pause to handle something more urgent, and resume or discard that work — so you can mark an update as *low priority* and keep the UI responsive while it happens.**

This is what Fiber's interruptible tree ([Lesson 01](../01-mental-model/01-how-react-works.md)) was built for.

---

## 2. The problem: one slow render blocks everything

```tsx
function Search() {
  const [query, setQuery] = useState("");
  const results = filterHugeList(query);       // 50,000 items, ~200ms

  return (<>
    <input value={query} onChange={e => setQuery(e.target.value)} />
    <ResultList results={results} />
  </>);
}
```

Every keystroke: `setQuery` → render → 200ms of filtering → commit → paint. **The input is frozen for 200ms per character.** Typing "payment" is an unusable 1.4 seconds of lag.

**The old fixes and what each costs:**

| Fix | Cost |
|---|---|
| Debounce | The input updates instantly but results lag by a fixed delay, even when fast. You've traded correctness of feel for a constant |
| Throttle | Same, plus dropped updates |
| Web worker | Real, and a lot of machinery for a filter |
| Virtualize | Helps the *render*, not the 200ms of filtering |

**The concurrent fix:** tell React the results are *less urgent than the input*. React renders the input immediately and works on the results in the background, abandoning that work if another keystroke arrives.

---

## 3. `useTransition`

```tsx
function Search() {
  const [query, setQuery] = useState("");
  const [deferredQuery, setDeferredQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    setQuery(e.target.value);                       // URGENT — the input must update now
    startTransition(() => {
      setDeferredQuery(e.target.value);             // TRANSITION — interruptible
    });
  }

  const results = filterHugeList(deferredQuery);

  return (<>
    <input value={query} onChange={onChange} />
    <ResultList results={results} style={{ opacity: isPending ? 0.6 : 1 }} />
  </>);
}
```

What React does:
1. `setQuery` is urgent → renders and commits immediately. **The input never lags.**
2. `setDeferredQuery` is a transition → React starts rendering it in the background.
3. Another keystroke arrives mid-render → React **throws away** the in-progress work and starts again with the new value.
4. `isPending` is `true` while a transition is in flight, so you can show a subtle stale/loading state.

**The crucial difference from debouncing:** with a debounce, you *wait* a fixed time regardless of how fast the machine is. With a transition, React starts immediately and only abandons work if something more urgent arrives — so on a fast machine the results appear instantly, and on a slow one the input still never blocks. **You get the best of both, adaptively.**

### React 19: async transitions ("Actions")
```tsx
const [isPending, startTransition] = useTransition();

function handleRefund() {
  startTransition(async () => {
    const result = await api.createRefund(payment.id, { amountMinor }, key);
    if (!result.ok) setError(result.error);
    else setRefund(result.value);
  });
}
// isPending stays true for the whole async operation — no manual isSubmitting state
```
React 19 lets transitions be async, which makes `isPending` a built-in form-submission state. That's the foundation of `useActionState` and `<form action>` ([Lesson 14](../04-composition/14-forms.md)).

### What transitions are *not* for
```tsx
// ❌ Don't put controlled-input updates in a transition — the input WILL lag
startTransition(() => setQuery(e.target.value));

// ❌ Don't use it to make a genuinely slow API call feel fast — it doesn't speed anything up
```
**Transitions reprioritise rendering, not network or CPU work.** A 200ms filter still takes 200ms; it just doesn't block the input. An interviewer asking "does `useTransition` make it faster?" is checking exactly this — **no, it makes it non-blocking.**

---

## 4. `useDeferredValue`

The simpler form: defer a *value* rather than wrapping the update.

```tsx
function Search() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);      // lags behind during heavy work
  const isStale = query !== deferredQuery;

  const results = useMemo(() => filterHugeList(deferredQuery), [deferredQuery]);

  return (<>
    <input value={query} onChange={e => setQuery(e.target.value)} />
    <div style={{ opacity: isStale ? 0.6 : 1 }}><ResultList results={results} /></div>
  </>);
}
```

React renders with the *old* `deferredQuery` first (fast, urgent), then re-renders with the new value at low priority.

| | `useTransition` | `useDeferredValue` |
|---|---|---|
| You control | The **update** (you wrap the setState) | The **value** (you read a lagging copy) |
| Use when | You own the state update | The value comes from props, or you can't wrap the setter |
| Pending signal | `isPending` | `value !== deferredValue` |

```tsx
// The common case for useDeferredValue: the value is a PROP
function Results({ query }: { query: string }) {
  const deferred = useDeferredValue(query);          // ← you don't own the setState
  return <ExpensiveList query={deferred} />;
}
```

**The pairing that makes it work:** `useDeferredValue` + `memo` on the expensive child. Without `memo`, the child re-renders with the new value anyway during the urgent pass, and you've deferred nothing.

```tsx
const ExpensiveList = memo(function ExpensiveList({ query }: { query: string }) { ... });
```
That dependency is the detail people miss, and it's a good follow-up question.

---

## 5. `useSyncExternalStore`

The hook that makes external state correct under concurrent rendering. You'll rarely call it directly — you'll use Zustand, Jotai, Redux or Valtio, all of which are built on it — but knowing what it solves is genuinely useful.

```tsx
const width = useSyncExternalStore(
  subscribe,      // (onChange) => unsubscribe
  getSnapshot,    // () => current value, must be referentially stable if unchanged
  getServerSnapshot,  // () => value for SSR/hydration
);

function subscribe(onChange: () => void) {
  window.addEventListener("resize", onChange);
  return () => window.removeEventListener("resize", onChange);
}
const getSnapshot = () => window.innerWidth;
const getServerSnapshot = () => 1024;           // a sensible default on the server
```

**The problem it solves — "tearing":** with concurrent rendering, React can pause mid-render. If an external store changes during that pause, some components render with the old value and some with the new — **one screen showing two different versions of the same data.** `useSyncExternalStore` forces a synchronous re-render of everything subscribed, so the whole tree is consistent.

The old `useState` + `useEffect` subscription pattern can tear:
```tsx
// ❌ Can tear under concurrent rendering, and misses updates between render and effect
function useWindowWidth() {
  const [w, setW] = useState(window.innerWidth);
  useEffect(() => {
    const on = () => setW(window.innerWidth);
    window.addEventListener("resize", on);
    return () => window.removeEventListener("resize", on);
  }, []);
  return w;
}
```

**The rule that catches everyone:** `getSnapshot` must return a **referentially stable** value when nothing changed, or React re-renders infinitely.
```tsx
// ❌ Infinite loop — a new object every call
const getSnapshot = () => ({ width: window.innerWidth, height: window.innerHeight });

// ✅ Cache it, and only create a new object when the values actually change
let cached = { width: 0, height: 0 };
const getSnapshot = () => {
  if (cached.width !== window.innerWidth || cached.height !== window.innerHeight) {
    cached = { width: window.innerWidth, height: window.innerHeight };
  }
  return cached;
};
```
(`useSyncExternalStoreWithSelector` from `use-sync-external-store/shim/with-selector` handles this for you, which is what library authors use.)

```tsx
// Genuinely useful direct uses
const isOnline = useSyncExternalStore(
  cb => { window.addEventListener("online", cb); window.addEventListener("offline", cb);
          return () => { window.removeEventListener("online", cb); window.removeEventListener("offline", cb); }; },
  () => navigator.onLine,
  () => true,                        // assume online during SSR
);

const prefersDark = useSyncExternalStore(
  cb => { const mq = matchMedia("(prefers-color-scheme: dark)"); mq.addEventListener("change", cb);
          return () => mq.removeEventListener("change", cb); },
  () => matchMedia("(prefers-color-scheme: dark)").matches,
  () => false,
);
```

---

## 6. `useId`

```tsx
function Field({ label, ...rest }: FieldProps) {
  const id = useId();
  return (<>
    <label htmlFor={id}>{label}</label>
    <input id={id} aria-describedby={`${id}-hint`} {...rest} />
    <span id={`${id}-hint`}>Must be at least 8 characters</span>
  </>);
}
```

**What it solves:** generating IDs that are **stable across server and client rendering**. `Math.random()` or an incrementing counter produce different values on each side, which causes a hydration mismatch ([Lesson 25](../07-server-react/25-ssr-streaming-hydration.md)).

Rules:
- **One `useId` per component, then derive suffixes** (`${id}-hint`, `${id}-error`) rather than calling it repeatedly.
- **Not for list keys.** It's per *component instance*, not per item, and keys must come from data ([Lesson 04](../01-mental-model/04-keys-and-identity.md)).
- The format (`:r1:` etc.) is deliberately not a valid CSS selector — don't build selectors from it; use `document.getElementById`.

**Accessibility is the real reason this hook exists:** `htmlFor`/`id` pairs, `aria-describedby`, `aria-labelledby`, and `aria-controls` all need unique, stable IDs, and a reusable component can't hardcode them.

---

## 7. `use` (React 19)

```tsx
function PaymentDetail({ paymentPromise }: { paymentPromise: Promise<Payment> }) {
  const payment = use(paymentPromise);       // suspends until resolved
  return <div>{payment.id}</div>;
}

<Suspense fallback={<Skeleton />}><PaymentDetail paymentPromise={promise} /></Suspense>
```

`use` unwraps a promise or reads a context, and — uniquely among hooks — **it can be called conditionally and inside loops**, because it isn't stateful in the usual sense.

```tsx
function Panel({ showDetails, promise }: Props) {
  if (showDetails) {
    const data = use(promise);               // ✅ legal — unlike useContext
    return <Details data={data} />;
  }
  return <Summary />;
}
```

> **The caveat to state honestly:** creating a promise *during render* in a Client Component is a mistake — it's a new promise each render, so it never resolves stably and you get an infinite suspend loop. Promises passed to `use` should come from a cache, a Server Component, or a framework loader. For client-side data fetching, a query library remains the right answer ([Lesson 17](../05-data/17-server-state.md)).

---

## 8. Putting it together — Ledger Console

```tsx
function PaymentsPage() {
  const [filters, setFilters] = useState<Filters>(initialFilters);
  const deferredFilters = useDeferredValue(filters);
  const isStale = filters !== deferredFilters;

  const { data } = usePayments(deferredFilters);      // query keyed on the DEFERRED value

  return (
    <>
      <FilterBar value={filters} onChange={setFilters} />   {/* instant */}
      <div style={{ opacity: isStale ? 0.6 : 1, transition: "opacity 150ms" }}>
        <VirtualPaymentsTable rows={data ?? []} />           {/* lags, doesn't block */}
      </div>
    </>
  );
}
```

```tsx
// Tab switching with a transition — the tab highlights instantly, content streams in
function Tabs() {
  const [tab, setTab] = useState<"payments" | "refunds" | "payouts">("payments");
  const [isPending, startTransition] = useTransition();

  return (<>
    <nav>{TABS.map(t => (
      <button key={t} aria-selected={tab === t} onClick={() => startTransition(() => setTab(t))}>
        {LABELS[t]}
      </button>
    ))}</nav>
    <div style={{ opacity: isPending ? 0.6 : 1 }}>{renderTab(tab)}</div>
  </>);
}
// Without the transition: clicking a tab freezes for as long as the new tab takes to render,
// and the clicked tab doesn't even highlight until then. With it: instant feedback.
```

**That tab example is the most convincing demo of transitions**, because the broken version is so obviously bad — the button you clicked doesn't respond until the whole page is ready.

```tsx
// Filter state also lives in the URL, so the transition wraps the navigation
const [isPending, startTransition] = useTransition();
function applyFilter(next: Filters) {
  startTransition(() => setSearchParams(toParams(next)));
}
```

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Never put a controlled input's own state in a transition** | The input will lag — that's the one update that must be urgent |
| **`useTransition` when you own the setState; `useDeferredValue` when the value is a prop** | Same mechanism, different handle |
| **`useDeferredValue` requires `memo` on the expensive child** | Otherwise it re-renders during the urgent pass and you've deferred nothing |
| **Show a pending signal (`isPending` / `value !== deferred`)** | Otherwise stale content looks like a bug |
| **Prefer a transition to a debounce for render-bound work** | Adaptive: instant on fast machines, non-blocking on slow ones |
| **Debounce is still right for *network* work** | Transitions don't reduce request count |
| **`getSnapshot` must be referentially stable** | A new object each call is an infinite render loop |
| **Always provide `getServerSnapshot` for SSR** | Or hydration throws |
| **One `useId` per component; derive suffixes** | It's per instance, and never for list keys |
| **Don't create promises during render for `use`** | A new promise each render never settles |

---

## 10. Interview traps

**Q1. "What is concurrent React?"**
Interruptible rendering with priorities: React can start a render, pause to handle something more urgent, then resume or **discard** that work. Enabled by Fiber's traversable tree. It's the machinery behind `useTransition`, `useDeferredValue`, Suspense and streaming SSR.

**Q2. "Does `useTransition` make things faster?"**
**No** — it makes them non-blocking. A 200ms filter still takes 200ms; the difference is that the input stays responsive because React prioritises it and can abandon the in-progress low-priority render. This is the most common misunderstanding.

**Q3. "`useTransition` vs debouncing?"**
Debounce waits a fixed time regardless of machine speed — so on a fast machine you've added artificial lag, and on a slow one you may still block. A transition starts immediately and only abandons work when something more urgent arrives: instant when it can be, non-blocking when it can't. **Debounce is still correct for reducing *network requests***, which transitions don't do.

**Q4. "`useTransition` vs `useDeferredValue`?"**
Same mechanism, different handle. `useTransition` wraps the *update* (you own the setState and get `isPending`). `useDeferredValue` gives you a lagging *value* (use it when the value arrives as a prop). Both need the expensive consumer to be `memo`'d for the deferral to actually help.

**Q5. "What is tearing, and what solves it?"**
Under concurrent rendering React can pause mid-render; if an external store updates during the pause, some components render with the old value and some with the new — one screen, two versions of the same data. `useSyncExternalStore` forces a synchronous, consistent re-render of all subscribers. It's why every modern state library is built on it.

**Q6. "Why does my `useSyncExternalStore` loop forever?"**
`getSnapshot` returns a new object each call, so React always sees a change. Cache the snapshot and only create a new object when the underlying values change — or use `useSyncExternalStoreWithSelector`.

**Q7. "What's `useId` for, and why not `Math.random()`?"**
Stable IDs across server and client render, for `htmlFor`, `aria-describedby`, `aria-labelledby`. Random or counter-based IDs differ between server and client and cause a hydration mismatch. **It's an accessibility hook first**, not a general ID generator — and never for list keys.

**Q8. "How would you make a search over 50,000 rows feel instant?"**
Layered, and name them in order: (1) urgent state for the input, deferred for the results, so typing never blocks; (2) `memo` the result list so the deferral works; (3) virtualize so only visible rows render; (4) an `isStale` opacity so the user sees it's catching up; (5) if the filtering is genuinely heavy, move it to a worker or the server. **Mentioning that a transition doesn't reduce the work, only its blocking, is the mark of understanding it.**

**Q9. "When does `use` beat `useEffect` for data?"**
When the promise comes from a Server Component, a framework loader, or a cache — then `use` + Suspense gives declarative loading with no `isLoading` state. **Not** for promises created during a client render; those never settle. For general client-side fetching, a query library still wins.

**Q10. "Can `use` be called conditionally?"**
Yes — uniquely among hooks, because it isn't stateful in the positional sense. That's why `use(SomeContext)` inside an `if` is legal where `useContext` isn't.

---

## 11. Build & break

### Build — the blocking demo (do this one; it's the most convincing)
```tsx
const ROWS = Array.from({ length: 50_000 }, (_, i) => ({ id: i, name: `Payment ${i}` }));

function Slow() {
  const [query, setQuery] = useState("");
  const results = ROWS.filter(r => r.name.includes(query));    // ~100ms
  return (<>
    <input value={query} onChange={e => setQuery(e.target.value)} />
    <p>{results.length} results</p>
    <ul>{results.slice(0, 100).map(r => <li key={r.id}>{r.name}</li>)}</ul>
  </>);
}
```
1. Type quickly. **The input lags visibly.** Throttle your CPU 4× in DevTools to make it dramatic.
2. Add `useDeferredValue` + `memo` on the list. Type again — the input is instant.
3. Remove the `memo` and observe that the deferral stops helping. **That's the dependency people miss.**
4. Now try it with a 300ms debounce and compare the *feel* on a fast machine. The debounced version has a constant lag the transition version doesn't.

### Build — the tab transition
Build three tabs where one renders 5,000 rows. Without a transition, clicking that tab freezes the UI and the button doesn't even highlight. Add `startTransition` around `setTab` plus an `isPending` opacity. **The difference is dramatic and it's the demo to have ready for an interview.**

### Build — `useSyncExternalStore` directly
Implement `useOnlineStatus`, `usePrefersDark` and `useMediaQuery(query)` with all three arguments. Then deliberately return a new object from `getSnapshot` and watch the infinite loop, so you've seen the failure mode.

### Break — four experiments
1. **Input in a transition.** Wrap `setQuery` itself in `startTransition` and watch the input become laggy. That's why it must stay urgent.
2. **Missing `getServerSnapshot`.** Use `useSyncExternalStore` in an SSR app without it and read the error.
3. **`useId` as a key.** Use it for list keys and watch identity break when the list reorders.
4. **A promise created in render.** `use(fetch(url).then(r => r.json()))` inside a Client Component. Watch it suspend forever.

### Explain out loud (90 seconds)
1. What concurrent rendering is, and what it does *not* speed up.
2. Transition vs debounce, and when each is right.
3. `useTransition` vs `useDeferredValue`, and the `memo` requirement.
4. What tearing is and what solves it.
5. Why `useId` exists.

---

## What's next

You've now seen every built-in hook that matters. The last piece of Module 3 is how to package them: custom hooks that genuinely compose, the rules that make them safe, and the abstractions that look clever and aren't.

Next → **[Lesson 12: Custom hooks](12-custom-hooks.md)**
