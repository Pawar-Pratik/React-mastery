# Lesson 30 — Machine coding & design rounds

> **Why this lesson exists:** knowing React and *performing* React under observation are different skills. The two live formats — **machine coding** (build a working component in 45 minutes while someone watches) and **frontend system design** (architect a UI system on a whiteboard) — both reward a repeatable method far more than raw knowledge. This lesson gives you both methods, worked examples, and the specific things that separate a pass from a strong hire.

**Time:** ~90 minutes to learn, then practise · **Prereq:** the whole track

---

## Part A — Machine coding

### 1. The format

45–60 minutes, a shared editor or CodeSandbox, one prompt: *"build an autocomplete"*, *"build an infinite-scroll list"*, *"build a multi-step form"*. They watch you work.

**What's actually being assessed**, in rough order of weight:
1. Do you clarify before coding?
2. Is your structure sensible from the first file?
3. Do you handle the **edge cases** — loading, empty, error, race conditions, keyboard?
4. Do you **talk while you work**?
5. Is it accessible?
6. Does it work?

**Note where "does it work" sits.** A half-finished component with clean structure, narrated decisions and handled edge cases beats a complete one that's a single 200-line file with `any` everywhere.

---

### 2. The method — 45 minutes

```
0–5    CLARIFY      Ask 4–5 questions. Write the answers down.
5–8    SKELETON     File structure, types, the component shape. No logic yet.
8–30   HAPPY PATH   Make it work for the main case. Narrate as you go.
30–40  EDGE CASES   Loading, empty, error, races, keyboard, a11y.
40–45  POLISH       Clean up, name what you'd do with more time.
```

**Announce the plan in minute one:** *"I'll spend a few minutes on requirements, then sketch the structure, get the happy path working, and leave time for edge cases and accessibility — those matter more to me than extra features. Stop me if you'd rather I go deeper somewhere."*

That does three things: shows you have a method, tells them you know where the depth is, and buys permission to control the clock.

### 3. The clarifying questions

Ask these. **Write the answers in a comment at the top of the file** — it looks like requirements gathering, not stalling.

| Question | Why it changes the code |
|---|---|
| **Is the data remote or local?** | Remote means debouncing, races, cancellation, loading states |
| **How many items?** | 50 vs 50,000 decides virtualization |
| **Do I need keyboard navigation?** (they'll say yes) | Arrow keys, Enter, Escape, roving focus |
| **Any library, or vanilla React?** | Whether you can use TanStack Query/Virtual |
| **Should I style it?** | Usually "basic is fine" — don't burn 15 minutes on CSS |
| **Anything explicitly out of scope?** | Protects your clock |

**If they say "use whatever you'd normally use"**, still say what you're choosing and why: *"I'd normally reach for TanStack Query here for caching and cancellation, but I'll hand-roll the fetch so you can see the race handling."* That's a better signal than silently importing a library.

---

### 4. Worked example — the autocomplete

The most common prompt. Here's what a strong 45 minutes produces.

```tsx
// REQUIREMENTS (asked and confirmed)
// - Remote search, ~200ms latency, up to 10k results
// - Keyboard nav required; basic styling fine
// - No libraries, so the race handling is visible

type Status = "idle" | "loading" | "success" | "error";
type State =
  | { status: "idle" }
  | { status: "loading"; query: string }
  | { status: "success"; query: string; results: Item[] }
  | { status: "error"; query: string; error: Error };
```
**Starting with a discriminated union for the state is a strong opening move** — it's the first thing they'll notice, and it means you never write `results?.map` or a defensive `if`.

```tsx
function Autocomplete({ onSelect }: { onSelect: (item: Item) => void }) {
  const [query, setQuery] = useState("");
  const [state, setState] = useState<State>({ status: "idle" });
  const [activeIndex, setActiveIndex] = useState(-1);
  const [open, setOpen] = useState(false);

  const debounced = useDebouncedValue(query, 250);
  const listboxId = useId();
  const inputRef = useRef<HTMLInputElement>(null);

  // ── Fetch, with cancellation AND race protection ──
  useEffect(() => {
    if (debounced.trim().length < 2) { setState({ status: "idle" }); return; }

    const ac = new AbortController();
    setState({ status: "loading", query: debounced });

    search(debounced, ac.signal)
      .then(results => setState({ status: "success", query: debounced, results }))
      .catch(e => { if (e.name !== "AbortError") setState({ status: "error", query: debounced, error: e }); });

    return () => ac.abort();        // ← the cleanup IS the race fix. Say this out loud.
  }, [debounced]);

  const results = state.status === "success" ? state.results : [];

  // ── Keyboard ──
  function onKeyDown(e: React.KeyboardEvent) {
    switch (e.key) {
      case "ArrowDown": e.preventDefault(); setOpen(true);
                        setActiveIndex(i => Math.min(i + 1, results.length - 1)); break;
      case "ArrowUp":   e.preventDefault();
                        setActiveIndex(i => Math.max(i - 1, 0)); break;
      case "Home":      e.preventDefault(); setActiveIndex(0); break;
      case "End":       e.preventDefault(); setActiveIndex(results.length - 1); break;
      case "Enter":     if (activeIndex >= 0 && results[activeIndex]) {
                          e.preventDefault(); choose(results[activeIndex]);
                        } break;
      case "Escape":    setOpen(false); setActiveIndex(-1); break;
      case "Tab":       setOpen(false); break;
    }
  }

  function choose(item: Item) {
    onSelect(item); setQuery(item.label); setOpen(false); setActiveIndex(-1);
  }

  return (
    <div className="combobox">
      <input
        ref={inputRef}
        role="combobox"
        aria-expanded={open}
        aria-controls={listboxId}
        aria-autocomplete="list"
        aria-activedescendant={activeIndex >= 0 ? `${listboxId}-opt-${activeIndex}` : undefined}
        value={query}
        onChange={e => { setQuery(e.target.value); setOpen(true); setActiveIndex(-1); }}
        onKeyDown={onKeyDown}
        onBlur={() => setTimeout(() => setOpen(false), 150)}   // allow a click to land
      />

      {/* Announce result counts to screen readers */}
      <div role="status" aria-live="polite" className="sr-only">
        {state.status === "success" && `${state.results.length} results`}
        {state.status === "loading" && "Searching"}
      </div>

      {open && (
        <ul id={listboxId} role="listbox">
          {state.status === "loading" && <li role="presentation">Searching…</li>}
          {state.status === "error"   && <li role="presentation">Couldn't search. Try again.</li>}
          {state.status === "success" && state.results.length === 0 &&
            <li role="presentation">No results for "{state.query}"</li>}
          {results.map((item, i) => (
            <li
              key={item.id}
              id={`${listboxId}-opt-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              onMouseDown={e => e.preventDefault()}      // ← don't blur the input before the click
              onClick={() => choose(item)}
            >{item.label}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**The eight things that make this a strong answer**, and you should *say each one as you write it*:
1. **Discriminated union state** — no `results?.` anywhere
2. **Debounce** — fewer requests
3. **`AbortController` in the cleanup** — "this is what stops an old response overwriting a newer one"
4. **`.trim().length < 2`** — don't search on one character
5. **Full keyboard set** — arrows, Home/End, Enter, Escape, Tab
6. **Complete ARIA combobox pattern** — `role`, `aria-expanded`, `aria-controls`, `aria-activedescendant`
7. **`aria-live` result count** — sighted users see the list; screen-reader users need it announced
8. **`onMouseDown` preventDefault** — the subtle one: without it, blur fires before click and the selection is lost

**Number 8 is the detail that gets remembered.** It's a real bug in most hand-rolled autocompletes, and mentioning it unprompted signals you've built one for real.

### What to say at minute 40
> *"With more time I'd: virtualize the list if results can be large, cache queries so backspacing is instant, add `useTransition` so rendering never blocks typing, and highlight the matched substring. And in production I'd use Radix or Headless UI's combobox rather than hand-rolling the ARIA — this pattern has a lot of edge cases and they've handled them."*

**Naming what you'd do next, and naming when you'd use a library instead, is a judgement signal** — it shows you know the difference between an exercise and production.

---

### 5. The other common prompts

| Prompt | The thing they're checking | Don't forget |
|---|---|---|
| **Infinite scroll** | `IntersectionObserver`, not scroll math | Cleanup, no duplicate fetches, a "no more" state, error retry |
| **Multi-step form** | State modelling | A reducer, back preserves data, validation per step, URL-synced step |
| **Data table** | Composition + scale | Sort, filter, pagination, empty/loading, `aria-sort`, stable keys |
| **Modal / dialog** | Accessibility | Focus trap, focus restore, Escape, portal, inert background, scroll lock |
| **Debounced search** | Races | Cancellation, min length, loading state, the stale-response test |
| **Drag and drop** | Pointer events + a11y | **Keyboard alternative** (most candidates forget), `aria-grabbed`/live region |
| **Tabs / accordion** | Compound components | Roving tabindex, arrow keys, correct roles, `useId` pairing |
| **Star rating** | Small but revealing | Keyboard, `role="radiogroup"`, hover vs selected state |
| **Todo/CRUD with API** | End-to-end judgement | Optimistic update **with rollback**, error states, dedup |

**Two patterns worth having in muscle memory** because they appear across several prompts:

```tsx
// IntersectionObserver for infinite scroll — better than scroll handlers
const sentinelRef = useRef<HTMLDivElement>(null);
useEffect(() => {
  const el = sentinelRef.current;
  if (!el || !hasMore || isLoading) return;
  const io = new IntersectionObserver(([entry]) => { if (entry?.isIntersecting) loadMore(); },
                                      { rootMargin: "200px" });   // ← load BEFORE it's visible
  io.observe(el);
  return () => io.disconnect();
}, [hasMore, isLoading, loadMore]);
```

```tsx
// Roving tabindex — for tabs, menus, toolbars, any composite widget
<button role="tab" tabIndex={isSelected ? 0 : -1} aria-selected={isSelected} />
// One stop in the tab order; arrow keys move within. This is the pattern interviewers listen for.
```

---

### 6. What loses points

| Mistake | Fix |
|---|---|
| Coding in silence | Narrate: *"I'm modelling state as a union so I don't have to check for undefined"* |
| Starting with markup, not state | Types and state shape first |
| Ignoring loading/empty/error | These are half the assessment |
| No keyboard support | Almost always an explicit requirement |
| `any` in the types | You're being assessed on TypeScript too |
| One 200-line component | Extract as you go; it reads as structure |
| Perfecting CSS | Ask; usually "basic is fine" |
| Running out of time silently | At minute 38, say what's left and prioritise out loud |
| Not testing it | Type in the input. Click things. **Use the keyboard.** |

---

## Part B — Frontend system design

### 7. The format

45 minutes, whiteboard or a doc: *"design the frontend for a Slack-like chat"*, *"design an analytics dashboard for 10,000 concurrent users"*, *"design a collaborative document editor."*

**What's assessed:** requirement extraction, component and data architecture, state decisions, performance at scale, and — the differentiator — **naming the trade-offs and what would change your mind.**

### 8. The method

```
0–5    CLARIFY       Users, scale, devices, must-haves, out of scope
5–12   DATA MODEL    Entities, relationships, what the API must provide
12–22  ARCHITECTURE  Routes, component tree, state placement
22–32  HARD PARTS    Scale, real-time, offline, races ← the differentiator
32–40  CROSS-CUTTING Performance, a11y, errors, testing
40–45  TRADE-OFFS    What you'd revisit, what you'd watch
```

**The clarifying questions:**
```
Who uses it, and on what devices?        → mobile-first? offline? low-end Android?
How much data?                            → 100 messages or 10 million?
How many concurrent users?                 → does real-time need fan-out infrastructure?
What must be real-time vs eventually consistent?
Is SEO or a public link preview relevant?  → decides the rendering strategy
What's explicitly out of scope?
```

### 9. Worked example — "design an analytics dashboard"

**Clarify (and state your assumptions):** internal users on desktop, ~10k concurrent, data updating every few seconds, millions of rows behind aggregates, authenticated so no SEO.

**Data model** — *and this is where you win points by pushing work to the server:*
> *"The client should never receive millions of rows. The API returns pre-aggregated series with a time bucket and a filter set, plus a paginated detail endpoint for drill-down. That decision removes most of the client-side performance problem before I've drawn a component."*

**Architecture:**
```
/dashboard?range=7d&metric=revenue&segment=enterprise
  ├─ FilterBar        URL state, debounced
  ├─ MetricCards      4 independent queries, own Suspense boundaries
  ├─ TimeSeriesChart  lazy chunk, own error boundary
  └─ DetailTable      virtualized, cursor-paginated

State:  filters → URL · data → query cache · chart zoom → local · session → context
```

**The hard parts** — volunteer all five:
1. **Scale:** server-side aggregation, virtualized detail table, cursor pagination
2. **Real-time:** one SSE connection updating the cache; components read the cache. Polling if the update rate is low — *"I'd measure before adding a connection; 10k clients polling at 0.5Hz is 5k req/s, which is when a connection starts to pay"*
3. **Responsiveness:** filters in a transition so the chart never blocks the controls; `isStale` opacity so users see it catching up
4. **Independence:** per-panel Suspense and error boundaries — *"a failed chart must not hide the table"*
5. **Loading:** route-level splitting, the chart library in its own chunk, skeletons sized to the real content so there's no CLS

**Cross-cutting:** measured budgets (LCP/INP/CLS at p75, throttled), keyboard access, `aria-live` for updating numbers, MSW-based component tests plus 3 Playwright journeys.

**Trade-offs — the close that matters:**
> *"Two things I'd revisit. First, if the update rate turns out to be low, I'd drop SSE for polling with `ETag`s — fewer moving parts, and an empty poll is ~150 bytes. Second, I've assumed server-side aggregation; if the backend can't do it, the whole design changes and I'd need a worker or a columnar client store. The thing I'd watch after launch is the detail table's query patterns — that's where the cost will show up first."*

**Naming the trigger that would change your decision is the single strongest thing you can add.** It proves the design is reasoned rather than recited.

### 10. The design-round phrases worth stealing

- *"Before I design the client, what does the API give me? That decision removes most of the problem."*
- *"I'd put filters in the URL — shareable, back/forward works, and it removes a re-render cascade."*
- *"That's server state, so it goes in a query cache, not `useState`."*
- *"One Suspense boundary per independently-loadable region, so the page doesn't wait for the slowest part."*
- *"A failed chart shouldn't hide the table — boundaries go wherever the surrounding UI is still useful."*
- *"I'd measure before adding a connection; polling with conditional requests is often enough."*
- *"Real-time updates the cache, not component state — then no component needs to know it exists."*
- *"This needs an idempotency key, because the network can duplicate a request even when the UI can't."*
- *"I'd be optimistic about a toggle, not about money."*
- *"Here's what would change my mind about this decision."*

---

## 11. Practice protocol

**These are performance skills. Reading this lesson is not practice.**

### Machine coding — 8 prompts, 45 minutes each, timed, out loud
1. Autocomplete with remote search
2. Infinite-scroll list with `IntersectionObserver`
3. Multi-step form with a reducer and URL-synced steps
4. Sortable, filterable data table
5. Accessible modal with focus trap
6. Toast system (split contexts, queue, auto-dismiss, pause on hover)
7. Drag-and-drop list **with a keyboard alternative**
8. Optimistic to-do list with rollback

**Score yourself after each:**
```
[ ] Did I clarify before coding?
[ ] Did I model state as a union before writing markup?
[ ] Did I narrate my decisions?
[ ] Loading / empty / error / race all handled?
[ ] Full keyboard support?
[ ] Correct ARIA roles and relationships?
[ ] Did I test it with the keyboard?
[ ] Did I finish on time, and say what was left?
```
**Under 6/8 is a fail. Redo that prompt in a week** — not the next day, so you're recalling rather than repeating.

### Design rounds — 6 prompts, 45 minutes each, on paper, out loud
1. Analytics dashboard, 10k concurrent users
2. Slack-like chat (real-time, presence, unread counts, offline)
3. Collaborative document editor (conflict resolution is the interesting part)
4. E-commerce product listing (SEO, filters, images, Core Web Vitals)
5. A payments dashboard — **you've built this one**; aim for fluency
6. A video platform (adaptive streaming, recommendations, watch history)

Prompts 2 and 3 are the ones that expose gaps, because they break the request/response pattern — chat needs fan-out and ordering; collaborative editing needs CRDTs or OT and a conflict story.

---

## 12. The final checklist

Before an interview, you should be able to do all of these cold:

```
EXPLAIN (60 seconds each, no notes)
[ ] Why React exists, and the trade it makes
[ ] The four re-render triggers — and why "props changed" isn't one
[ ] Component identity, and the lost-focus bug
[ ] The snapshot model and stale closures
[ ] Effects as synchronization, and six cases you don't need one
[ ] Context's limitation, and four ways around it
[ ] Server state vs client state
[ ] The five-level performance fix hierarchy
[ ] RSC vs SSR, mechanically
[ ] Hydration, and why a mismatch is worse than a warning

BUILD (45 minutes, live, narrated)
[ ] An accessible autocomplete with race handling
[ ] An infinite-scroll list
[ ] A modal with a focus trap
[ ] A multi-step form with a reducer

DESIGN (45 minutes, whiteboard)
[ ] A dashboard at scale, with the hard parts volunteered
[ ] A real-time feature, with the polling-vs-connection trade-off

SHOW
[ ] Ledger Console, deployed, with the 5-minute demo rehearsed
[ ] docs/architecture.md with rejections and triggers
[ ] docs/performance.md with measured before/after numbers
```

---

## The React track is complete

30 lessons: the mental model, state and effects, hooks, composition, data, performance, server React, the project, and interview performance.

**Before you call it done, do these four things:**

1. **Retake the 15-question self-test** from the [README](../README.md), cold and timed. Compare with your baseline from before Lesson 01.
2. **Build at least Phases 0–4 of Ledger Console** ([Lesson 27](../08-project/27-console-build-log.md)). If you read all 30 lessons and built nothing, you have recognition, not skill — and interviewers find out within two follow-up questions.
3. **Do three timed machine-coding rounds**, out loud, self-scored. Under 6/8 means redo it.
4. **Rehearse the 5-minute demo.** Lead with the double-click-produces-one-refund moment, not with CRUD.

---

## All three tracks are complete

```
API         32 lessons   →  Ledger, a payments API
TypeScript  22 lessons   →  the shared type layer
React       30 lessons   →  Ledger Console
─────────────────────────────────────────────────
            84 lessons, one product, three tracks
```

**The thing that makes this hang together** is that the same ideas appear at every layer, which is why the cross-references exist:

| Idea | API | TypeScript | React |
|---|---|---|---|
| **Contracts** | Endpoint design | Types as compile-time contracts | Props as a component API |
| **Boundaries** | Validate at the edge | `unknown` → parse → typed | Parse every response |
| **Illegal states** | State machines in SQL | Discriminated unions | Reducers that can't double-submit |
| **Idempotency** | Keys on money-moving POSTs | A branded key type in a ref | One request from two clicks |
| **Caching** | `ETag`, `stale-while-revalidate` | — | `staleTime`, stale-while-revalidate |
| **Evolution** | Tolerant readers, open enums | `LiteralUnion`, exhaustive switches | A `default` branch that doesn't break |
| **Observability** | `X-Request-Id`, structured logs | — | Surfacing `requestId` in the UI |

**When someone asks you a question about any one of these, you can answer it at three layers.** That's what "I know this properly" sounds like — and it's the thing almost nobody in an interview loop can do.

Go build it.
