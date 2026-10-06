# Lesson 10 — memo, useMemo, useCallback

> **Why this lesson exists:** memoization is where React developers waste the most effort for the least benefit. Codebases get wrapped in `useCallback` and `useMemo` "for performance" by people who never measured, which adds code, adds dependency arrays to get wrong, and frequently makes things *slower*. This lesson gives you the mechanism, the three reasons memoization silently fails, the cases where it's genuinely required for correctness, and an honest account of what the React Compiler changes.

**Time:** ~65 minutes · **Prereq:** Lessons 03, 09

---

## 1. The idea in one sentence

> **Memoization trades memory and comparison cost for skipped work — and it only pays off when the skipped work is genuinely expensive *and* the comparison genuinely succeeds, which is rarer than people assume.**

---

## 2. The three tools

```tsx
// memo — skip re-rendering a component if its props are shallow-equal
const Row = memo(function Row({ payment }: { payment: Payment }) { ... });

// useMemo — cache a computed VALUE between renders
const total = useMemo(() => payments.reduce((s, p) => s + p.money.amountMinor, 0), [payments]);

// useCallback — cache a FUNCTION between renders
const handleRefund = useCallback((id: PaymentId) => refund(id), [refund]);
// useCallback(fn, deps) === useMemo(() => fn, deps)
```

All three do the same thing: **compare dependencies with `Object.is`; if unchanged, reuse last time's result.**

They serve two distinct purposes, and conflating them is where the confusion starts:

| Purpose | Tool | Question |
|---|---|---|
| **Skip expensive work** | `useMemo`, `memo` | Is the computation/render actually slow? |
| **Preserve referential identity** | `useCallback`, `useMemo` | Does something downstream compare by reference? |

**The second is the more common legitimate reason**, and it's about *correctness of the optimisation*, not about the memo itself being fast.

---

## 3. Why memoization silently fails

### Failure 1 — An unstable prop defeats `memo`
```tsx
const Row = memo(RowImpl);

function Table({ payments }: { payments: Payment[] }) {
  return payments.map(p => (
    <Row
      key={p.id}
      payment={p}
      onSelect={() => select(p.id)}          // ❌ new function every render
      style={{ padding: 8 }}                  // ❌ new object every render
      actions={["refund", "capture"]}          // ❌ new array every render
    />
  ));
}
// memo compares by reference → always different → memo never skips,
// AND you now pay for a shallow comparison on every row. Net loss.
```

**`memo` + any fresh object/array/function prop = strictly worse than no `memo`.** This is the most common memoization mistake in real codebases.

```tsx
// ✅ Stabilise every reference
const CELL_STYLE = { padding: 8 };                        // module scope — created once
const ACTIONS = ["refund", "capture"] as const;

function Table({ payments }: { payments: Payment[] }) {
  const onSelect = useCallback((id: PaymentId) => select(id), [select]);
  return payments.map(p => (
    <Row key={p.id} payment={p} onSelect={onSelect} style={CELL_STYLE} actions={ACTIONS} />
  ));
  // Note: onSelect now takes the id as an argument instead of closing over it —
  // that's what makes ONE stable callback work for every row.
}
```
That last comment is the technique people miss: **pass the id as an argument rather than closing over it**, so one callback serves all rows.

### Failure 2 — Memoizing something that was already cheap
```tsx
const doubled = useMemo(() => count * 2, [count]);       // ❌ the memo costs more than the multiply
const sorted = useMemo(() => [...items].sort(), [items]); // ✅ maybe — if items is large
```
`useMemo` isn't free: it stores the value and deps on the fiber, allocates an array, and runs `Object.is` per dependency on every render. **For trivial work, the memo is the more expensive half.**

### Failure 3 — The dependency changes every render anyway
```tsx
function Chart({ config }: { config: ChartConfig }) {          // caller passes a fresh object
  const processed = useMemo(() => transform(data, config), [data, config]);   // ❌ config always new
}
```
Fix at the source: memoize `config` in the parent, or depend on its primitive fields.

### Failure 4 — `memo` can't stop a context change
```tsx
const Panel = memo(function Panel() {
  const theme = useContext(ThemeContext);      // ❌ re-renders when theme changes, memo or not
});
```
`memo` compares props; context isn't a prop ([Lesson 09](09-context.md)).

---

## 4. When memoization is required for *correctness*

This is the part people miss. Sometimes it isn't an optimisation at all.

```tsx
// 1. An effect dependency — without useCallback this fetches on every render, forever
const fetchData = useCallback(async () => { ... }, [url]);
useEffect(() => { fetchData(); }, [fetchData]);

// 2. A context value — without useMemo, every consumer re-renders on every provider render
const value = useMemo(() => ({ principal, login, logout }), [principal, login, logout]);

// 3. An expensive object identity that other memos depend on
const options = useMemo(() => ({ roomId, serverUrl }), [roomId, serverUrl]);
useEffect(() => connect(options), [options]);

// 4. Referential stability for an external subscription
const selector = useCallback((s: Store) => s.payments.filter(p => p.status === status), [status]);
const payments = useStore(selector);      // a new selector each render would re-subscribe
```

**In these four cases, omitting the memo isn't slower — it's a bug** (an infinite effect loop, a re-render storm, a resubscribe loop). That distinction is worth stating explicitly in an interview.

> Though note for cases 1 and 3, the *better* fix is usually to remove the dependency entirely — move the function inside the effect, or depend on primitives ([Lesson 08](../02-state-effects/08-effects.md)). Memoizing to satisfy a dependency array is the second-best answer.

---

## 5. When memoization genuinely helps performance

```tsx
// ✅ 1. Genuinely expensive computation over a large dataset
const chartData = useMemo(() => aggregateByDay(payments), [payments]);   // 50k rows, ~30ms

// ✅ 2. A large subtree that re-renders often with the same props
const Chart = memo(RevenueChart);       // renders 5,000 SVG nodes

// ✅ 3. A list where each row is non-trivial and the list re-renders frequently
const Row = memo(PaymentRow);            // 200 rows × a few ms each

// ✅ 4. Preserving identity so a DOWNSTREAM memo can work
const onSelect = useCallback(fn, []);    // so memo(Row) actually skips
```

**The threshold in practice:** a component rendering in under ~1ms is not worth memoizing. The Profiler's "rendered for Xms" column is the number to look at ([Lesson 20](../06-performance/20-measuring.md)).

### The correct order of operations
```
1. Measure with the Profiler                        ← never skip
2. Is it actually slow? If not, STOP
3. Fix it structurally:  move state down · pass children · split components · virtualize
4. Only then memoize — and RE-MEASURE to confirm it helped
```

**Step 3 beats step 4 almost every time.** Moving state down eliminates a whole cascade; `memo` just puts a comparison in front of it. Passing `children` gives you the same skip for free, with no dependency array to get wrong.

---

## 6. The React Compiler

React 19 ships an optional compiler (formerly "React Forget") that **automatically memoizes** — it analyses your components and inserts the equivalent of `useMemo`/`useCallback`/`memo` where they're provably safe.

```tsx
// You write:
function Table({ payments, onSelect }) {
  const total = payments.reduce((s, p) => s + p.money.amountMinor, 0);
  const handle = (id) => onSelect(id);
  return <Row total={total} onSelect={handle} />;
}

// The compiler emits (conceptually): both `total` and `handle` cached and
// invalidated only when their inputs change, plus memoized JSX where safe.
```

**What it changes:**
- Manual `useMemo`/`useCallback` become largely unnecessary for *performance* purposes
- Less code, fewer dependency arrays to get wrong
- More consistent — it doesn't forget

**What it doesn't change, and this is the honest part:**
- It **requires your components to follow the Rules of React** — no mutation during render, no side effects in render, no reading refs during render. It bails out on code it can't prove safe, silently.
- `useMemo` for **correctness** (the four §4 cases) still matters conceptually, even if the compiler often covers it.
- It doesn't fix structural problems — bad state placement, missing virtualization, an unnecessary context.
- Adoption is gradual: `eslint-plugin-react-compiler` tells you which components it can and can't optimise.

> **The interview answer:** *"The React Compiler auto-memoizes, so manual `useMemo`/`useCallback` for performance is mostly going away. Two caveats: it only works on components that follow the Rules of React and bails out silently otherwise — so the lint rule matters — and it doesn't fix structural problems like state being too high in the tree or a missing virtualized list. I'd still understand memoization, because you need it to reason about why something re-renders."*

**Practically, today:** if the compiler is on, stop writing memoization by hand and let it work. If it isn't, measure first and memoize narrowly.

---

## 7. `memo` and custom comparison

```tsx
const Row = memo(RowImpl, (prev, next) => prev.payment.id === next.payment.id
                                       && prev.payment.version === next.payment.version);
```
The second argument returns `true` to **skip** the render — note the inverted sense from `Array.prototype.sort`-style comparators, which catches people.

**Use it very sparingly.** A custom comparator is a correctness risk: if you forget to compare a prop, the component shows stale data with no error. Deep equality is usually worse than the render you're skipping. If you find yourself needing one, the real fix is usually narrower props:
```tsx
// Instead of a custom comparator over a big object:
<Row payment={payment} />                    // whole object
<Row id={p.id} amount={p.money.amountMinor} status={p.status} />   // ✅ primitives compare cheaply
```

---

## 8. Ledger Console's memoization (the honest audit)

```tsx
// ✅ Required for correctness — context value
const auth = useMemo(() => ({ principal, login, logout }), [principal, login, logout]);

// ✅ Required for correctness — store selector identity
const rows = useStore(useCallback((s: Store) => s.filtered(status), [status]));

// ✅ Measured: 4M-row aggregation, 40ms
const chartSeries = useMemo(() => aggregateByDay(payments), [payments]);

// ✅ Measured: the virtualized row renders 20× per scroll frame
const VirtualRow = memo(PaymentRow);

// ✅ Stable callback so memo(VirtualRow) actually works — id passed as an argument
const onSelectRow = useCallback((id: PaymentId) => setSelectedId(id), []);

// ❌ NOT memoized in Ledger Console:
//    formatMoney(payment.money)        — microseconds
//    payments.length                    — trivial
//    every onClick in the settings page — the page re-renders rarely
//    the filter bar's derived label     — cheap, and re-renders once per keystroke anyway
```

**Five memoizations in a real dashboard**, three of which are for correctness or to enable another memo. If your codebase is wrapped in `useCallback` everywhere, most of it is doing nothing.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Measure before memoizing. Re-measure after** | Most re-renders are free; most memos don't help |
| **Fix structurally first: move state down, pass children, split, virtualize** | Eliminates the cascade instead of comparing your way through it |
| **`memo` + a fresh object/array/function prop is worse than no `memo`** | The comparison always fails, and you pay for it |
| **Pass IDs as arguments, not closures, so one callback serves a whole list** | Makes a single `useCallback` work for N rows |
| **Hoist constant objects/arrays to module scope** | `const STYLE = {...}` beats `useMemo(() => ({...}), [])` |
| **Memoize context values and effect-dependency functions — that's correctness** | Omitting them causes re-render storms and effect loops |
| **`memo` cannot prevent a context-driven re-render** | It compares props; context isn't a prop |
| **Avoid custom `memo` comparators** | Forgetting a prop shows stale data silently. Narrow the props instead |
| **Don't memoize sub-millisecond work** | The memo costs more than the work |
| **If the React Compiler is on, stop hand-memoizing and enable its lint rule** | It bails out silently on rule-breaking components |

---

## 10. Interview traps

**Q1. "`useMemo` vs `useCallback` vs `memo`?"**
`useMemo` caches a value, `useCallback` caches a function (`useCallback(fn, d) === useMemo(() => fn, d)`), `memo` skips a component render on shallow-equal props. All three compare deps with `Object.is`. **Then the framing:** two purposes — skipping expensive work, and preserving referential identity for something downstream.

**Q2. "`memo` didn't help. Why?"**
Three reasons, and name all three: (1) a new object/array/function prop each render so the comparison always fails; (2) the component consumes a context that changed — `memo` can't stop that; (3) `children` is a fresh element each render. Add that a failing `memo` is *worse* than none, because you pay for the comparison.

**Q3. "Should you wrap every callback in `useCallback`?"**
No. It costs code, an array to get wrong, and a comparison — and it only helps if something downstream compares by reference. **`useCallback` on a handler passed to a plain `<button>` does nothing at all.** With the React Compiler on, it's largely unnecessary.

**Q4. "When is memoization about correctness rather than performance?"**
Four cases: a context value (or every consumer re-renders on every provider render), a function in an effect's dependency array (or the effect loops forever), an object passed to an effect, and a selector passed to an external store (or it resubscribes each render). In these, omitting it is a bug — though the better fix is often to remove the dependency instead.

**Q5. "How do you decide what to memoize?"**
The four-step order: measure → is it actually slow → fix structurally → memoize and re-measure. **Naming the structural fixes** (move state down, pass children, split components, virtualize) is what separates a real answer from "use `memo`."

**Q6. "What does the React Compiler change?"**
It auto-memoizes, so manual performance memoization becomes unnecessary. Caveats: it requires components to follow the Rules of React and **bails out silently** otherwise (hence the lint rule), and it doesn't fix structural problems. Understanding memoization still matters for reasoning about re-renders.

**Q7. "Is `useMemo` free?"**
No — it stores the value and deps on the fiber, allocates, and runs `Object.is` per dependency every render. For trivial work the memo is the expensive half. And remember it's a **hint, not a guarantee**: React may discard memoized values (it's documented as being allowed to, e.g. for offscreen content).

**Q8. "One stable callback for 200 rows — how?"**
Pass the row's id as an **argument** rather than closing over it: `useCallback((id) => select(id), [])` with `onClick={() => onSelect(p.id)}` in the row… except that inner arrow is itself fresh. The cleanest version: give the row the `payment` and the stable `onSelect`, and let the row call `onSelect(payment.id)` internally. Then the row's props are `{payment, onSelect}`, both stable per row.

**Q9. "Why is a custom `memo` comparator risky?"**
Forgetting to compare a prop means the component renders stale data with no error, and deep equality often costs more than the render. Prefer narrowing the props to primitives so the default shallow comparison works.

**Q10. "Your app re-renders the whole page on every keystroke. Walk me through it."**
Profile to confirm, then look for the cause rather than reaching for `memo`: the input's state is too high in the tree (move it down), an unmemoized context value, or the expensive subtree isn't passed as `children`. **Structural fix first**, and — for a search input specifically — debounce the commit to the URL or query ([Lesson 03](../01-mental-model/03-rendering-and-commit.md)).

---

## 11. Build & break

### Build — the memo lab
```tsx
const Expensive = memo(function Expensive({ n, onClick }: { n: number; onClick: () => void }) {
  console.log("render Expensive");
  let x = 0; for (let i = 0; i < 5_000_000; i++) x += i;    // simulate cost
  return <button onClick={onClick}>{n} {x}</button>;
});

function App() {
  const [count, setCount] = useState(0);
  const [other, setOther] = useState(0);
  const onClick = () => console.log("clicked");            // ← unstable, deliberately
  return (<>
    <button onClick={() => setOther(o => o + 1)}>other {other}</button>
    <Expensive n={count} onClick={onClick} />
  </>);
}
```
1. Click "other". `Expensive` re-renders despite `memo` — the unstable `onClick`.
2. Wrap `onClick` in `useCallback(..., [])`. It stops.
3. Add `style={{ margin: 4 }}` as a prop. It breaks again.
4. Hoist the style to module scope. Fixed.
5. Add `useContext(ThemeContext)` inside `Expensive` and change the theme. **It re-renders even though props are identical.** That's the context limitation, felt.

### Build — measure the threshold
Build a list of 500 rows where each row does a small amount of work. Profile it three ways: no memoization, `memo` on the row with an unstable callback, `memo` with a stable callback. **Record the three numbers.** Then reduce the work per row until `memo` stops mattering — that's your practical threshold, measured rather than assumed.

### Build — the structural fix comparison
Take a page with a search input and an expensive chart. Fix the keystroke re-render four ways and measure each:
1. `memo` the chart
2. Move the input's state into its own component
3. Pass the chart as `children`
4. Put the search in the URL with a debounce

Compare renders-per-keystroke and lines of code. **Options 2 and 3 usually win on both.**

### Break — four experiments
1. **Memo making it slower.** Memoize a trivial component with 10 props and render it 1,000 times. Compare with no `memo`.
2. **The effect loop.** Put an unmemoized function in an effect's deps. Watch it fetch forever.
3. **Unmemoized context.** Remove `useMemo` from a provider value and watch every consumer re-render on every provider render.
4. **Custom comparator.** Write one that forgets a prop, then change that prop and watch the UI show stale data with no warning.

### Explain out loud (60 seconds)
1. The two purposes of memoization.
2. Three reasons `memo` silently fails.
3. Four cases where memoization is correctness, not performance.
4. The four-step order of operations.
5. What the React Compiler does and doesn't fix.

---

## What's next

The hooks you've seen so far are the synchronous ones. Next: the concurrent hooks — `useTransition`, `useDeferredValue`, `useSyncExternalStore` and `useId` — which are how React keeps an app responsive while doing expensive work, and how external stores integrate correctly.

Next → **[Lesson 11: The concurrent hooks](11-concurrent-hooks.md)**
