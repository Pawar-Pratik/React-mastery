# Lesson 21 — Rendering performance

> **Why this lesson exists:** *"how would you render a million rows?"* is one of the most common senior front-end interview questions, and the answer isn't `memo` — it's **render fewer things**. This lesson covers the fixes in order of impact: virtualization, state placement, work reduction, and only then memoization. It's also where the Ledger Console's 4-million-row table becomes possible.

**Time:** ~70 minutes · **Prereq:** Lessons 10, 20

---

## 1. The idea in one sentence

> **Rendering performance is almost never about making renders faster — it's about doing fewer of them, over fewer nodes.**

---

## 2. The fix hierarchy

In descending order of impact. **Work top-down; most people start at the bottom.**

```
1. Render fewer components        virtualization, pagination, lazy sections
2. Render less often              state placement, composition, URL state
3. Do less work per render        move computation out, cheaper data structures
4. Skip renders                   memo / useMemo / useCallback — or the Compiler
5. Make renders non-blocking      useTransition, useDeferredValue
```

**A 10,000-row list virtualized to 20 visible rows is a 500× reduction.** No amount of `memo` competes with that. Yet the reflex is to reach for step 4 first, because it's the most-discussed and requires no structural thinking.

---

## 3. Virtualization — the biggest lever

Render only what's visible.

```tsx
import { useVirtualizer } from "@tanstack/react-virtual";

function PaymentsTable({ rows }: { rows: Payment[] }) {
  const scrollRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: rows.length,                          // 4,000,000
    getScrollElement: () => scrollRef.current,
    estimateSize: () => 48,                      // row height in px
    overscan: 5,                                 // render a few extra above/below
  });

  return (
    <div ref={scrollRef} style={{ height: 600, overflow: "auto" }}>
      {/* A spacer that makes the scrollbar the right size */}
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map(v => {
          const payment = rows[v.index]!;
          return (
            <PaymentRow
              key={payment.id}                   {/* ← DATA id, never v.index (Lesson 04) */}
              payment={payment}
              style={{ position: "absolute", top: 0, left: 0, width: "100%",
                       height: v.size, transform: `translateY(${v.start}px)` }}
            />
          );
        })}
      </div>
    </div>
  );
}
```

**~23 DOM nodes instead of 4,000,000.** Scrolling stays at 60fps because the virtualizer positions rows with `transform` (compositor-only, no layout).

### The details that make it correct
| Detail | Why |
|---|---|
| **Key by data ID, not virtual index** | The window slides; index keys make per-row state follow the *viewport position* ([Lesson 04](../01-mental-model/04-keys-and-identity.md)) |
| **`transform`, not `top`** | `transform` skips layout and paint; `top` triggers both |
| **`overscan: 3–10`** | Too low flickers on fast scroll; too high wastes renders |
| **Dynamic heights → `measureElement`** | For variable-height rows, measure and cache |
| **Accessibility: `role="grid"`, `aria-rowcount`** | Screen readers need the *total*, not the rendered count |
| **Keyboard navigation must work past the window** | Arrow keys need `scrollToIndex`, not just DOM focus |

**The accessibility point is the one candidates forget and interviewers notice.** A virtualized table where a screen reader announces "20 rows" when there are 4 million is broken for those users — `aria-rowcount={rows.length}` and `aria-rowindex` on each row fix it.

```tsx
// Windowing + infinite loading together (API Lesson 08's cursor pagination)
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = usePayments(filters);
const rows = data?.pages.flatMap(p => p.data) ?? [];

useEffect(() => {
  const last = virtualizer.getVirtualItems().at(-1);
  if (last && last.index >= rows.length - 10 && hasNextPage && !isFetchingNextPage) {
    fetchNextPage();                      // ← fetch when 10 rows from the end
  }
}, [virtualizer.getVirtualItems(), rows.length, hasNextPage, isFetchingNextPage, fetchNextPage]);
```

### When *not* to virtualize
```
❌ Under ~100 rows — the machinery costs more than it saves
❌ Content that must be Ctrl+F-searchable (off-screen rows don't exist in the DOM)
❌ Content that must be printable in full
❌ When the rows have wildly variable, unmeasurable heights (possible, but painful)
```
**The Ctrl+F point is a real product trade-off** worth raising: users expect browser search to find things. If that matters, provide in-app search instead, and say so.

> **CSS alternatives worth knowing:** `content-visibility: auto` with `contain-intrinsic-size` lets the browser skip rendering off-screen content with *zero JavaScript*. It doesn't reduce DOM node count (so it won't save you at 4M rows) but it's a one-line win for long documents and moderately-sized lists.

---

## 4. Render less often — state placement

Covered mechanically in [Lesson 03](../01-mental-model/03-rendering-and-commit.md); here's the performance framing.

```tsx
// ❌ One state change re-renders three expensive siblings
function Dashboard() {
  const [search, setSearch] = useState("");
  return (<>
    <input value={search} onChange={e => setSearch(e.target.value)} />
    <RevenueChart /><PaymentsTable /><ActivityFeed />
  </>);
}
```

Four fixes, in order of preference:
```tsx
// 1. Move state DOWN — the cascade disappears entirely
function Dashboard() { return <><SearchBox /><RevenueChart /><PaymentsTable /></>; }

// 2. Pass expensive subtrees as children — free bailout, no memo
<Shell><RevenueChart /></Shell>

// 3. Put it in the URL with a debounced commit (Lesson 16)
//    Typing re-renders only the input; the table re-renders once per pause

// 4. Split the state by consumer — a store with selectors (Lesson 09)
const search = useStore(s => s.search);      // only components reading `search` re-render
```

**Fix 1 and 2 require no libraries, no dependency arrays and no comparisons.** They're strictly better than memoization when available.

---

## 5. Do less work per render

```tsx
// ❌ Runs on EVERY render, even when `payments` is unchanged
function Table({ payments, filters }: Props) {
  const filtered = payments
    .filter(p => matchesFilters(p, filters))
    .sort((a, b) => b.created - a.created);      // 50k items → ~40ms
}

// ✅ Option A — memoize (measure first)
const filtered = useMemo(
  () => payments.filter(p => matchesFilters(p, filters)).sort(byCreated),
  [payments, filters]);

// ✅ Option B — do it on the SERVER (usually better)
// The API already supports filtering and sorting (API Lesson 08).
// Sending 50k rows to the client to filter them is the actual bug.

// ✅ Option C — a better data structure
const byId = useMemo(() => new Map(payments.map(p => [p.id, p])), [payments]);
// O(1) lookup instead of O(n) .find() inside a 200-row render
```

**Option B is the one people skip.** If your API supports server-side filtering, sorting and pagination — and yours does — then a client filtering 50,000 rows is downloading 49,980 rows it will never show. **The fastest client-side computation is the one the server did.**

```tsx
// The N+1 inside a render — the subtle killer
{payments.map(p => (
  <Row key={p.id} customer={customers.find(c => c.id === p.customerId)} />   // ❌ O(n×m)
))}
// With 200 payments and 5,000 customers that's a million comparisons per render.

const customerById = useMemo(() => new Map(customers.map(c => [c.id, c])), [customers]);
{payments.map(p => <Row key={p.id} customer={customerById.get(p.customerId)} />)}   // ✅ O(n)
```

---

## 6. Skip renders — memoization, in its proper place

Everything from [Lesson 10](../03-hooks/10-memoization.md), applied:

```tsx
// The pattern that actually works for lists
const PaymentRow = memo(function PaymentRow({ payment, onSelect }: RowProps) {
  return <tr onClick={() => onSelect(payment.id)}>…</tr>;
});

function Table({ rows }: { rows: Payment[] }) {
  const onSelect = useCallback((id: PaymentId) => setSelectedId(id), []);   // stable
  return rows.map(p => <PaymentRow key={p.id} payment={p} onSelect={onSelect} />);
  //                   ↑ two props, both stable per row → memo actually skips
}
```

**The three conditions for `memo` to pay off, all of which must hold:**
1. The component is genuinely expensive (>1ms in the Profiler)
2. It re-renders often with **identical** props
3. **Every** prop is referentially stable

Fail any one and you've added cost for nothing ([Lesson 10 §3](../03-hooks/10-memoization.md)).

### The React Compiler
```bash
npm i -D babel-plugin-react-compiler eslint-plugin-react-compiler
```
Auto-memoizes everything provably safe, so manual `useMemo`/`useCallback` for *performance* becomes largely unnecessary. **Two caveats to state honestly:** it requires components to follow the Rules of React and **bails out silently** on ones that don't (hence the lint rule), and it doesn't fix structural problems — bad state placement, a missing virtualizer, or 50,000 rows on the client are all still your job.

---

## 7. Make renders non-blocking

```tsx
// A filter over a large dataset: keep the input responsive (Lesson 11)
const deferredFilters = useDeferredValue(filters);
const rows = useMemo(() => filter(all, deferredFilters), [all, deferredFilters]);
const isStale = filters !== deferredFilters;
return <div style={{ opacity: isStale ? 0.6 : 1 }}><MemoTable rows={rows} /></div>;
```
**Remember the pairing:** `useDeferredValue` only helps if the expensive child is `memo`'d, or it re-renders during the urgent pass anyway.

```tsx
// Tab switching — instant feedback, content streams in
startTransition(() => setTab("refunds"));
```

**And the honest limit:** transitions don't make anything faster. A 200ms filter still takes 200ms; it just stops blocking input. If the work itself is the problem, move it to the server, a worker, or a better algorithm.

```tsx
// A web worker — when the computation is genuinely heavy and client-side is required
const worker = useMemo(() => new Worker(new URL("./aggregate.worker.ts", import.meta.url),
                                        { type: "module" }), []);
useEffect(() => {
  worker.postMessage({ payments });
  worker.onmessage = e => setAggregated(e.data);
  return () => worker.terminate();
}, [worker, payments]);
```
Workers are the right answer for CSV parsing, large aggregations, image processing and crypto. **The cost is serialization** — posting 50MB to a worker isn't free, so measure the transfer too (or use `Transferable` objects / `SharedArrayBuffer`).

---

## 8. Animation and scroll performance

React renders aren't the only source of jank.

```tsx
// ❌ A scroll handler in React state → a render per scroll event (60+/second)
const [scrollY, setScrollY] = useState(0);
useEffect(() => {
  const on = () => setScrollY(window.scrollY);
  window.addEventListener("scroll", on);
  return () => window.removeEventListener("scroll", on);
}, []);

// ✅ Don't put scroll position in state at all
useEffect(() => {
  const on = () => { headerRef.current?.classList.toggle("scrolled", window.scrollY > 50); };
  window.addEventListener("scroll", on, { passive: true });     // ← passive: don't block scrolling
  return () => window.removeEventListener("scroll", on);
}, []);

// ✅ Or IntersectionObserver, which doesn't run on every scroll event at all
```

**The CSS properties that are cheap vs expensive — worth memorising:**
```
Compositor only (cheap, GPU):     transform, opacity
Triggers paint:                    color, background, box-shadow, border-radius
Triggers LAYOUT (most expensive):  width, height, top, left, margin, padding, font-size
```
Animating `left` re-runs layout on every frame; animating `transform: translateX()` doesn't. **This is why virtualizers use `transform`** and why a CSS animation on `width` stutters where one on `transform: scaleX()` doesn't.

```tsx
// Layout thrashing — the classic, and invisible in the React Profiler
items.forEach(el => {
  const h = el.offsetHeight;            // READ  → forces layout
  el.style.height = h * 2 + "px";        // WRITE → invalidates layout
});                                       // repeat → layout runs N times

// ✅ Batch reads, then writes
const heights = items.map(el => el.offsetHeight);      // all reads
items.forEach((el, i) => el.style.height = heights[i]! * 2 + "px");   // all writes
```

---

## 9. Ledger Console's 4-million-row table

The complete answer to the interview question, as an actual design:

```
1. SERVER-SIDE filtering, sorting and cursor pagination     (API Lesson 08)
   → the client never holds 4M rows. It holds ~100 at a time.

2. useInfiniteQuery with cursor pagination                   (Lesson 17)
   → fetch the next page when 10 rows from the end

3. Virtualized rendering, ~20 rows in the DOM                (§3)
   → keyed by payment ID, positioned with transform

4. memo on the row, with stable props                        (§6)
   → so scrolling doesn't re-render rows that didn't change

5. Filters in the URL, debounced, deferred                   (Lessons 11, 16)
   → typing never blocks; one query per pause

6. keepPreviousData while the next page loads                (Lesson 17)
   → no skeleton flash when filters change

7. Prefetch the detail on row hover                          (Lesson 17)
   → opening a payment is instant

Measured: 60fps scroll · ~23 DOM nodes · 6ms commits · 180KB JS
```

**Point 1 is the answer, and the others are refinements.** A candidate who says "virtualize it" has given a good answer; one who says *"first, the client shouldn't have 4 million rows — the API does the filtering and sorting, and I paginate with cursors"* has given the right one. The client-side techniques exist because you still have 100 rows to render smoothly, not because you're trying to make 4 million work.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Work the hierarchy top-down: fewer components → less often → less work → skip → non-blocking** | Virtualization is a 500× win; `memo` is maybe 2× |
| **Virtualize lists over ~100 rows** | Below that, the machinery costs more than it saves |
| **Key virtualized rows by data ID, never the virtual index** | The window slides; state would follow the viewport |
| **Position with `transform`, never `top`** | Compositor-only vs full layout |
| **`aria-rowcount` and `aria-rowindex` on virtualized tables** | Otherwise screen readers announce 20 rows out of 4 million |
| **Filter, sort and paginate on the server when the API supports it** | Downloading 50k rows to show 20 is the actual bug |
| **Build a `Map` for lookups inside a render** | `.find()` in a `.map()` is O(n×m) per render |
| **Never put scroll position in React state** | A render per scroll event |
| **`{ passive: true }` on scroll and touch listeners** | Otherwise the browser waits for your handler before scrolling |
| **Animate `transform`/`opacity`; avoid `width`/`top`/`height`** | Layout vs compositor |
| **Batch DOM reads before writes** | Interleaving forces synchronous layout N times |
| **Remember transitions don't reduce work, only blocking** | If the work is the problem, move it or shrink it |
| **Re-measure after every change** | About half of intuitive optimisations don't help |

---

## 11. Interview traps

**Q1. "How would you render a million rows?"**
**Don't lead with virtualization.** Lead with: *"first, why does the client have a million rows? The server should filter, sort and paginate — the client holds a page at a time."* Then virtualization for smooth rendering of what you do hold, cursor pagination for loading more, `memo`'d rows with stable props, and `transform` positioning. **Then the accessibility caveat** — `aria-rowcount` so screen readers know the real total.

**Q2. "Why is virtualization more effective than `memo`?"**
Different orders of magnitude. Virtualization takes 10,000 components to 20 — a 500× reduction in work. `memo` skips a re-render of a component you're still creating an element for, and only when every prop is referentially stable. **You can't memoize your way out of a 10,000-node tree.**

**Q3. "What's the downside of virtualization?"**
Ctrl+F doesn't find off-screen content, printing shows only the window, accessibility needs explicit ARIA row counts, variable row heights need measurement, and it adds real complexity. **Not worth it under ~100 rows.** Raising the Ctrl+F trade-off unprompted is a good signal.

**Q4. "Typing in a filter box lags. Walk me through the fixes."**
In order: move the input's state into its own component (the cascade disappears); pass the expensive subtree as `children`; put filters in the URL with a debounced commit; `useDeferredValue` + `memo` for render-bound work; virtualize the list. **Only then** `memo` + `useCallback`. And measure between each.

**Q5. "Why does animating `left` stutter but `transform` doesn't?"**
`left` triggers layout and paint on every frame; `transform` is handled by the compositor with no layout or paint. Same reason virtualizers position rows with `transform` and why `opacity` is the other cheap property.

**Q6. "What's layout thrashing?"**
Interleaving DOM reads (`offsetHeight`, `getBoundingClientRect`) with writes in a loop, forcing synchronous layout on each iteration. Batch all reads, then all writes. **Invisible in the React Profiler** — you find it in the browser Performance panel as repeated purple layout blocks.

**Q7. "When would you use a web worker?"**
Genuinely heavy client-side computation that can't move to the server: CSV parsing, large aggregations, image processing, crypto. **And name the cost** — serialization across the worker boundary isn't free, so for a 50MB payload measure the transfer before assuming it's a win.

**Q8. "Does `useTransition` make the filter faster?"**
No — it makes it non-blocking. The work still takes the same time; it just doesn't block input, and React can abandon it if a newer keystroke arrives. If the work itself is too slow, the fix is server-side filtering, a worker, or a better algorithm.

**Q9. "You've virtualized and it's still slow. Now what?"**
Profile again and look at what's left: each row may be individually expensive (inline formatting, date parsing, an `Intl` constructor per cell), or the parent re-renders on scroll, or there's layout thrashing from measuring rows. **Hoist `Intl.NumberFormat` instances** — constructing one per cell is a surprisingly common and expensive mistake.

**Q10. "How do you make scrolling smooth?"**
Keep scroll position out of React state, use `{ passive: true }` listeners, use `IntersectionObserver` instead of scroll math where possible, animate only `transform`/`opacity`, keep DOM node count low (virtualize), and avoid expensive CSS (`box-shadow`, `filter`, `backdrop-filter`) on many elements.

---

## 12. Build & break

### Build — the virtualized payments table
Implement §3 with TanStack Virtual: cursor-paginated infinite loading, `memo`'d rows with stable props, data-ID keys, `transform` positioning, and full ARIA (`role="grid"`, `aria-rowcount`, `aria-rowindex`, keyboard navigation with `scrollToIndex`).

Then measure with 500,000 seeded rows: DOM node count, scroll FPS, commit duration, memory. **Compare against the non-virtualized version** — which will likely crash the tab, and that crash is the most persuasive demo you can record.

### Build — the fix-hierarchy experiment
Take a page with a search input and an expensive 5,000-row list. Fix the keystroke lag **five different ways**, measuring each:
1. `memo` the list
2. Move the input's state down
3. Pass the list as `children`
4. `useDeferredValue` + `memo`
5. Virtualize

Record renders-per-keystroke and commit time for each. **The ranking will surprise you** — and having measured it yourself is what makes the answer credible.

### Build — the `Intl` trap
Render 1,000 rows where each cell does `new Intl.NumberFormat("en-US", {...}).format(n)`. Profile. Then hoist a single formatter to module scope and profile again. The difference is usually 10×, and it's one of the most common real-world slow-render causes.

### Break — five experiments
1. **Index keys in a virtualized list.** Add a checkbox per row, tick one, scroll away and back. The tick is on a different payment.
2. **`top` instead of `transform`.** Switch the virtualizer's positioning and record scroll with the Performance panel. Compare the layout blocks.
3. **Scroll in state.** Store `window.scrollY` in state and profile a scroll. Count the renders.
4. **Layout thrashing.** Write the read/write loop from §8 over 500 elements and record it. Then batch it.
5. **N+1 in render.** `customers.find()` inside a 200-row map with 5,000 customers. Profile, then switch to a `Map`.

### Explain out loud (90 seconds)
1. The five-level fix hierarchy, in order.
2. How you'd render a million rows — **starting with the server**.
3. Three details that make virtualization correct.
4. Why `transform` beats `top`, and what layout thrashing is.
5. What transitions do and don't fix.

---

## What's next

Rendering performance is what happens *after* the app loads. The other half — often the bigger half for real users — is getting it to load at all: bundles, code splitting, and the Core Web Vitals that Google measures and users feel.

Next → **[Lesson 22: Loading performance & Core Web Vitals](22-loading-performance.md)**
