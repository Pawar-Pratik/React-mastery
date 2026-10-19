# Lesson 29 — Rapid-fire master Q&A

> **How to use this:** cover the answer column, say your answer **out loud**, then compare. Anything you hedge on goes into `RECALL.md` and gets re-tested cold every Sunday. Answers here are the *shape* of a strong response, not the full explanation — the lesson reference has the depth.

**Time:** ~90 min for a full pass · ~25 min to re-test

---

## Scoring

| Level | Sounds like |
|---|---|
| **1 — Recall** | The definition |
| **2 — Mechanism** | *Why* it works that way, and what breaks otherwise |
| **3 — Judgement** | Level 2 + the trade-off + what you'd do + what would change your mind |

---

## Part 1 — The mental model (20)

| # | Question | Strong answer |
|---|---|---|
| 1 | What problem does React solve? | It keeps the DOM in sync with a tree of JS values. Before React, UI code was imperative — you wrote the *transition* from state A to B, which is N² transitions for N states, and every branch is a chance to forget a cleanup. React lets you describe the UI for the current state. **You trade direct DOM control and some overhead for never writing a transition.** [L01] |
| 2 | What does JSX compile to? | `jsx()` calls producing plain objects with `type`, `props`, `key`, `ref`. **`<Badge/>` does not call `Badge`** — it creates an object whose `type` is the function, and React decides when to call it. That's what makes skipping, delaying and double-invoking possible. [L01] |
| 3 | Is the virtual DOM faster than the real DOM? | **No** — not faster than well-written imperative code. It's faster than *badly*-written imperative code, and it buys the declarative model. Solid and Svelte skip it entirely and are faster still. Saying this honestly beats repeating the marketing. [L01] |
| 4 | Explain reconciliation. | Diffing the previous and next element trees, made O(n) by two heuristics: different `type` ⇒ destroy and rebuild the subtree; same `type` ⇒ keep the fiber and update props. Children match by position unless keyed. [L01] |
| 5 | What is Fiber? | React's record for a component instance — holding state, effects, refs and props — and the tree of them. Restructuring it as a traversable linked list (2017) made rendering **interruptible**, which is what `useTransition`, Suspense and concurrent rendering are built on. [L01] |
| 6 | Where does state actually live? | On the **fiber**, keyed by hook call order — not in your function, whose locals die each render. Hence: state survives re-renders, hook order must be stable, and destroying the fiber destroys the state. [L01] |
| 7 | Render phase vs commit phase? | Render: calls components, diffs, marks effects — **interruptible, must be pure**. Commit: applies DOM changes, attaches refs, runs layout effects — **synchronous**. Then paint, then passive effects. [L01, L03] |
| 8 | Why must render be pure? | React may call your function twice (StrictMode), abandon a render (concurrent), or restart it (higher-priority update). Side effects in render would happen an unpredictable number of times. **Purity is what makes those capabilities safe.** [L01] |
| 9 | What does `{count && <X/>}` render when count is 0? | The number **`0`**, visibly. `0` is falsy so `&&` returns it, and React renders numbers. Use `count > 0 &&`. [L02] |
| 10 | How do you design a component's props? | As an API: composition over configuration, closed unions over multiple booleans, illegal combinations unrepresentable (`XOR`, discriminated unions), never pass derived data. **The test before adding a prop: could the consumer express this by arranging children?** [L02, L13] |
| 11 | When do you split a component? | State ownership, reuse, **different re-render cadence**, needing a `key`, and being able to name it honestly. **Not line count.** The cadence answer is the one juniors don't cite. [L02] |
| 12 | When does React re-render a component? | Four triggers: own state changed, parent re-rendered, consumed context changed, subscribed external store emitted. **Correct the common wrong answer: "props changed" isn't one** — a component re-renders because its parent did; changed props are the consequence. [L03] |
| 13 | Is re-rendering bad? | No — it's the mechanism. A render is a function call, some allocation, and a diff that usually commits nothing. It costs when the component is expensive, the subtree is huge, or it renders very frequently. **Measure first.** [L03] |
| 14 | Why didn't `memo` help? | Three reasons: a new object/array/function prop each render so the comparison always fails; the component consumes a changed context (`memo` can't stop that); or `children` is a fresh element. **A failing `memo` is worse than none** — you pay for the comparison. [L03, L10] |
| 15 | How do you stop a keystroke re-rendering the page? | Structurally, in order: move the input's state into its own component; pass the expensive subtree as `children`; put it in the URL with a debounce; a store with selectors. **Then** `memo`. **Leading with structure is the differentiator.** [L03] |
| 16 | Why does passing `children` avoid a re-render? | The element is created by the *parent's parent*, so when the middle component re-renders `props.children` is referentially identical and React bails out of that subtree. **Memoization via composition** — no `memo`, no dependency array. [L03] |
| 17 | What determines component identity? | **Type + position among siblings**, with `key` overriding position. Same identity ⇒ same fiber ⇒ state survives. Different ⇒ destroy and recreate. [L04] |
| 18 | My input loses focus every keystroke. Why? | A component defined **inside** another component. It's a new function object each render, so React sees a different `type`, destroys the subtree and mounts a new one — and a replaced DOM node can't keep focus. Move it to module scope. [L04] |
| 19 | Why is `key={index}` wrong? | React matches children by key to decide reuse vs recreate. With index keys, inserting at the front shifts every key, so React reuses each position's fiber with different data — and per-row state (a ticked checkbox, a typed note) **stays with the position, not the item**. Fine only for a static, stateless, unsorted list. [L04] |
| 20 | Two uses of `key` beyond lists? | Resetting state deliberately (`<Form key={payment.id}>` — better than a sync effect because it happens during render), and forcing a remount of a third-party integration (`<Chart key={`${w}x${h}`}>`). [L04] |

---

## Part 2 — State & effects (20)

| # | Question | Strong answer |
|---|---|---|
| 21 | `setCount(count+1)` three times — result? | **1**. All three read `count` from the same render's closure. **Distinguish the mechanisms:** batching is why there's one render; closures are why the value is 1. Fix with `setCount(c => c+1)`. [L05] |
| 22 | Why is my state one render behind? | It isn't — you're reading this render's snapshot. `count` is a `const` in the current scope; the new value only exists in the next render. **There is no "current state" to read.** [L05] |
| 23 | Why does my `setInterval` counter stick at 1? | The effect ran once with `count === 0` captured, and the interval keeps that closure forever. Fix with the updater form (captures nothing) or a ref. [L05] |
| 24 | When must you use the updater form? | Whenever the next state depends on the previous — always in async callbacks, intervals, subscriptions and effects. Bonus: it removes the value from your dependency array. [L05] |
| 25 | Why can't you mutate state? | React bails out with `Object.is`. A mutated array has the same reference, so React sees no change. Worse, it may *appear* to work when an unrelated render happens — inconsistent failure is harder to debug than consistent failure. [L05] |
| 26 | Where should state live? | Server data → query cache. Shareable/refresh-surviving → **URL**. Many distant readers → store with selectors. Two siblings → lift. Otherwise local. **Naming the URL is what separates this from a textbook answer.** [L05, L16] |
| 27 | How do you reset state when a prop changes? | `key` on the component — it happens during render, no extra commit, no flash. Or adjust state during render by comparing with a previous-value state. **Never** an effect that calls `setState`. [L05] |
| 28 | Why can't hooks be conditional? | They're stored positionally on the fiber and matched by call order. Skipping one shifts every subsequent hook's identity. [L05] |
| 29 | When do you move to `useReducer`? | Four signals: multiple states change together, transitions depend on several current values, the same update happens in many places, or you want to test the logic without rendering. [L06] |
| 30 | How do you prevent a double submit? | Three layers, and why each alone fails: a disabled button (defeatable by keyboard/race/second path); a state machine with **no `submit` transition from `submitting`** (structural); and an **idempotency key** on the request (the network can duplicate it even when your UI can't). [L06, L18] |
| 31 | Reducer vs state machine? | A reducer maps `(state, action) → state` with no constraint on transitions. A state machine switches on **state first**, so an action illegal in the current state has no case — making illegal transitions unrepresentable rather than guarded. [L06] |
| 32 | `useRef` vs `useState`? | Both survive re-renders; only state triggers one. State is a per-render snapshot; a ref is always current. **The test: does the screen need to change when this changes?** [L07] |
| 33 | Why can't you read a ref during render? | Render must be pure and may be doubled or abandoned, so a ref read/written during render gets an unpredictable value. The one exception is idempotent lazy initialisation. [L07] |
| 34 | My effect reconnects every render. Fix it. | The callback prop is a new function each render and it's in the deps. **Latest-ref pattern**: store the handler in a ref updated in a `useLayoutEffect`, subscribe once with `[]`, call `ref.current(...)`. That's what `useEffectEvent` is designed to formalise. [L07] |
| 35 | What is `useEffect` for? | **Synchronizing React state with an external system** — the DOM outside React, the network, subscriptions, timers, browser APIs, non-React libraries. Not "run after render". If no external system is involved, the code belongs in render or an event handler. [L08] |
| 36 | Name four cases where you don't need an effect. | Transforming data (compute in render), user events (event handler), resetting state on a prop change (`key`), adjusting state on a prop change (during render), sharing handler logic (extract a function), chains of effects (compute in one place). **Naming five or six is a strong signal.** [L08] |
| 37 | Why does my effect run twice? | StrictMode in development, deliberately — React unmounts and remounts to verify setup → cleanup → setup leaves the system unchanged. If it breaks, the cleanup is missing. **Never** disable StrictMode. [L08] |
| 38 | Search shows results for an old query. What happened? | A race: multiple in-flight requests resolving out of order, so the last to *resolve* wins rather than the last *sent*. Fix with a cleanup that sets an `ignore` flag or aborts. **This is the clearest demonstration that effects are synchronization** — the old sync is torn down, so its result is unwanted. [L08] |
| 39 | How do you decide dependencies? | You don't — the body does. Every reactive value it reads goes in. If the linter wants something you don't want to react to, the effect is wrong: depend on primitives, move the object inside, hoist non-reactive functions, or use the latest-ref pattern. [L08] |
| 40 | `useEffect` vs `useLayoutEffect`? | Layout effects run in the commit phase **before paint**, so they block it — use only when the user would otherwise see a wrong frame (tooltip positioning, scroll restoration). Also: they don't run during SSR. [L08] |

---

## Part 3 — Hooks & composition (20)

| # | Question | Strong answer |
|---|---|---|
| 41 | What is context for? | Dependency injection — getting a value to distant descendants. **Not a state manager: it has no selector**, so every consumer re-renders when any part of the value changes. [L09] |
| 42 | A context value changes — who re-renders? | Every `useContext` consumer of that context, **regardless of which part they use** — and `memo` doesn't help, because context isn't a prop. [L09] |
| 43 | All my consumers re-render when nothing changed. | The provider's value is a new object each render. `useMemo` it. **The most common context bug.** [L09] |
| 44 | How do you avoid re-rendering dispatch-only consumers? | Two contexts: state and `dispatch`. `dispatch` is referentially stable forever, so components that only dispatch never re-render — often most of them. [L09] |
| 45 | Context or a store? | Context for rarely-changing app-wide values (auth, theme, locale, a stable API object). A store for frequently-changing state read in different slices, because stores give **selector-based subscriptions** — the thing context lacks. Neither for server data. [L09] |
| 46 | `useMemo` vs `useCallback` vs `memo`? | Value / function / component-render-skip; all compare deps with `Object.is`. **Two purposes:** skipping expensive work, and preserving referential identity for something downstream. [L10] |
| 47 | When is memoization about correctness? | Four cases: a context value, a function in an effect's deps, an object passed to an effect, a selector passed to an external store. Omitting them causes re-render storms, effect loops, or resubscribe loops. **Though the better fix is often removing the dependency.** [L10] |
| 48 | What does the React Compiler change? | Auto-memoizes, so manual memoization for performance is largely unnecessary. Caveats: it requires the Rules of React and **bails out silently** otherwise (hence the lint rule), and it doesn't fix structural problems. [L10] |
| 49 | What is concurrent React? | Interruptible rendering with priorities: React can start a render, pause for something urgent, then resume or **discard** it. Enabled by Fiber. [L11] |
| 50 | Does `useTransition` make things faster? | **No** — non-blocking. A 200ms filter still takes 200ms; the input stays responsive because React prioritises it and can abandon the low-priority render. **The most common misunderstanding.** [L11] |
| 51 | Transition vs debounce? | Debounce waits a fixed time regardless of machine speed — artificial lag on a fast machine, still blocking on a slow one. A transition starts immediately and abandons only when something more urgent arrives. **Debounce is still correct for reducing network requests.** [L11] |
| 52 | `useTransition` vs `useDeferredValue`? | Same mechanism, different handle: wrap the *update* (you own the setState, get `isPending`) vs read a lagging *value* (it arrives as a prop). **Both need the expensive consumer `memo`'d.** [L11] |
| 53 | What is tearing? | Concurrent rendering can pause mid-render; if an external store updates during the pause, some components render with old data and some with new — one screen, two versions. `useSyncExternalStore` forces a consistent synchronous re-render. [L11] |
| 54 | What's `useId` for? | Stable IDs across server and client render, for `htmlFor`, `aria-describedby`, `aria-labelledby`. Random or counter IDs cause hydration mismatches. **An accessibility hook**, never for list keys. [L11] |
| 55 | What do custom hooks share? | **Logic, not state.** Two components calling `useCounter` have two independent counters. For shared state: lift, context, or a store. [L12] |
| 56 | When do you extract a custom hook? | Two call sites — **or one, if the logic is tricky** (a race-safe fetch, a fiddly cleanup, a focus trap). There the value isn't reuse, it's getting it right once. [L12] |
| 57 | Why did a hook's return break my consumer's effect? | It returns a fresh object each render, so `useEffect([value])` sees a new reference every time. **A hook's referential stability is part of its contract.** [L12] |
| 58 | Compound components vs slots vs render props? | Compound: context-shared state, arbitrary arrangement — interactive widgets. Slots: named regions as props, fixed structure, **more discoverable** — layouts. Render props: consumer controls rendering while the component owns behaviour — mostly replaced by headless hooks. [L13] |
| 59 | Controlled or uncontrolled component API? | Support both: `value !== undefined` selects the mode (not `!= null`, since `null` can be a real value), `onChange` fires in **both**, and never switch modes mid-life. [L13] |
| 60 | What does a correct Modal need beyond a portal? | Focus trap, focus-in on open, focus-return on close, `Escape`, inert background, scroll lock with restore, `role="dialog"` + `aria-modal` + `aria-labelledby`. **Then say you'd use `<dialog>` or Radix** rather than reimplement focus management. [L13, L15] |

---

## Part 4 — Forms, boundaries, data (20)

| # | Question | Strong answer |
|---|---|---|
| 61 | Controlled or uncontrolled inputs? | Controlled when you need the value **during** typing (live validation, dependent fields, formatting). Uncontrolled when you only need it at submit — which is most fields. **"Always controlled" is out of date;** React 19 Actions are built around `FormData`. [L14] |
| 62 | Where does validation run? | HTML attributes (free), client schema (UX), **server schema (security)**. Anyone can `curl` your endpoint. [L14] |
| 63 | `z.input` vs `z.output` in a form? | The DOM gives strings; the API wants integers in minor units. One schema with `.transform()` gives you both types, so the conversion lives in one place and can't drift. [L14] |
| 64 | Five accessibility requirements for a field? | `label htmlFor`/`id`, `aria-describedby` for hint and error, `aria-invalid`, `role="alert"` on the error, focus moved to the first invalid field on submit. **And `placeholder` is not a label.** [L14] |
| 65 | What is `useActionState`? | React 19's form hook: `[state, formAction, isPending]`. Pending state for free, runs in a transition, resets uncontrolled inputs on success, and **works without JavaScript** with Server Actions. [L14] |
| 66 | What do error boundaries catch? | Render, lifecycle and constructor errors in their subtree. **Not event handlers, async code, SSR, or errors in the boundary itself.** The event-handler exclusion is the important half — that's where most errors are. [L15] |
| 67 | Where do you put error boundaries? | Layered: root, per route, and around any widget whose failure shouldn't take down siblings. **The rule: wherever the surrounding UI is still useful without this piece.** [L15] |
| 68 | How does Suspense work? | A component signals it isn't ready; React walks up to the nearest boundary, shows its fallback, and retries on resolution. **`try/catch` for loading.** Triggered by `lazy`, `use`, RSC and framework loaders — **not** a `useEffect` fetch. [L15] |
| 69 | How do you avoid a loading flash on navigation? | `startTransition` — React keeps already-revealed content visible instead of reverting to the fallback. **The most valuable Suspense interaction.** [L15] |
| 70 | What does a portal preserve, and escape? | Preserves React-tree position: context flows, events bubble through React. Escapes DOM position: `overflow: hidden`, `z-index` stacking, `transform` ancestors. **Portal clicks reach React ancestors *and* `document`** — which breaks naive click-outside handlers. [L15] |
| 71 | Why is the URL a state container? | Shareable, bookmarkable, back/forward, refresh-proof, quotable in a support ticket — **and it removes the re-render cascade** from a shared parent. Filters, sort, cursor, active tab, linkable modals. [L16] |
| 72 | What's the fetch waterfall? | Component-level fetching means navigate → render → mount → effect → fetch, repeated *serially* for nested routes. Loaders fetch all matched levels in parallel and render once — often 2–3× faster navigation. [L16] |
| 73 | How do you structure a React project? | By feature, with an `index.ts` public surface per feature, shared primitives in `components/ui`, patterns in `components/patterns`, infra in `lib`. **Dependencies flow downward only, enforced by a lint rule** — unenforced structure decays in a sprint. [L16] |
| 74 | Client state vs server state? | **Ownership.** Client state you own. Server state is borrowed: someone else owns it, it can change without you, what you hold is a cache. That difference generates the whole feature list of a query library. [L17] |
| 75 | Why is `useEffect` + `fetch` the wrong default? | Race conditions, no cancellation, no cache, no dedup, no revalidation, no retry, unchecked `res.ok`, no validation, 8-state loading soup, repeated in every component. **That list is a caching library.** [L17] |
| 76 | `staleTime` vs `gcTime`? | `staleTime` = how long data is fresh → controls **refetching**. `gcTime` = how long unused data stays in memory → controls **eviction**. A query can be stale but cached — shown instantly, refetched in the background. [L17] |
| 77 | How do you choose `staleTime`? | By tolerable staleness: money = 0 (and poll); lists = tens of seconds; **terminal resources = `Infinity`, because they're immutable**; reference data = hours. The terminal-state case is the answer that stands out. [L17] |
| 78 | Walk me through an optimistic update. | Five steps: cancel in-flight queries, snapshot, apply, roll back in `onError`, **reconcile in `onSettled` even on success** (your optimistic value was a guess). **Say what breaks without each.** [L18] |
| 79 | When would you NOT be optimistic? | When failure is unpredictable (card declined), when the server computes what you can't (fees, IDs, balances), and when rollback would alarm the user. *"I'd be optimistic about a toggle, not a refund."* [L18] |
| 80 | How do you add real-time to React? | **One connection at the root updating the query cache** — components read the cache and get updates for free, with no real-time code in any page. Plus: version guard for ordering, dedupe, bounded lists, invalidate on reconnect, jittered backoff. [L19] |

---

## Part 5 — Performance & server React (20)

| # | Question | Strong answer |
|---|---|---|
| 81 | "The app feels slow." Walk me through it. | Classify first: **load or render?** Then: highlight updates → commits per interaction → ranked view → "why did this render" → fix the cause → re-measure. **Naming a method beats naming a fix.** [L20] |
| 82 | What's wrong with profiling in development? | Dev React is 2–5× slower, StrictMode double-renders, and dev-only work isn't evenly distributed — so it can mislead about **which** component is slow, not just by how much. [L20] |
| 83 | 200 narrow bars in the flame graph — what does it mean? | Each row is cheap, collectively expensive. The fix is **rendering fewer rows** (virtualization), not faster rows (`memo`). Compare with one wide bar and no children: genuinely expensive work in that component. [L20] |
| 84 | Five commits for one interaction? | An effect chain — an effect sets state that triggers another effect. Each link is a render with a visible intermediate state. Collapse it. [L20] |
| 85 | How would you render a million rows? | **Don't lead with virtualization.** Lead with: *"why does the client have a million rows? The server filters, sorts and paginates."* Then virtualization for the ~100 you hold, cursor pagination, `memo`'d rows, `transform` positioning — **and `aria-rowcount` so screen readers hear the real total.** [L21] |
| 86 | Why is virtualization more effective than `memo`? | Different orders of magnitude: 10,000 components → 20 is a 500× reduction. `memo` skips a re-render of a component you're still creating an element for. **You can't memoize your way out of a 10,000-node tree.** [L21] |
| 87 | What's the downside of virtualization? | Ctrl+F doesn't find off-screen content, printing shows only the window, accessibility needs explicit ARIA counts, variable heights need measurement. **Not worth it under ~100 rows.** [L21] |
| 88 | Why does animating `left` stutter but `transform` doesn't? | `left` triggers layout and paint every frame; `transform` is compositor-only. Same reason virtualizers use `transform`. [L21] |
| 89 | What is layout thrashing? | Interleaving DOM reads (`offsetHeight`) with writes in a loop, forcing synchronous layout each iteration. Batch reads then writes. **Invisible in the React Profiler** — find it in the browser Performance panel. [L21] |
| 90 | The three Core Web Vitals? | LCP (main content, ≤2.5s), INP (interaction responsiveness, ≤200ms), CLS (visual stability, ≤0.1), at **p75 of real users**. **INP replaced FID in 2024** — FID measured only the first interaction's input delay. [L22] |
| 91 | How do you reduce bundle size? | Measure with an analyzer first. Then: route-level splitting, replace heavy deps (`Intl` over a date library), lazy-load below-the-fold and conditional features, check for duplicates, verify tree-shaking. [L22] |
| 92 | Why does lazy-loading the LCP image hurt? | `loading="lazy"` deprioritises it, so the element that defines your LCP loads last. A blanket lazy-load on all images includes the hero. Use `fetchPriority="high"` instead. [L22] |
| 93 | How can a loading state cause CLS? | A skeleton whose dimensions don't match the content it replaces shifts the layout when it swaps. **"Add a skeleton" can make vitals worse** if the sizes don't match. [L22] |
| 94 | What is a Server Component, mechanically? | Runs **once** on the server and ships only its **serialized output** — not HTML, not JavaScript. Its code never reaches the browser, so its dependencies aren't in the bundle, and it never re-renders. [L23] |
| 95 | RSC vs SSR? | SSR renders to HTML **and still ships the components** to hydrate. RSC ships only the output — the code never goes to the client. Different problems (first paint vs bundle size), and **complementary**: a Next.js app uses both. [L23] |
| 96 | How do you keep a Server Component inside a Client Component? | Pass it as `children`, never import it. An import makes it part of the client module graph; `children` passes already-rendered output. **The most important RSC pattern.** [L23] |
| 97 | Is `"use server"` a security boundary? | **No — the opposite.** It creates a public HTTP endpoint anyone can call with any arguments. Every action must authenticate, authorize and validate, with tenant scope from the session. It's the BOLA problem with new syntax. [L23] |
| 98 | Explain Next.js caching. | Four layers: request memoization (per-request dedup), Data Cache (persistent, per-fetch), Full Route Cache (rendered output), client Router Cache. **Then the debugging procedure:** does a hard refresh fix it? That splits client from server immediately. [L24] |
| 99 | What is hydration, and what causes a mismatch? | Attaching React to server HTML by **matching**, not rendering. Mismatch causes: non-deterministic values, browser APIs in render, **locale/timezone differences**, invalid HTML nesting, browser extensions. [L25] |
| 100 | Why is a hydration mismatch worse than a warning? | React recovers by re-rendering the subtree on the client — silently and expensively — and in the worst case the whole page falls back to client rendering. **You've paid for SSR and lost its benefit.** [L25] |

---

## Part 6 — Judgement (10)

| # | Question | What a strong answer contains |
|---|---|---|
| 101 | When is React the wrong choice? | Mostly-static content (Astro), hard-realtime rendering of thousands of independently-updating nodes (Solid's fine-grained reactivity), or a strict bundle budget. Right for rich interactive apps, large teams, ecosystem and hiring. **Willingness to say this is a positive signal.** |
| 102 | When would you not use Next.js? | An authenticated, highly-interactive dashboard — nothing to index, no anonymous first paint, nearly everything is a client component, so RSC is ceremony without the benefit. **Declining the popular choice with reasoning is the signal.** |
| 103 | How do you decide where state lives? | The decision tree, with the URL as a first-class option. Then the two considerations most people miss: **re-render scope** and **shareability**. |
| 104 | How much should you memoize? | Measure first; fix structurally; memoize last and verify. With the Compiler on, stop hand-memoizing. *"If I'm adding `useCallback` without a Profiler reading, I'm guessing."* |
| 105 | Your team wants to add a state library. What do you ask? | What state? Server data → a query library, not Redux. Frequently-changing shared client state → a store with selectors. Rarely-changing app-wide → context. **Most "we need Redux" is actually "we need a query cache."** |
| 106 | How do you review React code? | Order of consequence: unvalidated boundaries → illegal states representable → effect misuse (the six cases) → identity/keys → accessibility → then performance. **A missing `role="alert"` outranks a naming nit.** |
| 107 | What's the hardest React bug you've debugged? | Have a real one. The good shape: symptom → what you assumed → what the tools actually showed → the mechanism → the systemic fix. Identity/key bugs and race conditions make the best stories because the symptom looks impossible. |
| 108 | How do you onboard someone to a React codebase? | The architecture doc first (features, layers, dependency direction), pair on one full feature end-to-end, and point them at the domain types — branded IDs and discriminated unions teach the model faster than prose. |
| 109 | What's the most valuable React habit? | **Keep "highlight updates" on and watch what re-renders.** People who become fluent look at it constantly; people who stay stuck reason about it in their heads. |
| 110 | What does React not fix? | Bad domain models, unvalidated inputs, missing error handling, race conditions, accessibility, and a slow API. **It's a rendering library — it makes the UI a function of state, and says nothing about whether the state is right.** |

---

## The 15-question self-test (cold, timed)

From [the README](../README.md). Two minutes each, out loud, no notes.

1. What does JSX compile to, and why does that matter?
2. Why does React re-render a component? Name all the triggers.
3. Your input loses focus on every keystroke. Why?
4. Why is `key={index}` wrong, and when is it actually fine?
5. `setCount(count + 1)` three times — what's the result and why?
6. Why does your effect see a stale value?
7. Why does StrictMode run your effect twice, and what is it telling you?
8. When should you NOT use `useEffect`? Name four cases.
9. A context value changes — who re-renders, and how do you limit it?
10. `useMemo` didn't help. Give three reasons.
11. Search-as-you-type shows results for an old query. What happened?
12. Why is `useEffect` + `fetch` the wrong default for server data?
13. How do you render 4 million rows?
14. What is a Server Component, mechanically? What can't it do?
15. Your LCP is 4.2s. Walk me through the investigation.

**13–15 confident → ready. 10–12 → drill the gaps. Under 10 → re-read the modules; you're recognising, not recalling.**

---

## What's next

Rapid-fire tests recall. The other two React interview formats are **machine coding** (build an autocomplete, live) and **frontend system design** (design a dashboard for 10,000 concurrent users). Both are performance skills that need rehearsing.

Next → **[Lesson 30: Machine coding & design rounds](30-machine-coding-and-design.md)**
