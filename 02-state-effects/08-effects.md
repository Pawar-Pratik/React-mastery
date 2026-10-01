# Lesson 08 — Effects: synchronization, not lifecycle

> **Why this lesson exists:** `useEffect` is the most-used and most-misused hook in React. The reason isn't that effects are hard — it's that almost everyone learns them as "componentDidMount, but with hooks", which is the wrong model and produces wrong code. The right model is **synchronization**, and once you have it, most of the effects in a typical codebase turn out not to be needed at all. This is the highest-leverage lesson in the track.

**Time:** ~85 minutes · **Prereq:** Lessons 05, 07

---

## 1. The idea in one sentence

> **An effect synchronizes React state with something *outside* React — and if there's no external system involved, you almost certainly don't need one.**

Not "run this after render." Not "run this on mount." **Synchronize.**

---

## 2. The lifecycle model is wrong

```tsx
// ❌ The mental model most people carry
useEffect(() => { /* componentDidMount */ }, []);
useEffect(() => { /* componentDidUpdate */ });
useEffect(() => () => { /* componentWillUnmount */ }, []);
```

This model *appears* to work and then produces bugs that make no sense within it: effects running twice, cleanups firing at odd times, dependency arrays that feel arbitrary.

**The correct model:**

> An effect describes **a connection between your component and an external system**, and React's job is to keep that connection correct as your state changes. The effect *sets up* the connection; the cleanup *tears it down*. React runs them as often as needed to keep reality matching your description.

```tsx
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);   // set up
  connection.connect();
  return () => connection.disconnect();                      // tear down
}, [serverUrl, roomId]);
```

Read that as a **description**, not a schedule: *"while this component is on screen, there should be a connection to `roomId` on `serverUrl`."*

- Mount → connect
- `roomId` changes → disconnect from the old, connect to the new
- Unmount → disconnect

**You never wrote "on mount" or "on update".** You described the desired synchronized state and React worked out the transitions — which is the exact same shift as the imperative-to-declarative move from [Lesson 01](../01-mental-model/01-how-react-works.md), applied to external systems instead of the DOM.

### Dependencies are not a trigger list
```tsx
useEffect(() => { ... }, [roomId]);
```
This does **not** mean "run when `roomId` changes." It means *"this effect's description depends on `roomId` — re-synchronize if it differs."* The distinction matters, because it tells you how to fill in the array: **you don't choose the dependencies; the effect body determines them.** Every reactive value the body reads must be in there, or React is synchronizing against stale information.

---

## 3. You probably don't need an effect

This is the section that changes how you write React. **Six cases**, and they cover most of the effects in a typical codebase.

### Case 1 — Transforming data for rendering
```tsx
// ❌ An extra render, and the UI briefly shows the wrong value
const [payments, setPayments] = useState<Payment[]>([]);
const [total, setTotal] = useState(0);
useEffect(() => {
  setTotal(payments.reduce((s, p) => s + p.money.amountMinor, 0));
}, [payments]);

// ✅ Just compute it
const total = payments.reduce((s, p) => s + p.money.amountMinor, 0);
```
The effect version renders twice (once with the old total, once with the new), so the user can see a stale number. **Derived values are computed during render, full stop.** If the computation is genuinely expensive, `useMemo` — after measuring ([Lesson 10](../03-hooks/10-memoization.md)).

### Case 2 — Handling user events
```tsx
// ❌ Which user action caused this? Impossible to tell from here.
useEffect(() => {
  if (refundSucceeded) {
    toast.success("Refund issued");
    analytics.track("refund_completed");
  }
}, [refundSucceeded]);

// ✅ The event handler knows exactly what happened
async function handleRefund() {
  const result = await api.createRefund(...);
  if (result.ok) {
    toast.success("Refund issued");
    analytics.track("refund_completed");
  }
}
```
**The test: was this caused by a user doing something, or by the component being displayed?** User action → event handler. Displayed → effect. Putting event logic in an effect loses the causal information and fires on any path that sets that state, including a cache rehydration.

### Case 3 — Resetting state when a prop changes
```tsx
// ❌ Renders twice; flashes the previous payment's data
useEffect(() => { setAmount(payment.money.amountMinor); setReason(null); }, [payment.id]);

// ✅ A different payment is a different component (Lesson 04)
<RefundForm key={payment.id} payment={payment} />
```

### Case 4 — Adjusting state when a prop changes
```tsx
// ❌
useEffect(() => { setSelection(null); }, [items]);

// ✅ Adjust during render — React restarts before committing, so there's no flash
const [prevItems, setPrevItems] = useState(items);
if (items !== prevItems) {
  setPrevItems(items);
  setSelection(null);
}
```
Calling `setState` during render of the *same* component is documented and supported: React throws away the in-progress render and restarts with the new state, **before touching the DOM.** No extra commit, no visible stale frame.

### Case 5 — Sharing logic between handlers
```tsx
// ❌ Fires whenever the state happens to be set — including on page load from a cached value
useEffect(() => { if (selectedIds.length > 0) prefetchDetails(selectedIds); }, [selectedIds]);

// ✅ Extract a function and call it from both handlers
function selectPayments(ids: PaymentId[]) { setSelectedIds(ids); prefetchDetails(ids); }
```

### Case 6 — Chains of effects
```tsx
// ❌ Four renders, and the intermediate states are visible
useEffect(() => { if (payment) setCustomer(payment.customer); }, [payment]);
useEffect(() => { if (customer) setAddress(customer.address); }, [customer]);
useEffect(() => { if (address) setTax(computeTax(address)); }, [address]);

// ✅ Compute everything in one place
const customer = payment?.customer ?? null;
const address = customer?.address ?? null;
const tax = address ? computeTax(address) : null;
```
**An effect that sets state which triggers another effect is always a smell.** Each link is a render, the intermediate frames are visible, and the whole chain re-runs from the top on any change.

### And the big one — Case 7: fetching data
```tsx
// ❌ The default everyone writes, and it's missing five things
useEffect(() => {
  fetch(`/v1/payments/${id}`).then(r => r.json()).then(setPayment);
}, [id]);
```
Missing: loading state, error handling, **race-condition handling**, cleanup/cancellation, caching, deduplication, refetch-on-focus, and retry. [Lesson 17](../05-data/17-server-state.md) covers why a query library is the right answer. §6 below covers the race condition, because you must understand it even when a library handles it for you.

> **The summary to carry:** *"an effect is for synchronizing with an external system — the DOM outside React, the network, a subscription, a timer, browser APIs, a non-React library. If none of those are involved, the code belongs in render (derived values) or in an event handler (user actions)."*

---

## 4. StrictMode's double-invoke — what it's telling you

```tsx
useEffect(() => {
  console.log("connected");
  return () => console.log("disconnected");
}, []);

// Development with StrictMode:
// connected
// disconnected     ← React immediately unmounts and remounts
// connected
```

**This is not a bug, and "turn off StrictMode" is the wrong response.** React is simulating what happens when a component unmounts and remounts — which really occurs with fast Refresh, back/forward navigation with restored state, Suspense retries, and future features like Activity/offscreen.

**The check it's running:** *does setting up, tearing down, and setting up again leave the system in the same state as setting up once?* If yes, your effect is correctly synchronized. If no, your cleanup is wrong or missing.

```tsx
// ❌ Fails the check — two connections, one leaked
useEffect(() => { const c = createConnection(); c.connect(); }, []);

// ✅ Passes — teardown undoes setup exactly
useEffect(() => {
  const c = createConnection();
  c.connect();
  return () => c.disconnect();
}, []);
```

```tsx
// ❌ The double-fetch complaint. Usually harmless (GETs are idempotent),
//    but it reveals a missing cancellation, which is a real bug (§6).
useEffect(() => { fetch(url).then(r => r.json()).then(setData); }, [url]);

// ✅
useEffect(() => {
  const ac = new AbortController();
  fetch(url, { signal: ac.signal }).then(r => r.json()).then(setData)
    .catch(e => { if (e.name !== "AbortError") setError(e); });
  return () => ac.abort();
}, [url]);
```

> **The interview answer:** *"StrictMode double-invokes effects in development to check that setup and cleanup are symmetrical. If my effect breaks under it, the effect is wrong — it would also break on fast refresh or a remount. The right response is to write the cleanup, not to disable StrictMode."*

**The exception worth knowing:** genuinely non-idempotent, one-time actions — an analytics "page viewed" event, or a `POST`. Those shouldn't be in a mount effect at all. Analytics belongs in a route-change handler or a ref-guarded effect; a `POST` belongs in an event handler.

---

## 5. Cleanup: the half people skip

**Every effect that creates something must destroy it.** The checklist:

```tsx
useEffect(() => {
  const id = setInterval(tick, 1000);        return () => clearInterval(id);
  const id2 = setTimeout(fn, 500);            return () => clearTimeout(id2);
  window.addEventListener("resize", fn);      return () => window.removeEventListener("resize", fn);
  const es = new EventSource(url);            return () => es.close();
  const ws = new WebSocket(url);              return () => ws.close();
  const ro = new ResizeObserver(fn); ro.observe(el);  return () => ro.disconnect();
  const io = new IntersectionObserver(fn);    return () => io.disconnect();
  const raf = requestAnimationFrame(loop);    return () => cancelAnimationFrame(raf);
  const sub = store.subscribe(fn);            return () => sub.unsubscribe();
  const ac = new AbortController();           return () => ac.abort();
  document.body.style.overflow = "hidden";    return () => { document.body.style.overflow = ""; };
}, []);
```

**Cleanup order:** before every re-run of the effect, and on unmount. So with deps `[roomId]` and a change from `"a"` to `"b"`: cleanup(a) → setup(b). Never two active connections.

```tsx
// The cleanup that restores, not just removes — easy to get wrong
useEffect(() => {
  const prev = document.body.style.overflow;
  document.body.style.overflow = "hidden";
  return () => { document.body.style.overflow = prev; };   // ✅ restores the ORIGINAL
}, []);
```
Setting it back to `""` assumes nobody else set it. With two nested modals, the inner one's cleanup would unlock the page while the outer is still open. Capturing the previous value is the correct form.

---

## 6. Race conditions — the bug nobody handles

```tsx
// ❌ Type "abc" fast: three requests fire, and they can resolve in ANY order.
useEffect(() => {
  fetch(`/v1/payments?q=${query}`).then(r => r.json()).then(setResults);
}, [query]);
```

```
user types:  a ────► request A (slow, 800ms)
             ab ───► request B (fast, 100ms)  → resolves → setResults(B) ✅
             abc ──► request C (fast, 120ms)  → resolves → setResults(C) ✅
                     request A resolves at 800ms → setResults(A) ❌

Final UI: results for "a" while the input says "abc".
```

**This is a real bug users hit constantly** and almost nobody handles in hand-rolled fetching. Two fixes:

```tsx
// ✅ Fix 1 — the ignore flag (works everywhere, no cancellation)
useEffect(() => {
  let ignore = false;
  fetch(`/v1/payments?q=${query}`)
    .then(r => r.json())
    .then(data => { if (!ignore) setResults(data); });     // ← stale responses are discarded
  return () => { ignore = true; };                          // cleanup on the NEXT keystroke
}, [query]);

// ✅ Fix 2 — AbortController (also cancels the request, saving bandwidth)
useEffect(() => {
  const ac = new AbortController();
  fetch(`/v1/payments?q=${query}`, { signal: ac.signal })
    .then(r => r.json()).then(setResults)
    .catch(e => { if (e.name !== "AbortError") setError(e); });
  return () => ac.abort();
}, [query]);
```

**Why the cleanup solves it:** React runs the previous effect's cleanup *before* the next effect's setup. So when `query` changes from `"ab"` to `"abc"`, the `"ab"` effect's cleanup sets `ignore = true` (or aborts), and when its response eventually arrives it's discarded.

> **This is the single best demonstration that "effects are synchronization, not lifecycle."** Under the lifecycle model, a cleanup that sets a boolean makes no sense. Under the synchronization model it's obvious: the old synchronization is being torn down, so its results are no longer wanted.

`AbortController` is better when the request is expensive (it actually cancels), the ignore flag is simpler and works with any async API. **A query library does both for you** ([Lesson 17](../05-data/17-server-state.md)) — which is most of why you should use one.

---

## 7. Dependencies, properly

### The rule
**Every reactive value the effect body reads must be a dependency.** Reactive = props, state, context, and anything derived from them during render. Not reactive: refs (`ref.current`), `dispatch`, `setState` functions, module-scope constants.

```tsx
// ✅ Trust the linter. It is essentially always right.
// eslint-plugin-react-hooks → exhaustive-deps
```

**Never silence `exhaustive-deps`.** If the linter wants a dependency you don't want to react to, the *effect* is wrong, not the linter. The fixes:

```tsx
// Problem: an object/function dependency that's new every render
function Chat({ options }: { options: { roomId: string } }) {
  useEffect(() => { connect(options); return () => disconnect(); }, [options]);
  //   ↑ `options` is a new object each render → reconnects every render
}

// ✅ Fix 1 — depend on primitives, not objects
useEffect(() => { connect({ roomId }); return () => disconnect(); }, [roomId]);

// ✅ Fix 2 — move the object inside the effect
useEffect(() => {
  const opts = { roomId, serverUrl };
  connect(opts); return () => disconnect();
}, [roomId, serverUrl]);

// ✅ Fix 3 — move the function outside the component entirely
// (if it doesn't use props/state, it isn't reactive)

// ✅ Fix 4 — the latest-ref pattern for callbacks you read but don't react to (Lesson 07)
const onMessageRef = useRef(onMessage);
useLayoutEffect(() => { onMessageRef.current = onMessage; });
useEffect(() => {
  const c = connect(roomId);
  c.on("message", m => onMessageRef.current(m));
  return () => c.disconnect();
}, [roomId]);                                    // ✅ honest deps, no reconnect on every render
```

**Fix 4 is the important one**, and it's precisely the "reads vs reacts to" distinction. The effect *reads* `onMessage` but shouldn't *re-synchronize* when it changes. React's proposed `useEffectEvent` formalises exactly this; until it ships, the ref pattern is the idiom.

### Empty dependencies
```tsx
useEffect(() => { ... }, []);
```
Legitimate when the effect genuinely reads nothing reactive — a subscription to a fixed URL, a global event listener, a one-time measurement. **If the linter complains about `[]`, you have a bug**, not a linter problem.

---

## 8. `useEffect` vs `useLayoutEffect` vs `useInsertionEffect`

```
commit DOM mutations
   │
   ├─ useInsertionEffect   ← CSS-in-JS libraries inject <style> here. You'll never write one.
   ├─ useLayoutEffect      ← runs, refs attached. BLOCKS PAINT.
   │
BROWSER PAINT              ← the user sees it
   │
   └─ useEffect            ← runs after paint
```

**Use `useLayoutEffect` only when the user would otherwise see a wrong frame:**
```tsx
// Positioning a tooltip — measuring in useEffect causes a visible jump
useLayoutEffect(() => {
  const rect = triggerRef.current!.getBoundingClientRect();
  setPosition({ top: rect.bottom + 8, left: rect.left });
}, [open]);
```
Legitimate cases: measuring and repositioning, scroll restoration, and preventing a flash of incorrect content. **Everything else is `useEffect`**, because layout effects delay the paint and make interactions feel slower.

> **The SSR caveat:** `useLayoutEffect` doesn't run on the server and React warns about it during SSR. Either guard it (`typeof window !== "undefined"`), use the common `useIsomorphicLayoutEffect` alias, or — better — question whether you need it.

---

## 9. Ledger Console's effects (the honest audit)

An entire dashboard needs surprisingly few. **This list is the proof of the lesson.**

```tsx
// ✅ Legitimate: an external system (SSE)
useEffect(() => {
  const es = new EventSource("/v1/events/stream");
  es.addEventListener("payment.succeeded", e => onEventRef.current(JSON.parse(e.data)));
  return () => es.close();
}, []);

// ✅ Legitimate: a browser API
useEffect(() => {
  function onKey(e: KeyboardEvent) { if (e.key === "/") { e.preventDefault(); searchRef.current?.focus(); } }
  window.addEventListener("keydown", onKey);
  return () => window.removeEventListener("keydown", onKey);
}, []);

// ✅ Legitimate: syncing with the document (outside React)
useEffect(() => { document.title = `${count} payments · Ledger`; }, [count]);

// ✅ Legitimate: a third-party library instance (Lesson 07)
useEffect(() => { const c = new Chart(ref.current!); return () => c.destroy(); }, []);

// ✅ Legitimate: debouncing a URL commit (an external system — the browser's history)
useEffect(() => {
  const t = setTimeout(() => setSearchParams(p => { p.set("q", draft); return p; }, { replace: true }), 300);
  return () => clearTimeout(t);
}, [draft, setSearchParams]);

// ❌ NOT effects in Ledger Console:
//    fetching payments        → TanStack Query (Lesson 17)
//    computing totals         → derived during render
//    showing a refund toast   → the event handler that issued the refund
//    resetting the form       → key={payment.id}
//    clearing selection       → adjust during render
```

**Five effects in a real dashboard.** If your app has forty, the majority are one of the six cases from §3.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **An effect synchronizes with an external system. No external system ⇒ no effect** | The single test that eliminates most effects |
| **Derived values are computed during render, never set in an effect** | An effect renders twice and flashes stale data |
| **User-caused logic belongs in event handlers** | An effect loses the causal information |
| **Reset state with `key`; adjust state during render** | No extra commit, no visible stale frame |
| **An effect that sets state read by another effect is a chain — collapse it** | Each link is a render with a visible intermediate state |
| **Every effect that creates something destroys it in cleanup** | StrictMode's double-invoke exists to catch this |
| **Cleanup restores the previous value, not a hardcoded default** | Nested modals unlock the page early otherwise |
| **Every async effect handles the race condition** | An ignore flag or `AbortController`; always |
| **Never silence `exhaustive-deps`** | If you don't want to react to a value, use the latest-ref pattern |
| **Depend on primitives, not objects and functions** | New references every render ⇒ the effect re-runs every render |
| **`useLayoutEffect` only when the user would see a wrong frame** | It blocks paint |
| **Don't disable StrictMode. Fix the effect** | It's telling you the cleanup is wrong |
| **Don't fetch in an effect — use a query library** | Race conditions, caching, dedup, retry, focus-refetch |

---

## 11. Interview traps

**Q1. "What is `useEffect` for?"**
Not "running code after render." *"Synchronizing React state with an external system — the DOM outside React, the network, subscriptions, timers, browser APIs, non-React libraries. The effect sets up the synchronization and the cleanup tears it down; React runs them as needed to keep the connection correct. If no external system is involved, the code belongs in render or an event handler."*

**Q2. "Name four cases where you don't need an effect."**
Transforming data (compute during render), handling user events (event handler), resetting state on a prop change (`key`), adjusting state on a prop change (during render), sharing logic between handlers (extract a function), and chains of effects (compute in one place). **Naming five or six is a strong signal** — it's the most impactful list in React.

**Q3. "Why does my effect run twice?"**
StrictMode in development, deliberately. React unmounts and remounts to verify that setup → cleanup → setup leaves the system unchanged. If it breaks, the cleanup is missing or wrong — and it would also break on fast refresh or a real remount. **Never** the answer: disable StrictMode.

**Q4. "Search-as-you-type shows results for an old query. What happened?"**
A race condition: multiple in-flight requests resolving out of order, and the last one to *resolve* wins rather than the last one to be *sent*. Fix with a cleanup that sets an `ignore` flag or calls `AbortController.abort()`. **Then the deeper point:** this is why a cleanup that sets a boolean makes sense — the old synchronization is being torn down, so its result is no longer wanted.

**Q5. "How do you decide what goes in the dependency array?"**
You don't decide — the body does. Every reactive value it reads goes in. If the linter wants something you don't want to react to, the effect is wrong: depend on primitives, move the object inside the effect, hoist non-reactive functions out, or use the latest-ref pattern for callbacks you read but don't re-synchronize on.

**Q6. "My effect reconnects on every render. Why?"**
An object or function dependency that's a new reference each render. Fix by depending on primitives, constructing the object inside the effect, or the latest-ref pattern.

**Q7. "`useEffect` vs `useLayoutEffect`?"**
Layout effects run in the commit phase **before paint**, so they block it — use them only when the user would otherwise see a wrong frame (tooltip positioning, scroll restoration, flash prevention). `useEffect` runs after paint and is the default. Also: layout effects don't run during SSR and React warns about it.

**Q8. "Should you fetch data in `useEffect`?"**
You can, and you'd have to hand-write loading state, error handling, race-condition guards, cancellation, caching, deduplication, retry, and refetch-on-focus. **That's a library's job.** Effects are for synchronization; server data is a cache with different requirements ([Lesson 17](../05-data/17-server-state.md)).

**Q9. "What's the cleanup function for?"**
Undoing what setup did — so React can re-synchronize when dependencies change and tear down on unmount. It runs **before every re-run** and on unmount, so there's never an overlap. The mental check: *"if React set this up, tore it down, and set it up again, would the system be in the same state?"*

**Q10. "How do you do something exactly once, like a page-view analytics event?"**
Acknowledge the tension: StrictMode will double-invoke a mount effect, and future features may remount legitimately. So a truly-once action shouldn't be in a mount effect. Options: fire it from a route-change handler, guard with a ref (accepting that it's per-instance), or do it server-side. **Recognising that "exactly once on mount" isn't a guarantee React offers is the answer.**

---

## 12. Build & break

### Build — the effect audit
Open any React project you have and list every `useEffect`. For each, classify it:
```
[ ] Legitimate — synchronizes with an external system (which one?)
[ ] Case 1 — derived value → compute during render
[ ] Case 2 — user event → event handler
[ ] Case 3 — reset on prop change → key
[ ] Case 4 — adjust on prop change → during render
[ ] Case 5 — shared handler logic → extract a function
[ ] Case 6 — chain → collapse
[ ] Case 7 — data fetching → query library
```
**Count the percentage that are legitimate.** In most codebases it's 20–40%. That number is the lesson, and it's a great thing to be able to quote from your own experience.

### Build — the race condition, then fix it
```tsx
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<string[]>([]);

  useEffect(() => {
    // Simulate variable latency so slow responses land last
    const delay = query.length === 1 ? 2000 : 200;
    let cancelled = false;
    setTimeout(() => { if (!cancelled) setResults([`results for "${query}"`]); }, delay);
    // return () => { cancelled = true; };     ← comment this out first
  }, [query]);

  return (<><input value={query} onChange={e => setQuery(e.target.value)} />
           <p>{results[0]}</p></>);
}
```
Type "abc" quickly **without** the cleanup: the display ends up showing results for `"a"`. Uncomment the cleanup and repeat. **Then rewrite it with `AbortController` and a real fetch** so you've done both forms.

### Build — the synchronization model, felt
```tsx
function Room({ roomId }: { roomId: string }) {
  useEffect(() => {
    console.log(`✅ connect ${roomId}`);
    return () => console.log(`❌ disconnect ${roomId}`);
  }, [roomId]);
  return <div>{roomId}</div>;
}
```
Toggle `roomId` between "general" and "support" a few times and read the log. **You'll see it always disconnects before connecting, and never holds two connections.** That ordering *is* the synchronization model.

### Break — five experiments
1. **StrictMode.** Write an effect that creates a `setInterval` with no cleanup. Watch it double under StrictMode, then navigate away and back and watch the console accelerate.
2. **The chain.** Build three effects where each sets state read by the next. Count the renders in the Profiler. Collapse them into derived values and count again.
3. **Object dependency.** `useEffect(..., [{ roomId }])` — it re-runs every render. Change to `[roomId]`.
4. **Silenced linter.** Remove a dependency and add `// eslint-disable-next-line`. Change that value and watch the effect use stale data. Then fix it properly with the latest-ref pattern.
5. **`useLayoutEffect` cost.** Put an expensive loop in a `useLayoutEffect` vs a `useEffect` and watch the difference in when the page paints.

### Explain out loud (2 minutes)
1. What an effect is for — the synchronization sentence.
2. Six cases where you don't need one.
3. What StrictMode's double-invoke is checking.
4. The race condition, and why a cleanup solves it.
5. How you decide dependencies, and what to do when the linter and your intent disagree.

---

## Module 2 complete — checkpoint

Cold, no notes:

- [ ] The snapshot model; why three `setCount(count+1)` give 1
- [ ] When you must use the updater form
- [ ] Why mutation doesn't re-render
- [ ] Store IDs, derive objects; don't store what you can derive
- [ ] The state-location decision tree (including the URL)
- [ ] Why you never sync props to state with an effect
- [ ] The four signals to move to `useReducer`
- [ ] Why switching on state first makes double-submit impossible
- [ ] `useRef` vs `useState`, with the decision test
- [ ] The latest-ref pattern and what it solves
- [ ] The four rules for wrapping an imperative library
- [ ] What an effect is actually for
- [ ] Six cases where you don't need one
- [ ] What StrictMode's double-invoke checks
- [ ] The race condition and its fix
- [ ] `useEffect` vs `useLayoutEffect`

---

## What's next

Module 3 covers the rest of the hooks properly — starting with the one that causes the most accidental performance problems in real apps: context, and why a single changed value can re-render half your tree.

Next → **[Lesson 09: Context, properly](../03-hooks/09-context.md)**
