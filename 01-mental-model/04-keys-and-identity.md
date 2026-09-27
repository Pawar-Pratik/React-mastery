# Lesson 04 — Keys, identity & lists

> **Why this lesson exists:** the single most confusing bug in React — *"my input lost focus"*, *"the wrong checkbox is ticked"*, *"my form has someone else's data"* — has one cause: **component identity**. And `key` is the only tool you have to control it. Most developers use `key={index}` because the warning goes away, never learn what it actually does, and then hit the bug in production where it looks like corruption rather than a React behaviour.

**Time:** ~55 minutes · **Prereq:** Lesson 03

---

## 1. The idea in one sentence

> **A component's identity is its *type* plus its *position among siblings* — and `key` overrides position. Same identity ⇒ React reuses the fiber, so state, refs and DOM survive. Different identity ⇒ React destroys and recreates, so state is lost.**

Every bug in this lesson follows from that sentence.

---

## 2. Identity is position, not props

```tsx
{isEditing ? <Input placeholder="Edit" /> : <Input placeholder="View" />}
```
Both branches render an `Input` in the **same position**, so React reuses the same fiber. The DOM node persists, the cursor position persists, the scroll position persists — only `placeholder` changes. That's usually what you want.

```tsx
{isEditing
  ? <div><Input /></div>
  : <span><Input /></span>}
```
Now the parent's *type* differs, so React destroys the whole subtree and mounts a fresh one. **The input's value is gone.** Same `Input`, same props, different identity.

**Neither behaviour is a bug** — but you need to know which one you're getting, because both surprise people at different times.

---

## 3. The lost-focus bug

The canonical React mystery. Look at this and predict what happens:

```tsx
function SearchPage() {
  const [query, setQuery] = useState("");

  function Results() {                              // ← declared INSIDE the component
    return <ul>{filter(query).map(r => <li key={r.id}>{r.name}</li>)}</ul>;
  }

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <Results />
    </>
  );
}
```

**The bug:** type one character and the input loses focus. Type another and it loses focus again.

**Why:** `Results` is a *new function object* on every render of `SearchPage`. React compares element `type` by reference. `Results !== Results` from the previous render, so it's a **different type** ⇒ destroy the old subtree, mount a new one. And because the whole subtree is remounted on every keystroke, the DOM nodes are replaced — and a replaced DOM node cannot hold focus.

```tsx
// ✅ Define components at module scope. Always.
function Results({ query }: { query: string }) {
  return <ul>{filter(query).map(r => <li key={r.id}>{r.name}</li>)}</ul>;
}
function SearchPage() {
  const [query, setQuery] = useState("");
  return (<><input value={query} onChange={e => setQuery(e.target.value)} /><Results query={query} /></>);
}
```

**Rule: never define a component inside another component.** Ever. It remounts the entire subtree on every parent render — destroying state, losing focus, restarting animations, re-running every effect, and killing performance. The same applies to components created inside `.map()` callbacks or memo factories.

> This is the highest-value five lines in the lesson. It's asked in interviews constantly, and when it happens in real code it looks like a browser bug rather than a React one.

---

## 4. Keys: what they actually do

Without keys, React matches children **by index**:

```
before: [<Row a/>, <Row b/>, <Row c/>]
after:  [<Row x/>, <Row a/>, <Row b/>, <Row c/>]

React's view (by index):
  0: a → x   UPDATE (props changed)
  1: b → a   UPDATE
  2: c → b   UPDATE
  3: —  → c  CREATE
```
Four operations, and — critically — **the fibers at positions 0–2 are reused**, so whatever state they held now belongs to different data.

With keys, React matches **by key**:
```
before: [a, b, c]  after: [x, a, b, c]
React's view:  "x" is new → CREATE.  "a","b","c" exist → MOVE.
```
One creation, three moves, and every row keeps its own state.

### The index-key bug, concretely

```tsx
// ❌ The bug that ships
{payments.map((p, i) => <PaymentRow key={i} payment={p} />)}
```

```tsx
function PaymentRow({ payment }: { payment: Payment }) {
  const [selected, setSelected] = useState(false);     // ← per-row state on the FIBER
  const [note, setNote] = useState("");
  return (
    <tr>
      <td><input type="checkbox" checked={selected} onChange={e => setSelected(e.target.checked)} /></td>
      <td>{payment.id}</td>
      <td><input value={note} onChange={e => setNote(e.target.value)} /></td>
    </tr>
  );
}
```

Now:
1. You tick row 2 (`pi_bbb`) and type a note in it.
2. A new payment arrives and is prepended.
3. `pi_bbb` is now at index 3. But the fiber at index 2 — holding `selected: true` and your note — is now rendered with `pi_ccc`'s data.

**The checkbox and the note have silently moved to the wrong payment.** No error. No warning. In a payments dashboard that's a user refunding the wrong transaction.

### When `key={index}` is actually fine
Three conditions, **all** of which must hold:
1. The list is **static** — never reordered, inserted into, or deleted from
2. Items have **no internal state** (no inputs, no checkboxes, no expanded/collapsed, no refs)
3. The list is **not filtered or sorted**

```tsx
// ✅ Genuinely fine
{["Home", "Payments", "Settings"].map((label, i) => <NavLink key={i}>{label}</NavLink>)}
```
**But use the value if it's unique** (`key={label}`) — it costs nothing and survives someone making the list dynamic later. *"Index keys are fine for static lists, and I still avoid them because lists rarely stay static"* is the right answer.

### What makes a good key
```tsx
key={payment.id}                              // ✅ stable, unique, from the data
key={`${date}-${index}`}                      // ⚠️ only if date+index is genuinely stable
key={JSON.stringify(item)}                     // ❌ expensive, and changes when any field does
key={Math.random()}                            // ❌ CATASTROPHIC — see below
key={index}                                    // ❌ for anything dynamic
```

```tsx
// The worst possible key:
{items.map(item => <Row key={Math.random()} item={item} />)}
```
A new key every render means **every row is destroyed and recreated on every render**. All state lost, all DOM replaced, all effects re-run, all animations restarted, and the performance is worse than not using React at all. It appears in real codebases because it makes the "unique key" warning go away.

**Key rules:**
- **Unique among siblings** (not globally — two different lists can both use `key="1"`)
- **Stable across renders** for the same logical item
- **Derived from the data**, not from the render
- Not readable by the component — `key` is React's, not a prop. If you need it, pass `id` separately.

### Keys on Fragments
```tsx
{rows.map(r => (
  <React.Fragment key={r.id}>        {/* shorthand <> can't take a key */}
    <tr>{r.a}</tr>
    <tr>{r.b}</tr>
  </React.Fragment>
))}
```

---

## 5. `key` as a deliberate reset — the underrated technique

Since a changed key destroys and recreates a component, you can use it **on purpose** to reset state.

```tsx
// ❌ The classic anti-pattern: syncing state with an effect
function PaymentForm({ payment }: { payment: Payment }) {
  const [amount, setAmount] = useState(payment.money.amountMinor);
  useEffect(() => {
    setAmount(payment.money.amountMinor);        // reset when the payment changes
  }, [payment.id]);                               // extra render, easy to get wrong, runs after paint
}

// ✅ Let identity do it
<PaymentForm key={payment.id} payment={payment} />
// A different payment ⇒ a different component ⇒ fresh state. No effect, no extra render.
```

This is one of the four "you don't need an effect" cases from [Lesson 08](../02-state-effects/08-effects.md), and it's the cleanest.

```tsx
// Other legitimate uses:
<Modal key={isOpen ? "open" : "closed"}>       // reset a modal's internal state each open
<Chart key={`${width}x${height}`} />            // force a re-init of a non-React chart lib
<ErrorBoundary key={retryCount}>               // reset an error boundary on retry
<Tab key={activeTab}>                          // each tab gets its own fresh state
```

> **The framing that makes it click:** *"`key` isn't just for lists — it's the general mechanism for telling React 'this is a different thing now.' Resetting state by changing a key is almost always better than resetting it in an effect, because it happens during render rather than after paint."*

**When *not* to:** if the component is expensive to mount (subscriptions, heavy DOM, a chart library init), a remount may cost more than a controlled reset. Measure.

---

## 6. List rendering in practice

```tsx
// The standard shape
{payments.map(p => <PaymentRow key={p.id} payment={p} />)}

// Empty and loading states belong OUTSIDE the map
{query.state === "loading" && <TableSkeleton rows={10} />}
{query.state === "success" && query.data.length === 0 && <EmptyState />}
{query.state === "success" && query.data.map(p => <PaymentRow key={p.id} payment={p} />)}
```

### Optimistic items need stable keys too
```tsx
// An optimistic refund exists before the server assigns an ID:
const optimistic = { id: `temp_${clientId}` as RefundId, ...input, pending: true };
// ✅ `clientId` is generated once per user action, so the key is stable
// ❌ `key={Date.now()}` would change on every render
```
This matters in [Lesson 18](../05-data/18-mutations-and-optimistic.md): when the server responds and the temp item is replaced by the real one, a stable key means React swaps the data in place rather than unmounting and remounting the row — which is the difference between a smooth update and a flicker.

### Grouped lists
```tsx
{Object.entries(byDate).map(([date, group]) => (
  <React.Fragment key={date}>
    <tr className="group-header"><td colSpan={4}>{date}</td></tr>
    {group.map(p => <PaymentRow key={p.id} payment={p} />)}
  </React.Fragment>
))}
```
Keys only need to be unique **within their own sibling set**, so the date keys and the payment keys don't interact.

### Virtualization changes the rules slightly
```tsx
const virtualizer = useVirtualizer({ count: rows.length, getScrollElement: () => ref.current,
                                     estimateSize: () => 48 });
{virtualizer.getVirtualItems().map(v => (
  <PaymentRow key={rows[v.index]!.id} payment={rows[v.index]!} style={{ transform: `translateY(${v.start}px)` }} />
))}
```
**Key on the data's ID, not on the virtual index** — the virtual window slides, so index-keying would make every row's state jump as you scroll. [Lesson 21](../06-performance/21-render-performance.md) covers virtualization properly.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Never define a component inside another component** | New function identity every render ⇒ full remount, lost focus and state |
| **`key` from stable data IDs, never the index, never random** | Index keys move state between items; random keys remount everything |
| **Keys unique among siblings, stable across renders** | That's the entire contract |
| **Use `key` to reset state deliberately** | Better than a sync effect — it happens during render |
| **`<React.Fragment key>` when you need a keyed fragment** | `<>` can't take a key |
| **Don't read `key` as a prop** | It's React's; pass `id` separately if the child needs it |
| **Key virtualized rows by data ID, not virtual index** | The window slides; state would follow the wrong rows |
| **Optimistic items need a stable client-generated ID** | So the real item replaces it in place, without a flicker |
| **Suspect identity first when state "moves" or focus is lost** | It's almost always identity, not a data bug |

---

## 8. Interview traps

**Q1. "Why is `key={index}` wrong?"**
Don't stop at "React needs unique keys." Give the mechanism and a consequence:
> *"React matches children by key to decide reuse vs recreate. With index keys, inserting at the front means every item's key shifts, so React reuses the fiber at each position with different data — and per-item state like a checked checkbox or a typed note stays with the position, not the item. In a payments table that means the user's selection silently moves to a different payment. It's fine only for a static, stateless, unsorted list."*

**Q2. "My input loses focus on every keystroke. Why?"**
A component defined inside another component. It's a new function object each render, so React sees a different `type`, destroys the subtree and mounts a new one — and a replaced DOM node can't keep focus. Move the component to module scope.

**Q3. "What determines whether React reuses a component's state?"**
**Type + position among siblings**, with `key` overriding position. Same identity ⇒ same fiber ⇒ state survives. Different identity ⇒ destroy and recreate.

**Q4. "How do you reset a form when the selected item changes?"**
`<Form key={item.id} item={item} />`. A changed key means a different component, so state resets automatically. Better than a `useEffect` that calls `setState`, because it happens during render rather than after paint, and there's no extra render or dependency to get wrong.

**Q5. "What happens with `key={Math.random()}`?"**
Every render produces new keys, so every child unmounts and remounts: all state lost, all DOM replaced, all effects re-run, animations restarted, and worse performance than plain DOM. It appears in real code because it silences the key warning.

**Q6. "Do keys need to be globally unique?"**
No — only among **siblings**. Two separate lists can both use `key="1"` with no conflict.

**Q7. "Can a component read its own `key`?"**
No. `key` is consumed by React and never appears in `props`. If the child needs the value, pass it separately as `id`.

**Q8. "When is `key={index}` acceptable?"**
Static list, no reordering or insertion, no per-item state or refs, no filtering or sorting. And even then, prefer a stable value from the data — lists rarely stay static, and the failure mode when it changes is silent.

**Q9. "How does keying interact with virtualization?"**
Key by the row's data ID, not the virtual index. The virtual window slides as you scroll, so index keys would make per-row state follow the *position in the viewport* rather than the row.

**Q10. "You see state 'jumping' between list items after a sort. Diagnose it."**
Index keys, essentially always. The sort reorders the data but the keys stay `0,1,2…`, so React reuses each position's fiber with different data — and any state on those fibers stays put. Fix: key by ID.

---

## 9. Build & break

### Build — the identity lab (do all four; each takes two minutes)

**1. Position determines identity**
```tsx
function App() {
  const [swap, setSwap] = useState(false);
  const a = <Counter label="A" />, b = <Counter label="B" />;
  return (<><button onClick={() => setSwap(s => !s)}>swap</button>
    {swap ? <>{b}{a}</> : <>{a}{b}</>}</>);
}
```
Increment both counters, then swap. **The counts stay in position, not with the labels** — because there are no keys. Add `key="a"`/`key="b"` and watch the counts travel with the labels.

**2. The index-key bug**
Render a list of 5 payments with a checkbox and a text input per row, keyed by index. Tick row 2, type in it, then prepend a new payment. **Your selection is now on the wrong payment.** Switch to `key={p.id}` and repeat.

**3. The lost-focus bug**
Reproduce §3 exactly. Type, lose focus, move the component to module scope, type again.

**4. `key` as a reset**
Build a form with local draft state. Add a list of items to select. Without a key, switching items keeps the old draft. Add `key={item.id}` and watch it reset. Then implement the same reset with a `useEffect` and compare — count the renders in the Profiler.

### Break — four experiments
1. **`key={Math.random()}`** on a list with inputs. Type in one. Watch it clear on the next render. Then check the Profiler — every row remounts every time.
2. **Duplicate keys.** Give two siblings the same key. Read React's warning, then observe the genuinely weird behaviour when you reorder them.
3. **Type change.** Wrap a stateful component in `{cond ? <div>…</div> : <section>…</section>}` and toggle. State resets.
4. **Read the key.** Try `props.key` inside a child. It's `undefined`, and React warns.

### Explain out loud (60 seconds)
1. What determines component identity.
2. Why a component defined inside another loses focus.
3. The index-key bug, with a concrete consequence.
4. Two legitimate uses of `key` beyond lists.

---

## Module 1 complete — checkpoint

Cold, no notes:

- [ ] What problem React solves, and the trade it makes
- [ ] What a component returns; why `<X/>` ≠ `X()`
- [ ] The three trees, and where state actually lives
- [ ] The two reconciliation heuristics
- [ ] Render phase vs commit phase, and why render must be pure
- [ ] The falsy-render trap (`{count && ...}`)
- [ ] Composition vs configuration, with the new-requirement test
- [ ] The four re-render triggers — and why "props changed" isn't one
- [ ] Why `memo` usually doesn't help
- [ ] Two structural fixes that beat `memo`
- [ ] What determines component identity
- [ ] The lost-focus bug and its cause
- [ ] The index-key bug, with a concrete consequence
- [ ] `key` as a deliberate reset

---

## What's next

Module 2 is where most React bugs live. It starts with state — and specifically with the idea that **every render is a snapshot**, which is the single explanation for every "stale value" bug you will ever hit.

Next → **[Lesson 05: State, snapshots & batching](../02-state-effects/05-state-and-snapshots.md)**
