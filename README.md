# React: Zero → Absolute Authority

> The goal is not "I can build components."
> The goal is: **you know exactly why the UI re-rendered, why that effect ran twice, and why the input lost focus** — and you can fix each at the right layer instead of guessing.

---

## Why most React knowledge is shallow

React is easy to start and hard to be right about. Most engineers learn it as a bag of APIs — `useState`, `useEffect`, `useMemo` — and build a mental model from what happens to work. Then:

- *"Why does my input lose focus every keystroke?"* (component identity, not a bug)
- *"Why does my effect run twice?"* (StrictMode telling you it isn't idempotent)
- *"Why is my state one render behind?"* (closures over a snapshot)
- *"Why didn't `useMemo` help?"* (you memoized the wrong thing)
- *"Why does the whole page re-render when one row changes?"* (context, or an unstable object)
- *"Why does my search show results for a query I already replaced?"* (a race condition nobody handled)

Every one has a precise, mechanical answer. This course teaches the mechanisms, so React stops surprising you.

### The five ideas everything else derives from

1. **UI is a function of state.** `view = f(state)`. You describe *what*, never *how to get there*.
2. **A render is just a function call.** It produces a description of UI, not DOM. Rendering and committing are separate phases.
3. **A component's identity comes from its position in the tree** (and its `key`). Move it, and React destroys and recreates it — state and all.
4. **Every render is a snapshot.** Props, state and handlers from that render are frozen in that render's closures. This explains every "stale value" bug.
5. **Effects are synchronization, not lifecycle.** They exist to sync React state with something *outside* React. Most `useEffect`s shouldn't exist.

Internalise those five and React becomes predictable.

---

## What you'll be able to do at the end

- Explain, precisely, why any component re-rendered — and fix it at the right layer
- Write effects that are correct under StrictMode, cleanup, race conditions and cancellation
- Choose the right state location: local, lifted, context, URL, server cache, or an external store
- Build a table of 4 million rows that scrolls at 60fps
- Ship optimistic updates with correct rollback
- Explain RSC, Server Actions and Next.js caching without hand-waving
- Measure and fix Core Web Vitals, not guess at them
- Answer *"build an autocomplete"* in a machine-coding round with debouncing, cancellation, keyboard nav and a11y

---

## The project: **Ledger Console**

The merchant dashboard for the [Ledger API](../API/README.md), typed with the [TypeScript track's](../TypeScript/README.md) shared layer.

- A payments table over **4M rows** — virtualized, filterable, cursor-paginated, URL-synced
- A **live payment feed** over SSE
- A **refund flow** with optimistic updates and rollback
- API key management, webhook delivery log with retry
- Revenue charts, auth with token refresh mid-request, full keyboard and screen-reader support

**Why this and not a to-do app:** every React concept appears for a real reason — virtualization because 4M rows exist, race-condition handling because search-as-you-type exists, optimistic updates because refunds feel slow, Suspense because the dashboard has independent panels. You can't fake your way through it, and you end with something worth demoing.

Blueprint: **[Ledger Console blueprint](08-project/26-console-blueprint.md)**

---

## Setup

| Tool | Notes |
|---|---|
| **React 19** | Actions, `use`, `useOptimistic`, ref-as-prop, and the React Compiler are all assumed |
| **Vite** | `npm create vite@latest -- --template react-ts` |
| **TypeScript 5.x, strict** | Non-negotiable. The TS track is a prerequisite |
| **React DevTools** | **Turn on "Highlight updates when components render"** on day one. It is the single best learning tool here |
| **TanStack Query** | Module 5 |
| **Next.js 15** | Module 7 only |

> **The habit that matters most:** keep the DevTools Profiler open and *look* at what re-rendered. People who become fluent in React are people who watch the render highlights constantly; people who stay stuck reason about it in their heads.

---

## Table of contents

### Module 1 — The mental model
> Everyone who finds React unpredictable is missing something here. Go slowly.

| # | Lesson | What you'll be able to do |
|---|---|---|
| 01 | [How React actually works](01-mental-model/01-how-react-works.md) | Explain the element tree, reconciliation and why React exists at all |
| 02 | [JSX, components & props](01-mental-model/02-jsx-components-props.md) | Design a component API; know exactly what JSX compiles to |
| 03 | [Rendering: render vs commit](01-mental-model/03-rendering-and-commit.md) | Answer "why did this re-render?" from first principles |
| 04 | [Keys, identity & lists](01-mental-model/04-keys-and-identity.md) | Explain the lost-input-focus bug and never write index keys again |

### Module 2 — State & effects
| # | Lesson | What you'll be able to do |
|---|---|---|
| 05 | [State, snapshots & batching](02-state-effects/05-state-and-snapshots.md) | Explain every stale-state bug; structure state correctly |
| 06 | [useReducer & state machines](02-state-effects/06-reducers-and-state-machines.md) | Model complex UI state so illegal states can't exist |
| 07 | [Refs & escape hatches](02-state-effects/07-refs-and-escape-hatches.md) | Know when to leave React's model, and how to do it safely |
| 08 | [Effects: synchronization, not lifecycle](02-state-effects/08-effects.md) | **The most important lesson.** Write correct effects; delete most of them |

### Module 3 — Hooks in full
| # | Lesson | What you'll be able to do |
|---|---|---|
| 09 | [Context, properly](03-hooks/09-context.md) | Use context without re-rendering the world |
| 10 | [memo, useMemo, useCallback](03-hooks/10-memoization.md) | Memoize the right thing; know when React Compiler makes it moot |
| 11 | [The concurrent hooks](03-hooks/11-concurrent-hooks.md) | `useTransition`, `useDeferredValue`, `useSyncExternalStore`, `useId` |
| 12 | [Custom hooks](03-hooks/12-custom-hooks.md) | Extract logic that composes; avoid the abstractions that don't |

### Module 4 — Composition & patterns
| # | Lesson | What you'll be able to do |
|---|---|---|
| 13 | [Component design & composition](04-composition/13-composition-patterns.md) | Compound components, slots, render props — and when each is right |
| 14 | [Forms, properly](04-composition/14-forms.md) | Controlled vs uncontrolled, validation, Actions, large forms |
| 15 | [Error boundaries, Suspense & portals](04-composition/15-boundaries-suspense-portals.md) | Fail gracefully; render outside the tree; suspend correctly |
| 16 | [Routing & app architecture](04-composition/16-routing-and-architecture.md) | Structure an app that survives 200 components |

### Module 5 — Data
| # | Lesson | What you'll be able to do |
|---|---|---|
| 17 | [Server state vs client state](05-data/17-server-state.md) | Stop using `useEffect` to fetch; cache correctly |
| 18 | [Mutations, optimistic updates & races](05-data/18-mutations-and-optimistic.md) | Ship optimistic UI with correct rollback |
| 19 | [Real-time in React](05-data/19-realtime.md) | SSE and WebSocket integrated with the cache, not fighting it |

### Module 6 — Performance
| # | Lesson | What you'll be able to do |
|---|---|---|
| 20 | [Why did it render? Measuring first](06-performance/20-measuring.md) | Profile before optimising; read a flame graph |
| 21 | [Rendering performance](06-performance/21-render-performance.md) | Virtualization, memoization, the Compiler, 60fps at 4M rows |
| 22 | [Loading performance & Core Web Vitals](06-performance/22-loading-performance.md) | Code splitting, bundles, LCP/INP/CLS |

### Module 7 — Server React
| # | Lesson | What you'll be able to do |
|---|---|---|
| 23 | [React Server Components](07-server-react/23-rsc.md) | Explain RSC mechanically — the question everyone fumbles |
| 24 | [Next.js App Router](07-server-react/24-nextjs-app-router.md) | Routing, caching layers, Server Actions, without hand-waving |
| 25 | [SSR, streaming & hydration](07-server-react/25-ssr-streaming-hydration.md) | Diagnose a hydration mismatch; choose a rendering strategy |

### Module 8 — Project & scale
| # | Lesson | What you'll be able to do |
|---|---|---|
| 26 | [Ledger Console blueprint](08-project/26-console-blueprint.md) | The full system: routes, state model, architecture, decisions |
| 27 | [Console build log](08-project/27-console-build-log.md) | Build it in 8 phases, each mapped to lessons |
| 28 | [Testing React](08-project/28-testing-react.md) | Test behaviour, not implementation; RTL, MSW, Playwright |

### Module 9 — Interview mastery
| # | Lesson | What you'll be able to do |
|---|---|---|
| 29 | [Rapid-fire master Q&A](09-interview/29-rapid-fire-master-qa.md) | Revise the track; drill until answers are reflexes |
| 30 | [Machine coding & design rounds](09-interview/30-machine-coding-and-design.md) | Build an autocomplete live; design a frontend system |

---

## The 15 questions that define mastery

Self-test now, note your score, retest at the end.

1. What does JSX compile to, and why does that matter?
2. Why does React re-render a component? Name all the triggers.
3. Your input loses focus on every keystroke. Why?
4. Why is `key={index}` wrong, and when is it actually fine?
5. `setCount(count + 1)` three times in a row — what's the result and why?
6. Why does your effect see a stale value?
7. Why does React 19 StrictMode run your effect twice, and what is it telling you?
8. When should you NOT use `useEffect`? Name four cases.
9. A context value changes — who re-renders, and how do you limit it?
10. `useMemo` didn't help. Give three reasons.
11. Search-as-you-type shows results for an old query. What happened and how do you fix it?
12. Why is `useEffect` + `fetch` the wrong default for server data?
13. How do you render 4 million rows?
14. What is a Server Component, mechanically? What can't it do?
15. Your LCP is 4.2s. Walk me through the investigation.

Every one is answered in the track, with the lesson named in [Lesson 29](09-interview/29-rapid-fire-master-qa.md).

---

## Pacing

| Intensity | Duration |
|---|---|
| **Sprint** (4 h/day) | ~3 weeks |
| **Serious** (2 h/day) | ~6 weeks — the sweet spot |
| **Steady** (1 h/day) | ~12 weeks |

Spend the most time on **Modules 1–2**. Rendering, identity, snapshots and effects are where every later problem comes from — and they're the four things most React developers never learn properly.

---

## One promise

By the end, React will have no magic in it. Every behaviour will be something you can explain mechanically, which means every bug becomes a diagnosis rather than a guess.

Start here → **[Lesson 01: How React actually works](01-mental-model/01-how-react-works.md)**
