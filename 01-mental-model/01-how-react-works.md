# Lesson 01 — How React actually works

> **Why this lesson exists:** most people learn React as "components and hooks" and build their mental model from what happens to work. That model breaks the first time something surprising happens, and there's nothing underneath to reason from. This lesson gives you the machine: what React *is*, what a component actually returns, and what reconciliation does — so that every later behaviour is derivable rather than memorised.

**Time:** ~55 minutes · **Prereq:** JavaScript, and ideally the [TypeScript track](../../TypeScript/README.md)

---

## 1. The idea in one sentence

> **React is a library for keeping a tree of DOM nodes in sync with a tree of JavaScript values — you describe what the UI *should be* for the current state, and React figures out the minimal set of DOM operations to make reality match.**

Everything else — hooks, effects, Suspense, Server Components — is machinery around that one job.

---

## 2. The problem React solves

Before React, UI code was **imperative**: you told the DOM how to change, step by step.

```js
// The world before. A payment status changes from "pending" to "succeeded".
const row = document.getElementById(`payment-${id}`);
row.querySelector(".status").textContent = "Paid";
row.querySelector(".status").className = "status status--success";
row.querySelector(".capture-btn")?.remove();
if (!row.querySelector(".refund-btn")) {
  const btn = document.createElement("button");
  btn.className = "refund-btn";
  btn.textContent = "Refund";
  btn.addEventListener("click", () => refund(id));       // ← and remember to remove this later
  row.querySelector(".actions").appendChild(btn);
}
updateSummaryTotals();
maybeShowToast();
```

Everything wrong with this, and these are the actual reasons React exists:

1. **You must know the previous state to write the transition.** The code above only works if the row was *pending*. From "failed" it produces garbage. With N states you need N² transitions.
2. **Every branch is a chance to forget something.** Forget to remove the capture button and it lingers. Forget `removeEventListener` and you leak.
3. **The UI and the data drift.** Two sources of truth — your variables and the DOM — and they diverge silently.
4. **It doesn't compose.** You can't take that block and reuse it inside another widget.

React's answer: **describe the UI for the current state; never describe the transition.**

```tsx
function PaymentRow({ payment }: { payment: Payment }) {
  return (
    <tr>
      <td className={`status status--${payment.status}`}>{STATUS_LABELS[payment.status]}</td>
      <td className="actions">
        {payment.status === "requires_capture" && <button onClick={capture}>Capture</button>}
        {payment.status === "succeeded" && <button onClick={refund}>Refund</button>}
      </td>
    </tr>
  );
}
```

There is **no transition code**. You state what the row looks like for any status; React computes the difference. That's the whole value proposition, and it's worth being able to articulate — *"React converts an O(n²) problem of transitions into an O(n) problem of descriptions."*

> **The trade you're making:** you give up direct control of the DOM and accept some runtime overhead, in exchange for never writing a transition again. For a static page that's a bad trade (hence Astro, plain HTML). For an app with lots of interactive state, it's an excellent one.

---

## 3. What a component actually returns

**A component does not return DOM. It does not "render to the screen."** It returns a plain JavaScript object.

```tsx
function Badge({ status }: { status: PaymentStatus }) {
  return <span className="badge">{status}</span>;
}
```

JSX is syntax sugar. That compiles (with the modern JSX transform) to:

```js
import { jsx as _jsx } from "react/jsx-runtime";

function Badge({ status }) {
  return _jsx("span", { className: "badge", children: status });
}
```

And calling it produces:

```js
{
  $$typeof: Symbol(react.element),   // a security marker — see below
  type: "span",                       // a string for a host element, a FUNCTION for a component
  key: null,
  ref: null,
  props: { className: "badge", children: "succeeded" },
}
```

**That object is a React element.** It is:
- **Immutable** — you never mutate it; you create a new one on the next render
- **Cheap** — creating it allocates one object, nothing touches the DOM
- **A description, not a thing** — like a blueprint, not a building

```tsx
// Nested JSX produces a nested tree of these objects:
<div className="row">
  <Badge status="succeeded" />
  <span>4999</span>
</div>

// →
{ type: "div", props: { className: "row", children: [
    { type: Badge, props: { status: "succeeded" } },      // ← type is the FUNCTION
    { type: "span", props: { children: "4999" } },
]}}
```

Two consequences that matter enormously:

**1. `<Badge />` does not call `Badge`.** It creates an object whose `type` *is* the `Badge` function. **React** decides when to call it — which is why React can skip rendering (memoization), delay it (transitions), or run it twice (StrictMode). If JSX called your function directly, none of that would be possible.

**2. `<Badge />` and `Badge()` are different things.**
```tsx
{<Badge status="x" />}      // an element — React manages it, it can have its own state
{Badge({ status: "x" })}    // a direct call — the returned elements are INLINED into the parent
```
The second works visually and is almost always wrong: `Badge` isn't a component in the tree, so it has no identity, no state, no hooks of its own, and no place in the DevTools. This is exactly why *"render props are called, components are rendered"* matters ([Lesson 13](../04-composition/13-composition-patterns.md)).

> **The `$$typeof` symbol** exists to prevent XSS: if a server returns JSON that looks like a React element, it can't be rendered, because JSON can't contain a `Symbol`. A small, elegant detail worth knowing.

---

## 4. The three trees

React maintains three parallel structures. Being able to name them is a real differentiator.

```
   YOUR CODE                REACT'S MEMORY                 THE BROWSER
   ─────────                ──────────────                 ───────────
   Element tree      →      Fiber tree              →      DOM tree
   (what you return)        (what React remembers)         (what the user sees)

   • immutable              • mutable, persistent          • the real thing
   • recreated each render  • survives across renders      • mutated minimally
   • cheap objects          • holds STATE, effects, refs    • expensive to touch
```

**The Fiber tree is the important one.** A fiber is React's internal record for one component instance, and it holds:
- the component type and current props
- **its state** (the hooks list, in order — this is why hook order matters)
- its effects, refs, and context subscriptions
- pointers to parent/child/sibling
- work-in-progress bookkeeping (priority, flags)

**State lives on the fiber, not in your function.** Your function runs and returns; its local variables die. The fiber persists between renders and remembers. That single fact explains where `useState` values are actually stored, why a component "keeps" its state, and — crucially — why **destroying the fiber destroys the state** ([Lesson 04](04-keys-and-identity.md)).

### Why Fiber exists (the 2017 rewrite)
The original React reconciler walked the tree recursively and couldn't stop. A big update blocked the main thread and the page froze. Fiber restructured the tree as a **linked list that can be traversed iteratively**, so React can:
- **Pause** work after any fiber and yield to the browser
- **Prioritise** — a keystroke jumps ahead of a background list re-render
- **Abandon** work that's no longer needed
- **Reuse** completed work

That capability is what `useTransition`, Suspense and concurrent rendering are all built on ([Lesson 11](../03-hooks/11-concurrent-hooks.md)).

---

## 5. Reconciliation: the diffing algorithm

When state changes, React re-renders and compares the new element tree with the previous one. A general tree-diff is O(n³); React gets to O(n) with **two heuristics**, and these two rules generate a surprising amount of real-world behaviour:

### Rule 1 — Different `type` ⇒ destroy and rebuild
```tsx
// render 1
<div><Counter /></div>
// render 2
<span><Counter /></span>
```
`div` ≠ `span`, so React **unmounts the entire subtree** and mounts a fresh one. `Counter`'s state is gone. It does not try to be clever about it.

```tsx
// The real-world version of this bug:
{isEditing ? <EditForm /> : <ViewForm />}
// Switching modes destroys everything below. Usually fine — sometimes a surprise.
```

### Rule 2 — Same `type` ⇒ keep the fiber, update the props
```tsx
<div className="a" />   →   <div className="b" />
// Same type: React keeps the DOM node and sets one attribute.
```
The component's state, refs and effects survive. This is the common case, and it's why React is fast.

### Rule 3 — Children are matched by position, unless you give a `key`
```tsx
// No keys: React matches by index
[<Row a />, <Row b />]  →  [<Row x />, <Row a />, <Row b />]
// React sees: position 0 changed a→x, position 1 changed b→a, position 2 is new.
// It updates two rows and creates one. All three rows' state shifts.

// With keys: React matches by key
[<Row key="a"/>, <Row key="b"/>]  →  [<Row key="x"/>, <Row key="a"/>, <Row key="b"/>]
// React sees: "x" is new, "a" and "b" moved. It creates ONE row and moves two.
```
[Lesson 04](04-keys-and-identity.md) is entirely this, because it causes the most confusing bug in React.

> **The mental model to carry:** *"a component's identity is its **type** plus its **position** among siblings (or its `key`). Change any of those and it's a different component — new fiber, fresh state."*

---

## 6. The render → commit pipeline

Two distinct phases. Confusing them is the source of most "why did this happen?" questions.

```
  1. TRIGGER        setState() / initial mount / parent re-render
        │
  2. RENDER PHASE   ← can be paused, restarted, or abandoned. Runs your component functions.
        │            Must be PURE: no DOM reads/writes, no mutations, no side effects
        │            Produces an element tree, diffs it, marks fibers with effect flags
        │
  3. COMMIT PHASE   ← synchronous, cannot be interrupted
        ├─ mutation:  applies DOM changes
        ├─ layout:    useLayoutEffect runs, refs are attached  (blocks paint)
        │
  4. BROWSER PAINT  ← the user finally sees it
        │
  5. PASSIVE        useEffect runs (after paint)
```

**Why the render phase must be pure** — this is the mechanical reason behind a rule people follow without understanding:

React may call your component function **more than once for one visible update**: StrictMode double-invokes it in development, concurrent rendering can start work and throw it away, and a higher-priority update can restart the render. If your function has side effects — mutating a variable, writing to the DOM, sending analytics — those effects happen an unpredictable number of times.

```tsx
// ❌ Impure — the render phase is not a safe place for this
let renderCount = 0;
function Bad() {
  renderCount++;                        // mutation
  document.title = "Payments";          // DOM write
  analytics.track("viewed");            // side effect — may fire twice, or zero times
  return <div />;
}

// ✅ Pure render; effects belong in effects
function Good() {
  useEffect(() => { document.title = "Payments"; }, []);
  return <div />;
}
```

**Why `useLayoutEffect` blocks paint and `useEffect` doesn't:** layout effects run in the commit phase, *before* the browser paints, so you can measure the DOM and adjust without the user seeing a flicker. `useEffect` runs after paint, so it never delays the user seeing something. That's the whole distinction ([Lesson 08](../02-state-effects/08-effects.md)).

---

## 7. React's actual guarantees

What React does and doesn't promise, stated plainly — useful because a lot of confusion comes from expecting guarantees that don't exist.

| React promises | React does **not** promise |
|---|---|
| The DOM will eventually match your render output | That it renders synchronously when you call `setState` |
| Your component function's output is used to compute the diff | That your component runs exactly once per update |
| Effects run after commit, and cleanup runs before the next effect | That effects run at a particular time relative to other components' |
| State updates are batched within an event handler (and, since 18, everywhere) | That state is updated by the next line of code |
| Cleanup runs on unmount | That unmount happens when you expect (StrictMode remounts in dev) |

That right-hand column is where bugs come from. **Every one of them is a place where people assume imperative behaviour from a declarative system.**

---

## 8. Where React fits, and the honest alternatives

| Approach | Mechanism | Trade |
|---|---|---|
| **React** | Re-render components, diff a virtual tree | Simple model; VDOM overhead; needs memoization discipline |
| **Solid / Svelte 5** | Fine-grained reactivity — only the exact DOM node updates | Faster, less to memoize; a more constrained model |
| **Vue** | Reactive proxies + a compiler | Middle ground |
| **Angular** | Signals (now) / zone-based change detection (before) | Batteries included; heavier |
| **htmx / Astro** | Server-rendered HTML, minimal JS | Excellent for content; poor for rich interaction |

**Where React is genuinely not the right answer:** a mostly-static content site (Astro), an app needing hard 60fps on thousands of independently-updating nodes (Solid), or anything where a 40kB baseline matters more than ecosystem.

**Where it is:** rich interactive applications, large teams (the ecosystem and hiring pool are decisive), and anywhere the component model's composability pays off. **Being able to say when React is the wrong tool is a positive signal**, not a disloyalty.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Render functions must be pure** | React may call them multiple times, or throw the work away |
| **Never mutate props or state** | The diff relies on comparing values; mutation makes it lie |
| **Don't touch the DOM during render** | Render can be interrupted; the DOM isn't there yet |
| **Think "what does the UI look like for this state", never "what changed"** | The entire value of React is not writing transitions |
| **`<Component />` not `Component()`** | The second inlines and loses identity, state and hooks |
| **Remember state lives on the fiber, not in your function** | Explains persistence *and* why destroying a fiber destroys state |
| **Keep DevTools "highlight updates" on while learning** | You'll see identity and re-render behaviour instead of guessing |

---

## 10. Interview traps

**Q1. "What is React and what problem does it solve?"**
Not "a UI library." The strong answer:
> *"It keeps the DOM in sync with a tree of JavaScript values. Before React, UI code was imperative — you wrote the transition from state A to state B, which means N² transitions for N states, and every branch is a chance to forget a cleanup. React lets you describe the UI for the current state and computes the difference. You trade direct DOM control and some runtime overhead for never writing a transition again."*

**Q2. "What does JSX compile to?"**
`jsx()` calls (the modern transform) producing plain objects with `type`, `props`, `key`, `ref`. Then the consequence: **`<Badge />` doesn't call `Badge`** — it creates an object whose `type` is the function, and React decides when to call it. That's what makes skipping, delaying and double-invoking possible.

**Q3. "What's the virtual DOM, and is it faster than the real DOM?"**
It's the element tree — a cheap JS description compared against the previous one. **And no, it isn't inherently faster than well-written imperative DOM code** — it's *faster than badly-written* imperative code, and it buys you the declarative model. Saying this honestly is a stronger answer than repeating the marketing. Frameworks like Solid skip the VDOM entirely and are faster still.

**Q4. "Explain reconciliation."**
The diff between the previous and next element trees, made O(n) by two heuristics: different `type` ⇒ destroy and rebuild the subtree; same `type` ⇒ keep the fiber and update props. Children match by position unless keyed. Those two rules generate the lost-state and lost-focus bugs.

**Q5. "What is Fiber?"**
React's internal representation of a component instance, and the tree of them. It holds state, effects, refs and props, and persists across renders. Restructuring the tree as a traversable linked list (2017) is what made rendering interruptible — which is what `useTransition`, Suspense and concurrent rendering are built on.

**Q6. "Render phase vs commit phase?"**
Render: calls your components, produces and diffs elements, **interruptible, must be pure**. Commit: applies DOM changes, runs layout effects and attaches refs, **synchronous and uninterruptible**. Then paint. Then passive effects (`useEffect`). This split is why render purity is a requirement rather than a style preference.

**Q7. "Why must render be pure?"**
Because React may call your function twice (StrictMode), abandon a render (concurrent), or restart it (a higher-priority update). Side effects in render would therefore happen an unpredictable number of times. Purity is what makes those capabilities safe.

**Q8. "Where does component state actually live?"**
On the **fiber**, keyed by hook call order — not in your function, whose locals die every render. Hence: state survives re-renders, hook order must be stable, and destroying the fiber (type change, position change, key change) destroys the state.

**Q9. "When is React the wrong choice?"**
Mostly-static content (Astro/plain HTML), hard-realtime rendering of thousands of independently-updating nodes (Solid's fine-grained reactivity), or a strict bundle budget. Right for rich interactive apps, large teams, and anywhere ecosystem and hiring matter.

---

## 11. Build & break

### Build — see the elements
```bash
npm create vite@latest react-lab -- --template react-ts && cd react-lab && npm i
```
```tsx
// src/inspect.tsx
const el = <div className="row"><span>hello</span></div>;
console.log(el);
console.log(JSON.stringify(el, (k, v) => (typeof v === "symbol" ? String(v) : v), 2));
```
**Look at the object.** Find `type`, `props`, `children`, `$$typeof`. This five-minute exercise removes most future confusion about what components "return".

Then compile a file and read the output:
```bash
npx esbuild src/inspect.tsx --loader=tsx --jsx=automatic
```

### Build — write the same UI imperatively, then declaratively
Implement a payment row with four states (`requires_capture`, `succeeded`, `failed`, `refunded`) and buttons that differ per state — **first in plain DOM**, handling every transition, then in React. Count the lines and the number of transitions you had to write.

**That count is the argument for React**, and having done it once makes you able to explain the value proposition from experience rather than repetition.

### Break — four experiments
1. **Type change destroys state.** Render `{flag ? <div><Counter/></div> : <span><Counter/></span>}`. Increment the counter, toggle `flag`, and watch the count reset. Then make both `<div>` and watch it survive.
2. **Component vs function call.** Render `<Badge status="x" />` and `{Badge({status:"x"})}` side by side. Both look identical. Now open DevTools and find only one of them in the component tree. Then add `useState` to `Badge` and watch the direct call break.
3. **Impure render.** Add `console.log("render")` and a `let count = 0; count++` to a component. Run in StrictMode and count the logs. Then remove StrictMode and count again.
4. **Watch the highlights.** Turn on "Highlight updates when components render" in DevTools and click around any app you have. You'll immediately see components re-rendering that you didn't expect — which is the whole of Module 6, previewed for free.

### Explain out loud (60 seconds)
1. What problem React solves, and the trade it makes.
2. What a component returns, and why `<X/>` ≠ `X()`.
3. The three trees, and where state lives.
4. The two reconciliation heuristics.
5. Render phase vs commit phase, and why render must be pure.

---

## What's next

You know what React *is*. Next: the part you write — JSX in detail, and how to design a component's props as a genuine API rather than an accumulation of options.

Next → **[Lesson 02: JSX, components & props](02-jsx-components-props.md)**
