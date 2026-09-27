# Lesson 03 — Rendering: render vs commit

> **Why this lesson exists:** *"why did this re-render?"* is the most-asked React question, and almost everyone answers it wrongly — usually with "because its props changed," which is **not how React works**. This lesson gives you the actual rules, which turns every performance investigation from guesswork into a two-minute diagnosis. It also kills the most damaging misconception in React: that re-rendering is bad.

**Time:** ~65 minutes · **Prereq:** Lesson 01

---

## 1. The idea in one sentence

> **React re-renders a component when its own state changes, when its parent re-renders, or when a context it consumes changes — *not* because "its props changed."**

Read that again. **Props changing is a *consequence* of the parent re-rendering, not a cause of the child re-rendering.** A component whose props are identical still re-renders if its parent did, unless you explicitly opt out with `memo`.

---

## 2. The four triggers, precisely

```
1. Its own state changed         setState / dispatch / useSyncExternalStore emitted
2. Its PARENT re-rendered        ← the one people get wrong
3. A context it consumes changed
4. A hook it uses forced it      (useSyncExternalStore, some library internals)
```

That's the complete list. There is no fifth.

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild />          {/* ← NO props at all. Re-renders anyway. */}
    </div>
  );
}
```

`ExpensiveChild` takes no props, so nothing about it changed — and it re-renders on every click, because **rule 2**. This surprises people, and it's the foundation of every React performance discussion.

### The cascade
```
Parent re-renders
  └─ every child element it creates is new
      └─ every child re-renders
          └─ every grandchild re-renders
              └─ … all the way down
```
**A state update near the root re-renders the entire subtree below it by default.** That's the behaviour; the question is whether it matters (§5).

### The `memo` bailout
```tsx
const ExpensiveChild = memo(function ExpensiveChild({ rows }: { rows: Payment[] }) {
  return <Table rows={rows} />;
});
```
`memo` inserts a check: *before* re-rendering, shallow-compare the new props with the old ones. If every prop is `Object.is`-equal, **skip this component and its entire subtree**.

The catch, and it's the reason most `memo` usage does nothing:
```tsx
function Parent() {
  const [count, setCount] = useState(0);
  const rows = payments.filter(p => p.status === "succeeded");   // ← NEW array every render
  const onSelect = (id: PaymentId) => console.log(id);            // ← NEW function every render
  return <ExpensiveChild rows={rows} onSelect={onSelect} />;      // memo compares by reference → always differs
}
```
**`memo` + a new object/array/function prop = `memo` does nothing**, plus you now pay for the comparison. Fixing that is [Lesson 10](../03-hooks/10-memoization.md); the point here is understanding *why*.

### The other bailout: same state value
```tsx
const [status, setStatus] = useState("idle");
setStatus("idle");        // same value (Object.is) → React bails out
```
React compares with `Object.is` and skips the re-render — **but not always on the first call**. React may still re-render that component once before bailing out, because the bailout is decided during the render. So don't rely on it for correctness; it's an optimisation, not a guarantee.

```tsx
// And this NEVER bails out, because it's a new object every time:
setUser({ ...user });          // Object.is is false → always re-renders
```

---

## 3. Render → commit, in detail

```
setState()
   │
   ├─ 1. SCHEDULE     React queues work at a priority. Does NOT render yet.
   │
   ├─ 2. RENDER PHASE (interruptible, pure)
   │      • call the component function → get elements
   │      • recurse into children
   │      • diff against the previous fiber tree
   │      • mark fibers with effect flags (Placement, Update, Deletion)
   │      ← may be paused, restarted, or thrown away entirely
   │
   ├─ 3. COMMIT PHASE (synchronous, uninterruptible)
   │      • before-mutation: getSnapshotBeforeUpdate
   │      • mutation: apply DOM changes; run useLayoutEffect CLEANUPS
   │      • layout: attach refs; run useLayoutEffect bodies   ← blocks paint
   │
   ├─ 4. BROWSER PAINT   ← the user finally sees the change
   │
   └─ 5. PASSIVE EFFECTS  useEffect cleanups, then useEffect bodies
```

Three consequences worth holding onto:

**A render doesn't always touch the DOM.** If a component re-renders and produces the same output, the diff finds nothing and the commit phase does nothing. **Rendering is cheap; DOM mutation is expensive.** That distinction is the whole of §5.

**`useLayoutEffect` runs before paint; `useEffect` after.** So a layout effect can measure and adjust the DOM with no visible flicker — and it *delays* the user seeing anything. That's why it's the exception, not the default ([Lesson 08](../02-state-effects/08-effects.md)).

**Refs are attached during the layout phase.** That's why `ref.current` is `null` during render and populated in effects.

---

## 4. Batching

```tsx
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  setName("x");
}
// ONE re-render, not three. React batches updates within the same tick.
```

**React 18 made this automatic everywhere.** Before 18, batching only happened inside React event handlers; updates in promises, `setTimeout` and native listeners each caused their own render.

```tsx
// React 17: 2 renders.  React 18+: 1 render.
async function save() {
  await api.save();
  setLoading(false);
  setSaved(true);
}
```

```tsx
// Opting out — you almost never want this
import { flushSync } from "react-dom";
flushSync(() => setOpen(true));      // forces a synchronous render + commit
inputRef.current?.focus();            // now the element definitely exists
```
Legitimate uses are narrow: focusing an element you just rendered, measuring immediately after a state change, or integrating with a third-party library that demands synchronous DOM. **`flushSync` disables batching and concurrent features for that update** — it's a performance escape hatch with a real cost, and using it casually is a review finding.

### The most common batching confusion
```tsx
const [count, setCount] = useState(0);
function handleClick() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
  console.log(count);            // 0 — state hasn't changed yet
}
// Result: 1, not 3.
```
`count` is `0` in this render's closure, so all three calls compute `0 + 1`. Fix with the updater form:
```tsx
setCount(c => c + 1);    // ×3 → 3   ← each receives the pending value
```
Full treatment in [Lesson 05](../02-state-effects/05-state-and-snapshots.md); it's listed here because people misattribute it to batching. **Batching is why there's one render; closures are why the value is 1.**

---

## 5. "Re-rendering is bad" — the misconception that causes bad code

**Re-rendering is how React works.** A re-render is:
1. Calling a function (fast — microseconds for most components)
2. Creating some objects (fast)
3. Diffing them (fast)
4. **Usually committing nothing**, because nothing changed

A re-render only *costs* when:
- The component does expensive work during render (a heavy computation, a big `.filter().sort()` over 10k items)
- It's rendering a very large subtree (thousands of nodes)
- It's rendering extremely frequently (every keystroke, every scroll event, every animation frame)
- The commit produces large DOM changes

**The correct order of operations:**
```
1. Measure (Profiler)          ← Lesson 20
2. Is it actually slow?        If not, STOP. You're done.
3. Fix the cause structurally  move state down, split components, compose
4. Only then reach for memo    and verify it helped
```

> **The sentence to say in an interview:** *"I don't optimise re-renders by default — I measure first, and then usually the fix is structural rather than `memo`. Moving state closer to where it's used eliminates a whole cascade; `memo` just puts a comparison in front of it. And `memo` on a component receiving a fresh object prop makes things slightly worse while looking like an optimisation."*

### The structural fixes, which beat `memo` almost every time

```tsx
// ❌ State at the top re-renders everything below it
function Dashboard() {
  const [search, setSearch] = useState("");
  return (
    <>
      <input value={search} onChange={e => setSearch(e.target.value)} />
      <ExpensiveChart />          {/* re-renders on every keystroke */}
      <PaymentsTable />           {/* so does this */}
    </>
  );
}

// ✅ Fix 1 — move state DOWN into the component that uses it
function Dashboard() {
  return (<><SearchBox /><ExpensiveChart /><PaymentsTable /></>);
}
function SearchBox() {
  const [search, setSearch] = useState("");     // now only SearchBox re-renders
  return <input value={search} onChange={e => setSearch(e.target.value)} />;
}

// ✅ Fix 2 — pass children UP, so the expensive subtree isn't re-created
function Shell({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return <div className={open ? "open" : ""}>{children}</div>;
}
<Shell><ExpensiveChart /></Shell>
// `children` is created by Shell's PARENT. Shell re-rendering doesn't recreate it,
// so the element is referentially identical → React bails out. No memo needed.
```

**Fix 2 is the underrated one.** Passing an expensive subtree as `children` means it's created once by a component that isn't re-rendering, so the same element object is reused and React skips it. It's `memo` for free, achieved by composition. Being able to explain *why* it works — the element object is identical, so the diff bails — is a strong signal.

---

## 6. Reading the Profiler

```
React DevTools → Profiler → record → interact → stop
```

What to look for, in order:
| Signal | Means |
|---|---|
| **Wide bars** | That component took a long time to render |
| **Many bars in one commit** | A big cascade — probably state too high in the tree |
| **Many commits per interaction** | Multiple renders where one would do; often an effect chain |
| **"Why did this render?"** (enable in settings) | Names the exact trigger — props, state, hooks, or parent |
| **Grey components** | Didn't re-render (bailed out) — this is what success looks like |

And the free one: **"Highlight updates when components render"**. Turn it on and type into an input. If the whole page flashes, you've found your problem before opening the Profiler.

**The diagnostic sequence for "the app feels slow":**
```
1. Is it a render problem or a load problem?  (Profiler vs Network/Lighthouse)
2. Which interaction?                          Record it
3. How many commits for one interaction?       >1 usually means an effect chain
4. Which component is the widest bar?          That's your target
5. Why did it render?                          Props / state / parent / context
6. Fix the CAUSE (move state, compose), then re-measure
```

---

## 7. Ledger Console: where this bites

```tsx
// The dashboard has a search input, a 4M-row table, and a chart.
// Typing must not re-render the table or the chart.

function PaymentsPage() {
  return (
    <Layout>
      <FilterBar />                {/* owns its own filter state */}
      <RevenueChart />             {/* independent */}
      <PaymentsTable />            {/* reads filters from the URL, not from a parent */}
    </Layout>
  );
}
```

The architectural decision that makes this work: **filters live in the URL, not in a shared parent's state.**

```tsx
function FilterBar() {
  const [params, setParams] = useSearchParams();
  const [draft, setDraft] = useState(params.get("q") ?? "");   // local, per-keystroke

  // Only commit to the URL after the user pauses — so the table re-renders once, not per keystroke
  useDebouncedEffect(() => {
    setParams(p => { draft ? p.set("q", draft) : p.delete("q"); return p; }, { replace: true });
  }, [draft], 300);

  return <input value={draft} onChange={e => setDraft(e.target.value)} />;
}
```

Three things that buys you, all of which are architecture rather than optimisation:
- Typing re-renders `FilterBar` only — the table and chart don't share a parent's state
- The filter state is **shareable and bookmarkable** (a real product feature)
- Back/forward navigation works for free

> **This is the pattern to internalise: the best re-render fix is usually a state-location decision, made before any measurement is necessary.** `memo` is what you reach for when you can't restructure.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Know the four triggers; "props changed" isn't one of them** | Parent re-rendering is the cause; changed props are the symptom |
| **Don't optimise re-renders before measuring** | Most are free; the fix is usually structural |
| **Move state DOWN to the component that uses it** | Eliminates a whole cascade with no memoization |
| **Pass expensive subtrees as `children`** | The element is created by a non-re-rendering parent → free bailout |
| **`memo` only after measuring, and verify it helped** | `memo` + a fresh object prop is strictly worse than no memo |
| **Prefer the updater form `setX(x => …)`** | Correct under batching and stale closures |
| **`flushSync` is an escape hatch, not a tool** | It disables batching and concurrent features for that update |
| **Rendering ≠ committing. Rendering is cheap; DOM mutation isn't** | Decides where to look when something is slow |
| **Keep "highlight updates" on while developing** | You'll see cascades immediately |
| **Never rely on the same-value bailout for correctness** | React may render once before bailing |

---

## 9. Interview traps

**Q1. "When does React re-render a component?"**
The four triggers: own state changed, parent re-rendered, consumed context changed, or a subscribed external store emitted. **Then correct the common wrong answer explicitly:** *"note that 'props changed' isn't a trigger — a component re-renders because its parent did, and the new props are a consequence of that. A component with no props at all still re-renders when its parent does."*

**Q2. "Is re-rendering bad?"**
No — it's the mechanism. A render is a function call, some object allocation, and a diff that usually commits nothing. It costs when the component does expensive work, renders a huge subtree, or renders very frequently. **Measure before optimising**, and prefer structural fixes.

**Q3. "`memo` didn't help. Why?"**
Almost always a new object, array or function prop each render, so the shallow comparison always fails — and now you pay for the comparison too. Other causes: the component consumes a changing context (memo can't stop that), or `children` is a fresh element each render.

**Q4. "How do you stop a keystroke re-rendering the whole page?"**
Structurally, in order of preference: move the input's state into its own component; pass the expensive subtree as `children`; put the value in the URL or a store with selectors; debounce the commit. Then, only if needed, `memo` + `useCallback`. **Leading with structure rather than `memo` is the differentiator.**

**Q5. "Render phase vs commit phase?"**
Render: calls components, diffs, marks effects — **interruptible and must be pure**. Commit: applies DOM mutations, attaches refs, runs layout effects — **synchronous**. Then paint, then passive effects. Purity is required *because* render can be interrupted, restarted or double-invoked.

**Q6. "What is batching, and what changed in React 18?"**
Multiple `setState` calls in the same tick produce one re-render. Before 18 this only applied inside React event handlers; 18 made it automatic everywhere, including promises and timers. Opt out with `flushSync` — rarely, and knowingly.

**Q7. "Three `setCount(count + 1)` calls — what's the result?"**
`1`. All three read `count` from the same render's closure. That's a **closure** issue, not a batching issue — batching is why there's one render, closures are why the value is 1. The updater form fixes it.

**Q8. "How do you diagnose 'the app feels slow'?"**
The §6 sequence: is it load or render? Which interaction? How many commits per interaction (more than one usually means an effect chain)? Which bar is widest? Why did it render? Then fix the cause and re-measure. **Naming the method rather than a fix is what's being assessed.**

**Q9. "Why does passing `children` avoid a re-render?"**
The `children` element object is created by the *parent's parent*. When the middle component re-renders, `props.children` is referentially identical to last time, so React's diff sees the same element and bails out of that subtree. It's memoization achieved by composition, with no `memo` and no dependency array.

**Q10. "When is `flushSync` justified?"**
Focusing an element you just conditionally rendered, measuring the DOM immediately after a state change, or integrating with a library that requires synchronous DOM updates. It disables batching and concurrent scheduling for that update, so it's a deliberate trade — not a way to make async feel synchronous.

---

## 10. Build & break

### Build — the cascade lab
```tsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child label="A" />
      <Child label="B" />
    </>
  );
}
function Child({ label }: { label: string }) {
  console.log("render", label);
  return <div>{label}</div>;
}
```
1. Click. Both children log, despite unchanged props. **Rule 2, demonstrated.**
2. Wrap `Child` in `memo`. They stop logging.
3. Add `onClick={() => {}}` as a prop. `memo` stops working — a new function every render.
4. Wrap the handler in `useCallback`. It works again.
5. Remove `memo` entirely and instead pass the children from above:
   ```tsx
   function App() { const [c, setC] = useState(0);
     return <Shell counter={<button onClick={() => setC(x=>x+1)}>{c}</button>}>
       <Child label="A" /><Child label="B" />
     </Shell>; }
   ```
   Children stop re-rendering with **no `memo` and no `useCallback`.** That's the composition fix.

### Build — the state-location experiment
Build a page with an input, a chart that logs its render, and a 1,000-row list. Then measure three versions with the Profiler:
1. Search state at the top
2. Search state in a `SearchBox` child
3. Search state in the URL with a debounced commit

Record renders-per-keystroke for each. **Those three numbers are the lesson**, and they're a great thing to be able to quote.

### Break — five experiments
1. **Highlight updates.** Turn it on in a real app and type in a form. Watch what flashes.
2. **`memo` doing nothing.** `memo` a component and pass `style={{ color: "red" }}`. It re-renders every time. Hoist the object out and it stops.
3. **Commit vs render.** Add `console.log` to a component and a `useLayoutEffect` that logs too. Re-render with identical output and see the render log fire while the DOM stays untouched.
4. **Batching.** In React 18, `setTimeout(() => { setA(1); setB(2); }, 0)` → count the renders (one). Then wrap each in `flushSync` → two.
5. **The same-value bailout.** `setStatus("idle")` when it's already `"idle"`, with a `console.log` in the body. Note that it may still render once.

### Explain out loud (90 seconds)
1. The four triggers, and why "props changed" isn't one.
2. Why `memo` usually doesn't help.
3. Render vs commit, and why render must be pure.
4. Two structural fixes that beat `memo`.
5. The diagnostic sequence for a slow interaction.

---

## What's next

You know *when* React re-renders. Next: how it decides whether a component is **the same component** across renders — which is the mechanism behind the most confusing bug in React, and the reason `key={index}` is wrong.

Next → **[Lesson 04: Keys, identity & lists](04-keys-and-identity.md)**
