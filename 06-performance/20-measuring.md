# Lesson 20 — Why did it render? Measuring first

> **Why this lesson exists:** most React performance work is wasted because it starts with a guess. Someone adds `memo` everywhere, the app gets no faster, and the codebase gets worse. This lesson is the **method** — the tools, the order to use them, and how to read what they tell you — so that every optimisation you make is one you can prove worked.

**Time:** ~60 minutes · **Prereq:** Lessons 03, 10

---

## 1. The idea in one sentence

> **Never optimise what you haven't measured, and never claim a fix without measuring again — because in React the intuitive culprit is wrong more often than it's right.**

---

## 2. First: which kind of slow?

Before any tool, classify the problem. **These have completely different causes and different fixes.**

| Symptom | Kind | Where to look |
|---|---|---|
| The page takes 4s to appear | **Load** | Network, bundle size, server ([Lesson 22](22-loading-performance.md)) |
| Typing lags | **Render** | React Profiler ([Lesson 21](21-render-performance.md)) |
| Clicking a button takes 300ms to respond | **Render or network** | Profiler + Network |
| Scrolling stutters | **Render or paint** | Performance panel |
| The tab uses 2GB of memory | **Leak** | Memory panel |
| It's fast for you, slow for a customer | **Data volume or device** | Profile with their data, throttled |

**Getting this classification right is half the work.** A team that spends a sprint memoizing components when the real problem is a 3MB JavaScript bundle has measured nothing and achieved nothing.

```
The first question is always: does this happen BEFORE or AFTER the app is interactive?
  Before → load performance
  After  → render performance
```

---

## 3. The tools, in order of usefulness

### 1. "Highlight updates when components render" — free, instant, always on
```
React DevTools → ⚙ → General → ✅ Highlight updates when components render
```
Type in a search box. **If the whole page flashes, you've found a problem before opening any profiler.** This is the single best learning tool in React, and it costs nothing to leave on during development.

The colours tell you frequency: blue/green = occasional, yellow/red = very frequent. A red border around your entire layout on every keystroke is a state-placement problem ([Lesson 03](../01-mental-model/03-rendering-and-commit.md)).

### 2. The React Profiler — for render problems
```
React DevTools → Profiler → ⚙ → ✅ "Record why each component rendered"
→ Record → do the slow thing → Stop
```

What to read, in order:

| Reading | Means |
|---|---|
| **Number of commits** (the bars along the top) | One interaction should usually be 1–2 commits. Five means an effect chain ([Lesson 08](../02-state-effects/08-effects.md)) |
| **Commit duration** | Anything over ~16ms drops a frame |
| **Flame graph width** | How long that component took *including* children |
| **Ranked view** | The same data sorted by self-time — **start here** |
| **Grey components** | Didn't re-render. This is what success looks like |
| **"Why did this render?"** | props / state / hooks / **parent** — the answer is usually "parent" |

**The "why did this render" panel is the highest-value feature in the Profiler**, and it's off by default. Turn it on permanently.

```
Typical readings and what they mean:

"Parent component rendered"     → state is too high; move it down or pass children
"Props changed: style, onClick"  → unstable references defeating memo
"Hook 3 changed"                 → a context or store subscription
"Context changed"                → a context without selectors (Lesson 09)
A wide bar with no children      → genuinely expensive work in that component
Many narrow bars                 → a long list; virtualize (Lesson 21)
```

### 3. The browser Performance panel — for everything else
The React Profiler only knows about React. Use Chrome's Performance panel when:
- Scrolling or animation stutters (layout/paint, not React)
- Something blocks the main thread that isn't a render
- You need to see *why* a frame was long — scripting vs layout vs paint

```
Performance → ⚙ CPU: 4× slowdown, Network: Fast 4G → Record → interact → Stop
```
**Always throttle.** Your machine is not your users' machine, and a problem invisible on an M3 is painful on a mid-range Android. 4× CPU throttling is a reasonable proxy.

What to look for: long yellow (scripting) blocks, purple (layout/style recalc), and — the one people miss — **the "Long Tasks" markers**, any task over 50ms that blocks input.

### 4. `<Profiler>` — programmatic measurement
```tsx
<Profiler id="PaymentsTable" onRender={(id, phase, actualDuration, baseDuration) => {
  if (actualDuration > 16) {
    logger.warn({ id, phase, actualDuration, baseDuration }, "slow_render");
  }
}}>
  <PaymentsTable />
</Profiler>
```
- `actualDuration` — what it took this time
- `baseDuration` — what it would take with no memoization
- **`baseDuration` much larger than `actualDuration` means your memoization is working.** If they're equal, nothing is being skipped.

Use it to catch regressions in CI or to sample real-user render times, not for day-to-day debugging.

### 5. Real user monitoring — the numbers that actually matter
```tsx
import { onLCP, onINP, onCLS } from "web-vitals";
onLCP(m => beacon("lcp", m.value));
onINP(m => beacon("inp", m.value));      // ← the interaction metric (Lesson 22)
onCLS(m => beacon("cls", m.value));
```
**Lab measurements on your machine are a proxy; field data is the truth.** Track p75 per route, because an aggregate number hides the one slow page.

---

## 4. The diagnostic playbook

Memorise this. It's the answer to *"the app feels slow"* in an interview and the actual procedure at work.

### "Typing lags"
```
1. Highlight updates on. Type. What flashes?
   → The whole page          → state too high (Lesson 03): move it down, pass children, or URL+debounce
   → Just the list            → the list is expensive: virtualize (Lesson 21)
   → Just the input           → the input itself is doing work: check onChange
2. Profiler: how many commits per keystroke? >1 means an effect is setting state.
3. Ranked view: which component has the biggest self-time?
4. "Why did this render?" on that component.
5. Fix the CAUSE. Re-measure.
```

### "Clicking is slow"
```
1. Is it network or render? Network panel first — if there's a 300ms request, it isn't React.
2. If render: Profiler. One commit or several?
3. Long single commit → an expensive component, or a huge subtree rendering at once
   → useTransition to unblock (Lesson 11), or split the work
4. Several commits → an effect chain. Collapse it (Lesson 08).
```

### "Scrolling stutters"
```
1. Performance panel, 4× CPU throttle, record a scroll.
2. Long scripting blocks     → a scroll handler doing work; throttle it or use IntersectionObserver
3. Long layout/recalc blocks → layout thrashing: reading offsetHeight in a loop after writing
4. Long paint                → too many DOM nodes, or expensive CSS (box-shadow, filter, large images)
5. React renders on scroll   → you're storing scroll position in state. Don't (Lesson 07)
```

### "It's slow for one customer"
```
1. Get their data volume. 50 rows vs 50,000 rows is a different application.
2. Reproduce with realistic seed data — this is the step people skip.
3. Profile with their filter/sort combination.
4. Check the network: are they far from your region? (API Lesson 02)
```

**That second step is the most common miss.** Staging with 20 rows will never reveal the problem that 50,000 rows creates, and "it's fast for me" is the most expensive sentence in performance work.

---

## 5. Reading a flame graph

```
Commit 3 of 7  —  42.1ms

App                                                          42.1ms  ██████████████████
├─ Layout                                                    41.8ms  ██████████████████
│  ├─ Sidebar                                                 0.3ms  ▏
│  └─ PaymentsPage                                           41.2ms  █████████████████
│     ├─ FilterBar                                            0.8ms  ▎
│     ├─ RevenueChart                                        28.4ms  ████████████        ← 67%
│     └─ PaymentsTable                                       11.9ms  █████
│        ├─ PaymentRow ×200                                  11.4ms  ████                ← 200 × 0.06ms
│        └─ Pagination                                        0.2ms  ▏
```

How to read it:
- **Width = time including children.** `Layout` is wide because its child is, not because `Layout` is slow.
- **`RevenueChart` at 28ms with no children is genuinely expensive work** — a computation or a big SVG. Target it.
- **`PaymentRow ×200` at 11.4ms total** is 0.06ms each — individually fine, collectively a problem. The fix is *fewer rows* (virtualization), not faster rows (memo).

**That distinction is the one to internalise:** a wide bar with no children means fix *that component*; many narrow bars means render *fewer of them*.

---

## 6. Setting a budget

Numbers you can hold onto:

| Budget | Why |
|---|---|
| **Commit < 16ms** | 60fps. Above this, frames drop |
| **Interaction → paint < 200ms (INP)** | Google's "good" threshold; feels instant |
| **Any single task < 50ms** | Above this, input is blocked (a "long task") |
| **Initial JS < 200KB gzipped** | On mid-range mobile, more is seconds of parse+execute |
| **LCP < 2.5s** at p75 | Core Web Vitals "good" |

**Measure at p75 on throttled hardware, not p50 on your laptop.** Your users' median device is roughly a 4× slowdown from a developer machine, and the users who complain are at the tail.

```tsx
// Guard against regressions in CI — Lighthouse budgets
// lighthouserc.json
{ "ci": { "assert": { "assertions": {
  "categories:performance": ["error", { "minScore": 0.9 }],
  "total-byte-weight": ["error", { "maxNumericValue": 500000 }],
  "interactive": ["error", { "maxNumericValue": 3500 }]
}}}}
```
A budget nobody enforces is a wish. Put it in CI.

---

## 7. The measurement mistakes

| Mistake | Why it misleads |
|---|---|
| **Profiling in development** | Dev React is 2–5× slower, StrictMode double-renders, and there's no minification. **Profile a production build** (the Profiler works with `react-dom/profiling`) |
| **Not throttling** | Your machine hides everything |
| **Testing with 20 rows** | The customer has 50,000 |
| **Measuring once** | Variance is large. Repeat and take the median |
| **Optimising without re-measuring** | Half of "optimisations" make things slower |
| **Trusting the Lighthouse score alone** | It's a lab proxy. Field data (RUM) is the truth |
| **Assuming the cause** | *"It must be the table"* — it's usually the chart |

> **Profiling a development build is the most common error**, and it doesn't just exaggerate — it can mislead about *which* component is slow, because dev-only work (hook validation, warnings) isn't evenly distributed.

---

## 8. Ledger Console's measurement setup

```tsx
// 1. A dev-only render counter for suspicious components
function useRenderCount(name: string) {
  const count = useRef(0);
  count.current++;
  if (import.meta.env.DEV) console.log(`${name} render #${count.current}`);
}

// 2. Production sampling of slow renders
<Profiler id="payments-table" onRender={(id, phase, actual) => {
  if (actual > 50) beacon("slow_render", { id, phase, actual, rows: rowCount });
}}>

// 3. Web Vitals to the backend, tagged by route
onINP(m => beacon("web_vital", { name: "INP", value: m.value, route: location.pathname }));

// 4. A realistic seed script — the thing that makes staging useful
npm run seed -- --payments 500000 --customers 20000
```

**Point 4 is the one most teams don't do and should.** A staging environment with realistic data volume turns "we'll find out in production" into "we found it on Tuesday."

**The measured baselines for Ledger Console** (your numbers will differ; the point is having them):
```
Initial load (4G, 4× CPU):   LCP 1.8s · INP 90ms · JS 180KB gz
Typing in the filter bar:     1 commit/keystroke, FilterBar only, <2ms
Loading 50k payments:         virtualized, 18 rows rendered, commit 6ms
Opening a payment detail:     instant (cache-seeded), background refetch
Scrolling the table:          60fps, no React renders (virtualizer uses transform)
```

**Write your own version of that table.** Having measured numbers for your project is what makes a performance answer in an interview credible rather than theoretical.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Classify first: load problem or render problem** | Completely different causes and fixes |
| **Leave "highlight updates" on in development** | You'll see cascades as you create them |
| **Turn on "record why each component rendered"** | The highest-value Profiler feature, off by default |
| **Always profile a production build** | Dev React is 2–5× slower and misleads about *which* component is slow |
| **Always throttle CPU and network** | Your machine hides everything your users experience |
| **Seed realistic data volumes in staging** | 20 rows will never reveal the 50,000-row problem |
| **Wide bar with no children → fix that component; many narrow bars → render fewer** | The flame-graph reading that decides your fix |
| **More than one commit per interaction means an effect chain** | Collapse it before optimising anything else |
| **Re-measure after every fix** | Roughly half of intuitive optimisations don't help |
| **Track p75 field data per route, not aggregate p50** | The aggregate hides the one slow page |
| **Put a performance budget in CI** | An unenforced budget is a wish |

---

## 10. Interview traps

**Q1. "The app feels slow. Walk me through it."**
Lead with classification: *"first, is it slow to load or slow to respond? Those have different causes."* Then the playbook — highlight updates, commits per interaction, ranked view, "why did this render", fix the cause, re-measure. **Naming a method rather than a fix is what's being assessed.**

**Q2. "How do you find out why a component re-rendered?"**
React DevTools Profiler with "record why each component rendered" enabled — it reports props, state, hooks, or parent. The answer is usually "parent re-rendered", which points at state placement rather than memoization.

**Q3. "What's wrong with profiling in development?"**
Dev React is 2–5× slower, StrictMode double-renders, and warnings/validation add work that isn't evenly distributed — so it can mislead about *which* component is slow, not just by how much. Profile a production build.

**Q4. "You see 200 narrow bars in the flame graph. What does that tell you?"**
Each row is individually cheap; collectively expensive. The fix is **rendering fewer rows** — virtualization — not making each row faster with `memo`. Compare with one wide bar and no children, which means genuinely expensive work in that component.

**Q5. "One interaction produces five commits. What's happening?"**
An effect chain: an effect sets state, which triggers another effect, which sets more state. Each link is a render with a visible intermediate state. Collapse it by computing derived values in one place ([Lesson 08](../02-state-effects/08-effects.md)).

**Q6. "What's `baseDuration` in the Profiler?"**
What the render *would* have cost with no memoization. If `actualDuration` is much lower, your memoization is working; if they're equal, nothing is being skipped and your `memo` calls are doing nothing but cost.

**Q7. "It's fast for you and slow for a customer. What do you do?"**
Find the difference: data volume (50 vs 50,000 rows), device class, network distance, or their specific filter/sort combination. **Reproduce with realistic seed data** — that's the step teams skip, and it's usually where the answer is.

**Q8. "What performance budgets would you set?"**
Commit under 16ms, INP under 200ms, no task over 50ms, initial JS under ~200KB gzipped, LCP under 2.5s — all at **p75 on throttled hardware**. And enforced in CI, because an unenforced budget is a wish.

**Q9. "Lighthouse says 95. Are you done?"**
No. Lighthouse is a lab measurement on one simulated device. Field data (RUM) is the truth, because real users have varied devices, networks and data volumes. Track Web Vitals per route at p75 and compare with the lab number — a large gap usually means your test data isn't realistic.

**Q10. "The React Profiler shows nothing slow but scrolling stutters. Now what?"**
It's not a React problem. Use the browser Performance panel: look for layout thrashing (reading `offsetHeight` after a write in a loop), expensive paint (large images, `box-shadow`, `filter`), too many DOM nodes, or a scroll handler doing work on every event. **The React Profiler only knows about React.**

---

## 11. Build & break

### Build — your measurement setup
1. Enable "highlight updates" and "record why each component rendered" permanently.
2. Add a production build script that keeps profiling enabled: `vite build --mode profiling` with the `react-dom/profiling` alias.
3. Add `web-vitals` reporting to your backend, tagged by route.
4. Write the seed script: `npm run seed -- --payments 500000`.
5. Add a Lighthouse CI budget.

### Build — the baseline table
Measure and write down Ledger Console's numbers (§8 format) **before** doing any optimisation. Then re-measure after Module 6. **The delta is what you'll quote in interviews**, and it's far more persuasive than "I know about virtualization."

### Build — the diagnosis drill
Deliberately introduce each of these, then diagnose it **using only the tools** (no peeking at your own code):
1. State at the top of a page that should be local
2. An expensive `.filter().sort()` in a component body
3. An effect chain of three links
4. A non-virtualized 5,000-row list
5. A `memo`'d component receiving a fresh object prop
6. A scroll handler that reads `getBoundingClientRect` on every event

For each, record: which tool found it, what the reading looked like, and how long the diagnosis took. **That log is the skill.**

### Break — four experiments
1. **Dev vs prod profiling.** Profile the same interaction in both builds. Compare the numbers — usually 2–5×, and sometimes a different component tops the ranked view.
2. **Throttling.** Profile a typing interaction unthrottled, then at 4× CPU. The second is what your users have.
3. **Data volume.** Render 20 rows, then 20,000. Note which problems only appear at scale.
4. **`baseDuration`.** Wrap a component in `<Profiler>`, log both durations, then add and remove `memo`. Watch the gap open and close.

### Explain out loud (60 seconds)
1. The first classification question, and why it matters.
2. The four tools and what each is for.
3. How you read a flame graph — the two shapes and their fixes.
4. Three measurement mistakes.
5. Your performance budget numbers.

---

## What's next

You can find what's slow. Next: fixing it — virtualization for large lists, the memoization decisions that actually pay, the React Compiler, and how to render a 4-million-row table at 60fps.

Next → **[Lesson 21: Rendering performance](21-render-performance.md)**
