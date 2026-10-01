# Lesson 07 — Refs & escape hatches

> **Why this lesson exists:** refs are the one place React hands you a mutable box and says "you're on your own." Used correctly they solve problems state can't — DOM access, latest-value reads in stale closures, values that shouldn't cause renders. Used as a general-purpose variable they silently break rendering, because **React doesn't know when a ref changes.** This lesson draws the line precisely.

**Time:** ~55 minutes · **Prereq:** Lesson 05

---

## 1. The idea in one sentence

> **State is for values the UI renders; refs are for values the UI doesn't — and the moment you find yourself wanting the screen to update when a ref changes, you wanted state.**

---

## 2. What a ref actually is

```tsx
const ref = useRef(0);
// ref === { current: 0 }   ← a plain mutable object, stored on the fiber
```

That's the whole implementation. `useRef(x)` creates `{ current: x }` once, stores it on the fiber, and returns **the same object** on every render.

| | `useState` | `useRef` |
|---|---|---|
| Survives re-renders | ✅ | ✅ |
| Changing it re-renders | **✅** | **❌** |
| Value is a snapshot per render | ✅ | ❌ — always the latest |
| Safe to read/write during render | read yes, write only in the restart pattern | **❌ neither** |
| Use for | anything the UI displays | everything else |

**The two properties that define its use:** mutating `.current` doesn't re-render, and `.current` is always the *current* value — not a snapshot. Those are exactly the two things state can't do.

---

## 3. The three legitimate uses

### Use 1 — DOM access
```tsx
function SearchBox() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => { inputRef.current?.focus(); }, []);

  return <input ref={inputRef} />;
}
```

What DOM refs are for: focus, text selection, scroll position, measuring (`getBoundingClientRect`), media playback (`video.play()`), canvas contexts, and integrating non-React libraries.

**What they're not for:** reading or writing values that React manages.
```tsx
// ❌ Reading the DOM instead of state
const value = inputRef.current?.value;
// ❌ Writing the DOM behind React's back — the next render overwrites it
inputRef.current!.textContent = "Paid";
inputRef.current!.style.display = "none";
```
React owns anything it rendered. Mutating it means the DOM and the element tree disagree until the next render silently reverts you.

**Timing:** `ref.current` is `null` during render and populated in the **layout phase** of the commit ([Lesson 03](../01-mental-model/03-rendering-and-commit.md)). So refs are readable in `useLayoutEffect` and `useEffect`, never in the render body.

```tsx
// ref-as-prop (React 19) — no forwardRef needed
function Input({ ref, ...rest }: React.ComponentPropsWithRef<"input">) {
  return <input ref={ref} {...rest} />;
}
// React 18 and earlier:
const Input = forwardRef<HTMLInputElement, Props>((props, ref) => <input ref={ref} {...props} />);
```

```tsx
// Callback refs — for measuring, or when the node changes identity
const [height, setHeight] = useState(0);
const measureRef = useCallback((node: HTMLDivElement | null) => {
  if (node) setHeight(node.getBoundingClientRect().height);
}, []);
return <div ref={measureRef} />;
```
**Callback refs fire on mount and unmount with the node**, which makes them the right tool when you need to react to the element appearing — a `useEffect` + `useRef` can't tell you *when* the node arrived. In React 19 a callback ref may also return a cleanup function.

### Use 2 — Values that shouldn't trigger renders
```tsx
function Stopwatch() {
  const [elapsed, setElapsed] = useState(0);
  const intervalRef = useRef<number | null>(null);        // ← the UI doesn't render this
  const startedAtRef = useRef<number>(0);

  function start() {
    startedAtRef.current = Date.now();
    intervalRef.current = window.setInterval(() => {
      setElapsed(Date.now() - startedAtRef.current);       // state → renders
    }, 100);
  }
  function stop() {
    if (intervalRef.current) clearInterval(intervalRef.current);
    intervalRef.current = null;
  }
  useEffect(() => stop, []);                                // cleanup on unmount
  return <><span>{elapsed}ms</span><button onClick={start}>Start</button></>;
}
```
Timer IDs, WebSocket instances, observer instances, animation frame handles, scroll positions you only read on demand, and "has this already run?" flags. **None of them appear on screen, so none of them should cause a render.**

### Use 3 — The latest-value ref (the stale-closure escape)
```tsx
// The problem: a callback that outlives its render (Lesson 05)
function usePaymentStream(onPayment: (p: Payment) => void) {
  // ❌ Adding onPayment to deps reconnects the socket every render,
  //    because the caller passes a new inline function each time.
  useEffect(() => {
    const es = new EventSource("/v1/events/stream");
    es.onmessage = e => onPayment(JSON.parse(e.data));
    return () => es.close();
  }, [onPayment]);
}

// ✅ The latest-ref pattern: subscribe once, always call the newest callback
function usePaymentStream(onPayment: (p: Payment) => void) {
  const onPaymentRef = useRef(onPayment);
  useLayoutEffect(() => { onPaymentRef.current = onPayment; });   // no deps: update every render

  useEffect(() => {
    const es = new EventSource("/v1/events/stream");
    es.onmessage = e => onPaymentRef.current(JSON.parse(e.data));  // reads the LATEST
    return () => es.close();
  }, []);                                                          // connects once ✅
}
```

**This pattern is the answer to "my effect reconnects on every render".** It separates *what the effect depends on* (nothing — the URL) from *what it calls* (the latest handler). Points worth knowing:
- Use `useLayoutEffect` (or assign during render — see below) so the ref is updated before any child effect could fire the callback.
- This is what React's proposed `useEffectEvent` formalises. Until it ships, the ref pattern is the idiomatic version, and you'll see it in every serious library.

```tsx
// The reusable hook — put this in your utils
export function useEventCallback<A extends unknown[], R>(fn: (...args: A) => R) {
  const ref = useRef(fn);
  useLayoutEffect(() => { ref.current = fn; });
  return useCallback((...args: A) => ref.current(...args), []);   // stable identity, fresh behaviour
}
```
A stable function reference whose *behaviour* is always current. Genuinely useful for event handlers passed to `memo`'d children, subscriptions, and anything that shouldn't change identity.

> **The caveat to state honestly:** `useEventCallback` is unsafe to call *during render* (it may read a not-yet-updated value) and it hides a real dependency. Use it for event handlers and subscription callbacks, not as a blanket way to empty dependency arrays.

---

## 4. Ref anti-patterns

### Anti-pattern 1 — Using a ref where you need a render
```tsx
// ❌ The UI never updates
const countRef = useRef(0);
function increment() { countRef.current++; }
return <span>{countRef.current}</span>;      // renders 0 forever
```
**The test:** does the screen need to change when this value changes? Then it's state. This sounds obvious and people do it anyway, usually while trying to "optimise away re-renders."

### Anti-pattern 2 — Reading or writing a ref during render
```tsx
function Bad() {
  const ref = useRef(0);
  ref.current++;                     // ❌ impure render (Lesson 01)
  return <div>{ref.current}</div>;   // ❌ and wrong under StrictMode / concurrent
}
```
React may call your function twice or abandon a render. A ref mutated during render gets an unpredictable value. **Read and write refs in event handlers and effects only.**

The one documented exception is lazy initialisation:
```tsx
function Video() {
  const playerRef = useRef<Player | null>(null);
  if (playerRef.current === null) playerRef.current = new Player();   // ✅ runs once, idempotent
}
```

### Anti-pattern 3 — Refs as a dependency-array escape
```tsx
// ❌ "The linter complains, so I'll ref everything"
const dataRef = useRef(data);
dataRef.current = data;
useEffect(() => { doSomething(dataRef.current); }, []);   // never re-runs when data changes
```
You've silenced the linter and broken the effect. **If the effect should re-run when the value changes, it's a dependency.** Refs are for values the effect *reads* but shouldn't *react to* — a genuinely different thing, and the distinction is the whole skill here.

### Anti-pattern 4 — Manipulating React-owned DOM
```tsx
// ❌ React will overwrite this on the next render
ref.current!.classList.add("highlight");
ref.current!.innerHTML = "<b>Paid</b>";

// ✅ Let React render it
<div className={cx({ highlight })}>{label}</div>
```

### Anti-pattern 5 — Exposing DOM nodes through `useImperativeHandle` unnecessarily
```tsx
// ❌ A whole imperative API for something props could do
useImperativeHandle(ref, () => ({ setValue, getValue, clear, validate, focus, blur }));

// ✅ Expose only what genuinely can't be expressed declaratively
useImperativeHandle(ref, () => ({ focus: () => inputRef.current?.focus() }), []);
```
`useImperativeHandle` is legitimate for focus, scroll-into-view, play/pause, and opening a native `<dialog>` — imperative *actions* with no declarative equivalent. **Not** for reading or setting values; those are props and callbacks.

---

## 5. Integrating non-React libraries

The most common real use of refs, and the pattern is always the same.

```tsx
function RevenueChart({ data }: { data: Series }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<Chart | null>(null);

  // 1. Create once, destroy on unmount
  useEffect(() => {
    const el = containerRef.current;
    if (!el) return;
    chartRef.current = new Chart(el, { data: [] });
    return () => { chartRef.current?.destroy(); chartRef.current = null; };   // ← cleanup is mandatory
  }, []);

  // 2. Sync data imperatively when it changes
  useEffect(() => { chartRef.current?.setData(data); }, [data]);

  return <div ref={containerRef} />;         // React owns this div; the library owns its children
}
```

Four rules for wrapping any imperative library:
1. **Create in an effect, destroy in its cleanup.** Never in render — StrictMode will create two.
2. **Give the library its own container** that React renders but never populates. React must not diff the library's DOM.
3. **Sync updates in separate effects** keyed to what changed — don't recreate on every data change.
4. **The cleanup must genuinely tear down**: remove listeners, cancel animation frames, disconnect observers, close sockets. StrictMode's double-mount in dev exists precisely to catch a missing one.

```tsx
// Observers follow the same shape
useEffect(() => {
  const el = ref.current;
  if (!el) return;
  const ro = new ResizeObserver(([entry]) => setWidth(entry!.contentRect.width));
  ro.observe(el);
  return () => ro.disconnect();
}, []);
```

---

## 6. `flushSync` — the other escape hatch

```tsx
import { flushSync } from "react-dom";

function addRow() {
  flushSync(() => setRows(r => [...r, newRow]));   // render + commit synchronously
  listRef.current?.lastElementChild?.scrollIntoView();   // the row definitely exists now
}
```
Without `flushSync`, the state update is batched and the DOM isn't updated yet when the next line runs.

**Legitimate uses:** scrolling to or focusing an element you just rendered, measuring immediately after a state change, and integrating with libraries that demand synchronous DOM. **The cost:** it disables batching and concurrent scheduling for that update, so overuse makes the app janky. It's a scalpel, and using it casually is a review finding.

> Often there's a better answer: a **callback ref** fires when the node appears, so `<div ref={node => node?.scrollIntoView()} />` achieves the same thing with no `flushSync`. Reach for that first.

---

## 7. Ledger Console's ref usage

| Need | Tool |
|---|---|
| Focus the search input on `/` keypress | `useRef<HTMLInputElement>` + a key handler |
| Virtualized table scroll container | `useRef<HTMLDivElement>` passed to the virtualizer |
| Measure the table width for column sizing | callback ref + `ResizeObserver` |
| SSE connection instance | `useRef<EventSource>` — never rendered |
| "Has the payment list already scrolled to top?" flag | `useRef<boolean>` |
| Chart library instance | `useRef<Chart>` + create/destroy effect |
| Latest `onPaymentReceived` handler for the stream | the latest-ref pattern (§3) |
| Idempotency key for an in-flight refund | `useRef<IdempotencyKey>` — stable across re-renders, not rendered |

That last one is worth pausing on: the idempotency key from [API Lesson 18](../../API/04-production/18-reliability-and-idempotency.md) must be **the same value across retries** but must **not** cause a re-render. A ref is exactly right — generate it once when the user opens the refund modal, reuse it on every retry, and reset it when they start a new refund.

```tsx
function useIdempotencyKey(resetOn: unknown) {
  const keyRef = useRef<IdempotencyKey | null>(null);
  const prevRef = useRef(resetOn);
  if (keyRef.current === null || prevRef.current !== resetOn) {   // lazy-init pattern
    keyRef.current = crypto.randomUUID() as IdempotencyKey;
    prevRef.current = resetOn;
  }
  return keyRef.current;
}
```

---

## 8. Production rules

| Rule | Why |
|---|---|
| **If the UI must change when it changes, it's state — not a ref** | React doesn't know when a ref changes |
| **Never read or write `.current` during render** | Render may be doubled or abandoned; the exception is lazy init |
| **`ref.current` is `null` during render, populated in the layout phase** | Read it in effects, not the body |
| **Never mutate DOM React owns** | The next render silently reverts you |
| **Use callback refs when you need to know *when* a node appears** | `useRef` + `useEffect` can't tell you |
| **The latest-ref pattern for callbacks in long-lived subscriptions** | Subscribe once, always call the newest handler |
| **Never use a ref to silence the exhaustive-deps lint** | If it should re-run, it's a dependency |
| **Wrap imperative libraries: create in an effect, destroy in cleanup, own container** | StrictMode will catch a missing teardown |
| **`useImperativeHandle` for actions, never for values** | Values are props and callbacks |
| **Prefer a callback ref over `flushSync` for "act on a just-rendered node"** | No batching penalty |

---

## 9. Interview traps

**Q1. "`useRef` vs `useState`?"**
Both survive re-renders; only state triggers one. State is a per-render snapshot; a ref is always current. **The decision test:** does the screen need to change when this value changes? Then state. Otherwise a ref.

**Q2. "When would you use a ref?"**
Three cases: DOM access (focus, measure, scroll, media, canvas, third-party libraries), values that shouldn't render (timer IDs, socket instances, flags), and the latest-value pattern for callbacks that outlive their render.

**Q3. "Why can't you read a ref during render?"**
Render must be pure and may be called twice (StrictMode) or abandoned (concurrent). A ref read or mutated during render gets an unpredictable value and makes the render impure. The one documented exception is lazy initialisation, because it's idempotent.

**Q4. "My effect reconnects a WebSocket on every render. Fix it."**
The callback prop is a new function each render and it's in the deps. Use the **latest-ref pattern**: store the handler in a ref updated in a `useLayoutEffect`, subscribe once with `[]`, and call `ref.current(...)` inside. Mention that this is what `useEffectEvent` is designed to formalise.

**Q5. "What's a callback ref and when is it better?"**
A function passed as `ref` that React calls with the node on mount and `null` on unmount. Better when you need to *react* to the node appearing — measuring, scrolling into view, initialising a library — because a `useRef` gives you no notification of when it was set.

**Q6. "How do you integrate a charting library?"**
A container div React owns but never fills; create the instance in an effect with a destroy cleanup; sync data in separate effects keyed on what changed. Then the detail that shows experience: **StrictMode's double mount in dev exists to catch a missing cleanup** — if the chart appears twice, your teardown is wrong.

**Q7. "When is `flushSync` justified?"**
Scroll/focus an element you just rendered, measure immediately after a state change, or integrate with a library needing synchronous DOM. It disables batching and concurrent scheduling for that update. **And usually a callback ref does the same job for free** — preferring that shows judgement.

**Q8. "Can you use a ref to avoid putting something in a dependency array?"**
Only if the effect should genuinely *not* re-run when it changes. If it should, you've silenced the linter and broken the effect. The distinction is "values the effect **reads**" vs "values the effect **reacts to**" — and it's the whole skill.

**Q9. "How do you keep an idempotency key stable across retries?"**
A ref, generated lazily and reset when the user starts a new operation. It must be identical across retries (or the server treats them as separate operations) and must not cause a re-render. **Connecting this to the API-side idempotency design is a strong cross-domain signal.**

---

## 10. Build & break

### Build — the ref lab
```tsx
function Lab() {
  const [stateCount, setStateCount] = useState(0);
  const refCount = useRef(0);
  console.log("render", { stateCount, ref: refCount.current });
  return (
    <>
      <button onClick={() => setStateCount(c => c + 1)}>state {stateCount}</button>
      <button onClick={() => { refCount.current++; console.log("ref now", refCount.current); }}>
        ref {refCount.current}
      </button>
    </>
  );
}
```
Click the ref button five times: the console shows it incrementing, the screen shows `0`. Then click the state button once and watch the ref's label jump to `5`. **That's "React doesn't know it changed", demonstrated.**

### Build — the latest-ref hook
Implement `useEventCallback` from §3 and use it to wire an SSE stream. Verify with a log that the connection opens **once** while the handler prop changes on every render.

### Build — wrap an imperative library
Integrate any chart library (Chart.js, uPlot) with the four rules. Then **deliberately omit the destroy cleanup** and run under StrictMode — you'll get two charts stacked. That's the double-mount doing its job, and it's worth seeing once.

### Break — five experiments
1. **Ref where state belongs.** Render `{refCount.current}` and increment it. Nothing updates.
2. **Ref during render.** `ref.current++` in the body under StrictMode. Log it. The value is double what you expect.
3. **DOM behind React's back.** `ref.current.textContent = "Paid"`, then trigger any re-render. It reverts.
4. **Ref as a deps escape.** Ref a value the effect should react to, use `[]`, then change the value. The effect never re-runs.
5. **Missing cleanup.** Create a `setInterval` in an effect with no cleanup. Navigate away and back a few times. Watch the console accelerate.

### Explain out loud (60 seconds)
1. `useRef` vs `useState`, with the decision test.
2. Three legitimate uses of refs.
3. Why you can't read a ref during render.
4. The latest-ref pattern and what it solves.
5. The four rules for wrapping an imperative library.

---

## What's next

The most important lesson in the React track. Effects are the most-used and most-misused hook in React — and the fix isn't better `useEffect` code, it's **deleting most of them**. Next: what effects are actually for, the four cases where you don't need one, and why StrictMode runs yours twice.

Next → **[Lesson 08: Effects — synchronization, not lifecycle](08-effects.md)**
