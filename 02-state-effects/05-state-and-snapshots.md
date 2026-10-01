# Lesson 05 — State, snapshots & batching

> **Why this lesson exists:** every "my state is wrong" bug in React comes from one misunderstanding — treating state like a mutable variable instead of **a snapshot frozen into each render's closures**. Once you see that, `setCount(count+1)` three times giving `1`, stale values in timers, and state being "one render behind" all stop being mysteries and become the same single mechanism.

**Time:** ~65 minutes · **Prereq:** Lesson 03

---

## 1. The idea in one sentence

> **A render is a snapshot: props, state and every function defined during that render capture the values at that moment and never see any later value.**

State isn't a variable you read. It's a value that was baked into this render.

---

## 2. The snapshot model

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    console.log(count);          // 0 — always 0 in this render
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

**Result after one click: `1`, not `3`.**

Why, mechanically: `count` is a `const` in *this* render's scope. In the render where `count` is `0`, all three calls compute `0 + 1 = 1`. Then React re-renders, `Counter()` runs again, and a **new** `count` const is created with the value `1`.

```tsx
// What React effectively does:
// Render #1:  const count = 0;  handleClick captures 0
// Render #2:  const count = 1;  a NEW handleClick captures 1
```

There is no "current value" to read. **There is only this render's value.** That single idea explains everything below.

### The fix: the updater function
```tsx
setCount(c => c + 1);    // ×3 → 3
```
An updater receives the **pending** state — the value after any earlier queued updates in this batch. React processes the queue in order: `0 → 1 → 2 → 3`.

```tsx
// Mixing forms behaves exactly as the queue suggests:
setCount(count + 1);      // queue: [replace with 1]
setCount(c => c + 1);     // queue: [replace with 1, add 1]  → 2
setCount(42);              // queue: [..., replace with 42]  → 42
```

> **The rule to internalise:** *"if the next state depends on the previous state, use the updater form."* It's correct under batching, correct in async callbacks, correct in effects, and it removes the value from your dependency arrays ([Lesson 08](08-effects.md)). Not a style preference — a correctness rule.

---

## 3. Where stale closures actually hurt

The snapshot model is harmless in event handlers. It bites when a function **outlives its render**.

```tsx
// ❌ The classic broken timer
function Timer() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);       // `count` is 0, captured on mount, FOREVER
    }, 1000);
    return () => clearInterval(id);
  }, []);                         // empty deps → the effect runs once → the closure never updates
  return <h1>{count}</h1>;        // stuck at 1
}

// ✅ The updater has no closure dependency
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);                           // correctly empty — nothing from the render is captured
```

```tsx
// ❌ Stale value in an async callback
async function handleSave() {
  await api.save(draft);
  console.log(draft);            // the draft from THIS render, not whatever the user typed since
}

// ✅ If you need the latest, read a ref (Lesson 07)
const draftRef = useRef(draft);
draftRef.current = draft;
async function handleSave() {
  await api.save(draftRef.current);
}
```

```tsx
// ❌ Stale in a subscription
useEffect(() => {
  socket.on("message", m => setMessages([...messages, m]));   // `messages` frozen at mount
  return () => socket.off("message");
}, []);

// ✅
useEffect(() => {
  const handler = (m: Message) => setMessages(prev => [...prev, m]);
  socket.on("message", handler);
  return () => socket.off("message", handler);
}, []);
```

**The pattern across all three: any callback that survives past its render must either use the updater form or read from a ref.** That's the entire rule.

---

## 4. Immutability, and why it's non-negotiable

```tsx
const [payments, setPayments] = useState<Payment[]>([]);

// ❌ Mutation — React compares with Object.is, the reference is unchanged, nothing re-renders
payments.push(newPayment);
setPayments(payments);

// ✅ New reference
setPayments(prev => [...prev, newPayment]);
```

React's bailout check is `Object.is(oldState, newState)`. Mutating and passing the same reference means React sees "no change" and skips the render — and then it *appears* to work later when something else triggers a render, which is far worse than failing consistently.

```tsx
// The operations, correctly:
setItems(prev => [...prev, item]);                                   // append
setItems(prev => [item, ...prev]);                                   // prepend
setItems(prev => prev.filter(i => i.id !== id));                     // remove
setItems(prev => prev.map(i => i.id === id ? { ...i, note } : i));   // update one
setItems(prev => prev.toSorted((a, b) => a.n - b.n));                // sort (ES2023 — non-mutating)
setItems(prev => prev.with(2, newItem));                             // replace at index (ES2023)

// Nested updates — and this is where it gets ugly
setState(prev => ({
  ...prev,
  filters: { ...prev.filters, status: [...prev.filters.status, "failed"] },
}));
```

> **`toSorted`, `toReversed`, `toSpliced` and `with` (ES2023)** are the non-mutating array methods. They remove the most common accidental-mutation bug — `arr.sort()` mutates in place, which is a genuine footgun when the array came from state or props.

**When nesting gets painful, that's a signal**, not a formatting problem: either flatten the state, split it into multiple `useState` calls, or move to `useReducer` ([Lesson 06](06-reducers-and-state-machines.md)). Reaching for Immer is reasonable for genuinely deep state, but deep state is usually a modelling smell.

---

## 5. Structuring state: five rules

### Rule 1 — Group state that changes together
```tsx
// ❌ Three states that always change together → three chances to get it wrong
const [x, setX] = useState(0);
const [y, setY] = useState(0);
const [dragging, setDragging] = useState(false);

// ✅
const [drag, setDrag] = useState({ x: 0, y: 0, dragging: false });
```

### Rule 2 — Never duplicate state
```tsx
// ❌ Two sources of truth that will disagree
const [payments, setPayments] = useState<Payment[]>([]);
const [selectedPayment, setSelectedPayment] = useState<Payment | null>(null);
// Update a payment in the list → `selectedPayment` is now stale

// ✅ Store the ID; derive the object
const [selectedId, setSelectedId] = useState<PaymentId | null>(null);
const selected = payments.find(p => p.id === selectedId) ?? null;
```
**Store IDs, derive objects.** This is the most useful state-structuring rule there is, and it's the same normalisation instinct as a database schema.

### Rule 3 — Don't store what you can derive
```tsx
// ❌ Three states, two derivable, all able to drift
const [items, setItems] = useState<Payment[]>([]);
const [count, setCount] = useState(0);
const [total, setTotal] = useState(0);

// ✅ One state, two derived values
const [items, setItems] = useState<Payment[]>([]);
const count = items.length;
const total = items.reduce((s, i) => s + i.money.amountMinor, 0);
```
*"But isn't recomputing on every render slow?"* — almost never. `reduce` over a few hundred items is microseconds. **Only memoize after measuring** ([Lesson 10](../03-hooks/10-memoization.md)). Derived state that can drift is a correctness bug; recomputation is at worst a performance one.

### Rule 4 — Avoid redundant, contradictory state
```tsx
// ❌ 8 representable states, 3 meaningful (TS Lesson 09)
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<string | null>(null);
const [data, setData] = useState<Payment[] | null>(null);

// ✅ 3 states, each carrying exactly its data
const [state, setState] = useState<Async<Payment[]>>({ status: "idle" });
```

### Rule 5 — Put state as low as possible; lift only when shared
```
Only this component needs it?     → useState here
Two siblings need it?              → lift to their nearest common parent
Many components, rarely changing?  → Context (Lesson 09)
Many components, often changing?   → an external store with selectors (Lesson 09)
It's server data?                  → a query cache, NOT useState (Lesson 17)
It should survive a refresh / be shareable?  → the URL (Lesson 16)
```

**The URL row is the one people forget**, and it's the highest-value habit in this list: filters, sort order, pagination, the selected tab and open modals usually belong in the URL. You get shareable links, working back/forward, and refresh-survival for free — and, as [Lesson 03](../01-mental-model/03-rendering-and-commit.md) showed, it also removes a whole re-render cascade.

---

## 6. `useState` mechanics you should know

```tsx
// Lazy initialiser — the function runs ONCE, on mount
const [state, setState] = useState(() => expensiveInit());   // ✅
const [state2, setState2] = useState(expensiveInit());        // ❌ runs on EVERY render
```
The second form calls `expensiveInit()` on every render and throws the result away. A silent performance bug — and reading `localStorage` in an initialiser is the common real-world case.

```tsx
// Storing a function in state needs the updater form, or it gets CALLED
const [fn, setFn] = useState(() => initialFn);      // lazy init, stores initialFn
setFn(() => newFn);                                  // updater returning newFn
```

```tsx
// The same-value bailout
setStatus("idle");   // already "idle" → React may skip the re-render
setUser({ ...user }); // new object → NEVER bails out, even if the contents match
```

```tsx
// Hook order must be stable — this is why the rules of hooks exist
if (cond) { const [x] = useState(0); }     // ❌ hooks are stored positionally on the fiber
```
Hooks are a linked list on the fiber, matched **by call order**, not by name. A conditional hook shifts every subsequent hook's identity — so `useState` #2 starts reading #3's value. That's the mechanical reason behind the rule, and it's a good interview answer.

---

## 7. Derived state and the "sync props to state" trap

```tsx
// ❌ The most common React anti-pattern
function PaymentForm({ payment }: { payment: Payment }) {
  const [amount, setAmount] = useState(payment.money.amountMinor);
  useEffect(() => {
    setAmount(payment.money.amountMinor);        // "keep state in sync with props"
  }, [payment]);
  // Problems: an extra render after paint, a visible flash of stale data,
  // and it silently discards the user's edits whenever the parent re-renders.
}

// ✅ Option A — don't copy props into state at all; derive
const amount = draft ?? payment.money.amountMinor;

// ✅ Option B — reset via key (Lesson 04). Best when you genuinely need local draft state.
<PaymentForm key={payment.id} payment={payment} />

// ✅ Option C — adjust during render (rare, but React supports it explicitly)
function List({ items }: { items: Payment[] }) {
  const [selection, setSelection] = useState<PaymentId | null>(null);
  const [prevItems, setPrevItems] = useState(items);
  if (items !== prevItems) {                      // ← comparing during render is allowed
    setPrevItems(items);
    setSelection(null);
  }
  // React restarts the render immediately, before committing. No flash, no effect.
}
```

**Option C looks illegal and isn't.** Calling `setState` during render of the *same* component is a documented pattern: React discards the in-progress render and restarts with the new state, before touching the DOM. It's strictly better than an effect for this job — no extra commit, no paint with stale data. Knowing it exists is a genuine differentiator, and the key reset (Option B) is still cleaner when it applies.

---

## 8. Ledger Console's state model

```tsx
// URL — shareable, bookmarkable, survives refresh, avoids cascades
//   ?status=succeeded&created[gte]=2026-01-01&sort=-created_at&cursor=...

// Server cache (TanStack Query, Lesson 17) — payments, customers, balance
//   NOT useState. Server data has different needs: caching, revalidation, dedup.

// Local component state — drafts, open/closed, hover, the search input before debounce
const [draft, setDraft] = useState("");

// Reducer — multi-step flows with legal/illegal transitions (Lesson 06)
const [refund, dispatch] = useReducer(refundReducer, { step: "idle" });

// External store — auth session, feature flags, toasts (Lesson 09)
const principal = useAuthStore(s => s.principal);
```

**The decision tree, written down once:**
```
Is it server data?                → query cache
Should it survive a refresh or be shareable?  → URL
Do many distant components read it?           → store (with selectors) or context
Do two siblings need it?                      → lift to the common parent
Otherwise                                     → local useState
```
Being able to recite that tree — and having applied it — is what "knows how to structure a React app" actually means.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Use the updater form when the next state depends on the previous** | Correct under batching, in async callbacks, and in effects |
| **Never mutate state** | React compares references; mutation means "nothing changed" |
| **Prefer `toSorted`/`toReversed`/`with` over the mutating versions** | `arr.sort()` on a state array is a real bug |
| **Store IDs, derive objects** | Two copies of the same entity will disagree |
| **Don't store what you can derive** | Derived state drifts; recomputation is usually microseconds |
| **Model async state as a discriminated union** | 3 states, not 8 combinations |
| **State as low as possible; lift only when shared** | Removes re-render cascades for free |
| **Filters, sort, pagination, tabs → the URL** | Shareable, back/forward works, refresh-safe |
| **Lazy initialiser for expensive initial state** | `useState(f())` calls `f` on every render |
| **Never copy props into state and sync with an effect** | Extra render, visible flash, discards user edits. Use a key or derive |
| **Never call hooks conditionally** | They're matched positionally on the fiber |
| **Any callback that outlives its render uses an updater or a ref** | Otherwise it reads a frozen snapshot |

---

## 10. Interview traps

**Q1. "`setCount(count + 1)` three times — what happens?"**
Result is `1`. All three read `count` from the same render's closure, so each computes `0 + 1`. **Distinguish the two mechanisms explicitly:** batching is why there's one re-render; closures are why the value is 1. Fix with `setCount(c => c + 1)`.

**Q2. "Why is my state one render behind?"**
It isn't — you're reading this render's snapshot. `console.log(count)` right after `setCount` shows the old value because `count` is a `const` in the current scope and the new value only exists in the *next* render. There is no "current state" to read.

**Q3. "Why does my `setInterval` counter stick at 1?"**
The effect ran once with `count === 0` captured, and the interval callback keeps that closure forever. Fix with the updater form (which captures nothing), or a ref. **This is the canonical stale-closure question.**

**Q4. "When do you use the updater form?"**
Whenever the next state depends on the previous — always in async callbacks, intervals, subscriptions and effects. Bonus: it removes the state value from your effect's dependency array, which is often what makes an effect correct with `[]`.

**Q5. "Why can't you mutate state?"**
React bails out with `Object.is(old, new)`. A mutated array has the same reference, so React sees no change and skips the render. Worse, it may *appear* to work when an unrelated render happens — inconsistent failure is harder to debug than consistent failure.

**Q6. "How do you decide where state lives?"**
The §8 decision tree: server data → query cache; shareable/refresh-surviving → URL; many distant readers → store with selectors; two siblings → lift; otherwise local. **Naming the URL as a state location is what separates this from a textbook answer.**

**Q7. "How do you reset state when a prop changes?"**
`key` on the component (Lesson 04) — reset happens during render with no extra commit. If you need finer control, adjust state during render by comparing with a previous-value state. **Never** an effect that calls `setState` — it renders twice, flashes stale data, and blows away user edits.

**Q8. "Why can't hooks be called conditionally?"**
They're stored as a positional linked list on the fiber and matched by call order, not name. Skipping one shifts every subsequent hook's identity, so `useState` #2 starts reading #3's slot.

**Q9. "Is deriving values on every render a performance problem?"**
Almost never — a `filter`/`reduce` over hundreds of items is microseconds. Derived state that can drift is a *correctness* bug; recomputation is at worst a performance one. Memoize only after the Profiler says so.

**Q10. "What's `useState(() => init())` vs `useState(init())`?"**
The lazy initialiser runs once on mount; the second form runs on **every** render and discards the result. It matters when the initialiser reads `localStorage`, parses JSON, or does real work.

---

## 11. Build & break

### Build — the snapshot lab
```tsx
function Lab() {
  const [n, setN] = useState(0);
  return (
    <>
      <p>{n}</p>
      <button onClick={() => { setN(n + 1); setN(n + 1); setN(n + 1); }}>direct ×3</button>
      <button onClick={() => { setN(c => c + 1); setN(c => c + 1); setN(c => c + 1); }}>updater ×3</button>
      <button onClick={() => { setN(n + 1); alert(n); }}>set then alert</button>
      <button onClick={() => setTimeout(() => alert(n), 3000)}>alert in 3s</button>
    </>
  );
}
```
Predict each before clicking. The last one is the important one: click it, then click "updater ×3" twice, and wait. **The alert shows the value from when you clicked, not the current one.** That's the snapshot, made visible.

### Build — Ledger's state decisions
For each piece of Console state, write down where it lives and why:
```
search query (while typing) · applied filters · sort order · current page cursor
selected payment id · refund modal open · refund amount draft · payments list
auth session · toast queue · table column widths · sidebar collapsed
```
Then check against §8's tree. **The interesting ones are "column widths" (localStorage) and "sidebar collapsed" (localStorage, not URL — it's a preference, not a view).**

### Break — five experiments
1. **Mutation.** `items.push(x); setItems(items);` — nothing happens. Then click something else and watch it suddenly appear. That inconsistency is the danger.
2. **The stuck timer.** Reproduce §3's broken interval. Fix it with the updater. Then fix it with a ref and compare.
3. **The sync-props-to-state flash.** Build a form that copies a prop into state with an effect. Type in it, then trigger a parent re-render. Watch your input get wiped.
4. **Eager initialiser.** `useState(JSON.parse(localStorage.getItem("x") ?? "{}"))` with a `console.log` inside. Count the calls per render. Switch to lazy.
5. **`sort()` on state.** `setItems(items.sort(...))` — it mutates *and* returns the same reference, so it doesn't render. Use `toSorted`.

### Explain out loud (90 seconds)
1. The snapshot model in one sentence.
2. Why three `setCount(count+1)` calls give 1 — both mechanisms.
3. When you must use the updater form.
4. Where state should live, as a decision tree.
5. Why you never sync props to state with an effect.

---

## What's next

`useState` handles independent values. When state has **rules** — legal transitions, multiple fields that must change together, an undo stack — you want a reducer. And a reducer plus a discriminated union is how you make illegal UI states impossible.

Next → **[Lesson 06: useReducer & state machines](06-reducers-and-state-machines.md)**
