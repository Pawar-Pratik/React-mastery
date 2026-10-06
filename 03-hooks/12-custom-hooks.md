# Lesson 12 — Custom hooks

> **Why this lesson exists:** custom hooks are React's unit of logic reuse, and they're also where codebases accumulate their worst abstractions — `useApi`, `useEverything`, hooks that take twelve options and return nineteen values. The mechanics take five minutes; the judgement about **what deserves to be a hook** is the actual skill, and it's the same API-design discipline you've already learned twice.

**Time:** ~55 minutes · **Prereq:** Module 3 so far

---

## 1. The idea in one sentence

> **A custom hook is just a function that calls hooks — so it shares *stateful logic*, never state itself, and every component calling it gets its own independent copy.**

---

## 2. The mechanics

```tsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(o => !o), []);
  const setTrue = useCallback(() => setOn(true), []);
  const setFalse = useCallback(() => setOn(false), []);
  return { on, toggle, setTrue, setFalse };
}
```

Two rules, and that's it:
1. **The name must start with `use`** — this is how the linter knows to apply the Rules of Hooks, and how React's tooling identifies it. Not a convention you can ignore.
2. **Rules of Hooks apply**: top level only, no conditionals, no loops.

**The property people misunderstand:**
```tsx
function A() { const { on } = useToggle(); }   // A's own state
function B() { const { on } = useToggle(); }   // B's own, completely independent state
```
**Custom hooks share *logic*, not *state*.** Two components calling `useToggle` are as independent as two components calling `useState`. If you want shared state, that's context ([Lesson 09](09-context.md)) or a store — a very common misconception, and a good interview question.

### Returns: tuple or object?
```tsx
// Tuple — for 1–2 values, so callers can rename
function useToggle(init = false) {
  const [on, setOn] = useState(init);
  return [on, useCallback(() => setOn(o => !o), [])] as const;   // ← as const makes it a tuple
}
const [isOpen, toggleOpen] = useToggle();
const [isDark, toggleDark] = useToggle();      // two in one component, renamed

// Object — for 3+, so callers don't memorise an order
function usePayment(id: PaymentId) {
  return { payment, isLoading, error, refetch };
}
const { payment, isLoading } = usePayment(id);   // destructure only what you need
```
**Without `as const` a tuple return infers as `(boolean | (() => void))[]`** and destructuring loses the types ([TS Lesson 17](../../TypeScript/05-ecosystem/17-typescript-with-react.md)).

---

## 3. What makes a *good* custom hook

This is the judgement half, and it's [API Lesson 01](../../API/01-foundations/01-what-an-api-really-is.md) again: **a hook is an API, so design the contract.**

### Rule 1 — One concern
```tsx
// ❌ Three unrelated jobs in one hook
function usePaymentsPage() {
  const [filters, setFilters] = useState(...);
  const { data } = useQuery(...);
  const [selected, setSelected] = useState(...);
  const [modalOpen, setModalOpen] = useState(...);
  useEffect(() => { document.title = ...; }, [data]);
  return { filters, setFilters, data, selected, setSelected, modalOpen, setModalOpen };
}
// Every consumer re-renders on every change. Nothing is reusable. It can't be tested in pieces.

// ✅ Small, composable hooks — and compose them if you want a convenience wrapper
const filters = useFilters();
const payments = usePayments(filters);
const selection = useSelection(payments.data);
```

### Rule 2 — Name it after *what it gives you*, not how it works
```tsx
useLocalStorage("theme", "light")       // ✅ a value, synced to storage
useDebouncedValue(query, 300)            // ✅ a debounced value
usePayment(id)                            // ✅ a payment
useFetchDataFromApiWithCache()            // ❌ implementation in the name
useUtils()                                 // ❌ means nothing
```

### Rule 3 — Hooks that return nothing are legitimate
```tsx
useDocumentTitle(`${count} payments · Ledger`);
useLockBodyScroll(isModalOpen);
useKeyboardShortcut("/", focusSearch);
```
Perfectly good — they encapsulate an effect and its cleanup. **This is often the best use of a custom hook**: hiding an effect with a fiddly cleanup so nobody has to get it right twice.

### Rule 4 — Don't wrap a single hook for no reason
```tsx
// ❌ An indirection that adds nothing
function useCount() { return useState(0); }

// ✅ Wrap when you're adding behaviour
function useCounter(initial = 0, { min = -Infinity, max = Infinity } = {}) {
  const [count, setCount] = useState(initial);
  const inc = useCallback(() => setCount(c => Math.min(max, c + 1)), [max]);
  const dec = useCallback(() => setCount(c => Math.max(min, c - 1)), [min]);
  return { count, inc, dec, reset: useCallback(() => setCount(initial), [initial]) };
}
```

### Rule 5 — Extract when there are two call sites, not one
The same rule as any abstraction. **One use case isn't enough information to design the right interface** — you'll guess at the generality and guess wrong. Wait for the second, then extract the shape they actually share.

> **The counter-argument worth acknowledging:** extract early when the logic is *tricky* rather than *repeated* — a race-condition-safe fetch, an effect with a subtle cleanup, a keyboard trap. There the value isn't reuse, it's **getting it right once**. That's a legitimate reason for a single-call-site hook, and saying so shows judgement rather than rule-following.

---

## 4. The hooks worth having

```tsx
// ── useDebouncedValue — the single most useful custom hook ──
export function useDebouncedValue<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(t);            // ← the cleanup IS the debounce
  }, [value, delayMs]);
  return debounced;
}
```
Note how small the correct implementation is: each new value cancels the previous timer via cleanup. **That's the synchronization model from [Lesson 08](../02-state-effects/08-effects.md) doing exactly what it's designed for.**

```tsx
// ── useLocalStorage — with the traps handled ──
export function useLocalStorage<T>(key: string, initial: T, schema?: ZodType<T>) {
  const [value, setValue] = useState<T>(() => {          // lazy init — reads storage ONCE
    try {
      const raw = localStorage.getItem(key);
      if (raw === null) return initial;
      const parsed = JSON.parse(raw);
      return schema ? schema.parse(parsed) : (parsed as T);   // ← validate! (TS Lesson 15)
    } catch { return initial; }                                // corrupt / quota / private mode
  });

  const set = useCallback((next: T | ((prev: T) => T)) => {
    setValue(prev => {
      const resolved = typeof next === "function" ? (next as (p: T) => T)(prev) : next;
      try { localStorage.setItem(key, JSON.stringify(resolved)); } catch { /* quota */ }
      return resolved;
    });
  }, [key]);

  return [value, set] as const;
}
```
Four traps handled: lazy initialisation (don't read storage every render), **schema validation** (last month's shape is today's crash), `try/catch` (private mode and quota errors throw), and the updater form.

```tsx
// ── useMediaQuery — on useSyncExternalStore, SSR-safe (Lesson 11) ──
export function useMediaQuery(query: string): boolean {
  const subscribe = useCallback((cb: () => void) => {
    const mq = matchMedia(query);
    mq.addEventListener("change", cb);
    return () => mq.removeEventListener("change", cb);
  }, [query]);
  return useSyncExternalStore(subscribe, () => matchMedia(query).matches, () => false);
}

// ── useEventCallback — a stable identity with fresh behaviour (Lesson 07) ──
export function useEventCallback<A extends unknown[], R>(fn: (...args: A) => R) {
  const ref = useRef(fn);
  useLayoutEffect(() => { ref.current = fn; });
  return useCallback((...args: A) => ref.current(...args), []);
}

// ── useKeyboardShortcut — an effect with a cleanup nobody should rewrite ──
export function useKeyboardShortcut(key: string, handler: () => void, opts: { meta?: boolean } = {}) {
  const stable = useEventCallback(handler);
  useEffect(() => {
    function onKey(e: KeyboardEvent) {
      if (e.key !== key) return;
      if (opts.meta && !(e.metaKey || e.ctrlKey)) return;
      const target = e.target as HTMLElement;
      if (target.tagName === "INPUT" || target.isContentEditable) return;  // ← don't hijack typing
      e.preventDefault();
      stable();
    }
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [key, opts.meta, stable]);
}
```
That `INPUT`/`contentEditable` guard is the detail that separates a working shortcut hook from one that breaks every form on the page — exactly the kind of thing worth encapsulating once.

```tsx
// ── usePrevious — genuinely useful, and often a smell ──
export function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```
> **The caveat:** `usePrevious` reads the value *after* paint, so it's a render behind. For "did this prop change?" logic, **compare during render** with the previous-state pattern from [Lesson 05](../02-state-effects/05-state-and-snapshots.md) instead — it's synchronous and doesn't cause an extra frame.

---

## 5. Testing custom hooks

```tsx
import { renderHook, act } from "@testing-library/react";

test("useToggle flips", () => {
  const { result } = renderHook(() => useToggle(false));
  expect(result.current.on).toBe(false);
  act(() => result.current.toggle());
  expect(result.current.on).toBe(true);
});

test("useDebouncedValue delays", () => {
  vi.useFakeTimers();
  const { result, rerender } = renderHook(({ v }) => useDebouncedValue(v, 300), {
    initialProps: { v: "a" },
  });
  rerender({ v: "b" });
  expect(result.current).toBe("a");                 // not yet
  act(() => { vi.advanceTimersByTime(300); });
  expect(result.current).toBe("b");
});
```

**But the honest guidance:** prefer testing the *component* that uses the hook. A hook is an implementation detail; testing it directly couples your tests to that detail, and `renderHook` tests pass while the component is broken ([Lesson 28](../08-project/28-testing-react.md)).

**Test the hook directly when** it's genuinely reusable library-grade code with many consumers, or the logic is complex enough (debounce timing, retry backoff) that component tests would be convoluted.

---

## 6. Anti-patterns

### Anti-pattern 1 — The god hook
```tsx
// ❌ Returns 19 values; every consumer re-renders on every change
const { user, payments, filters, setFilters, modal, setModal, selected, ... } = usePaymentsPage();
```
Split by concern. If a "page hook" is convenient, compose small hooks inside it rather than writing one large one — and be aware that anything it returns is a re-render trigger for every consumer.

### Anti-pattern 2 — Expecting shared state
```tsx
// ❌ These are NOT the same counter
function A() { const { count } = useCounter(); }
function B() { const { count } = useCounter(); }
```
Custom hooks share logic, not state. For shared state: lift it, context, or a store.

### Anti-pattern 3 — A hook that just wraps `useState`
```tsx
function useName() { return useState(""); }      // ❌ indirection with no behaviour
```

### Anti-pattern 4 — Conditional hook calls inside a custom hook
```tsx
function useMaybeData(enabled: boolean) {
  if (!enabled) return null;                      // ❌ early return before hooks
  const { data } = useQuery(...);
  return data;
}
// ✅ Pass the condition down; most libraries support it
function useMaybeData(enabled: boolean) {
  const { data } = useQuery({ ..., enabled });
  return enabled ? data : null;
}
```

### Anti-pattern 5 — Hiding an important effect
```tsx
// ❌ Nothing in the name says this subscribes to a socket and reconnects
function usePaymentData(id: PaymentId) { /* opens an SSE connection */ }

// ✅ Name what it does
function usePaymentLiveUpdates(id: PaymentId) { ... }
```
A hook that silently creates a connection, registers a global listener, or writes to storage should **say so in its name.** Surprising side effects behind an innocent name is how a codebase becomes hard to reason about.

### Anti-pattern 6 — Returning unstable references
```tsx
// ❌ New object every render → breaks every consumer's memo and effect deps
function useFilters() {
  const [status, setStatus] = useState(...);
  return { filters: { status }, setStatus };       // `filters` is fresh each render
}
// ✅
const filters = useMemo(() => ({ status }), [status]);
```
**A hook's return value is part of its contract** — including referential stability. Returning a fresh object each render means every consumer's `useEffect([filters])` fires forever.

---

## 7. Ledger Console's hooks

```tsx
// ── Domain hooks — thin wrappers over the query layer (Lesson 17) ──
usePayments(filters)          → Async<Paginated<Payment>>
usePayment(id)                 → Async<Payment>
useRefundPayment()             → mutation with optimistic update (Lesson 18)
useBalance()                   → Async<Balance>

// ── UI hooks ──
useDebouncedValue(value, ms)
useFilters()                   → reads/writes the URL, returns { filters, setFilter, reset }
useSelection(items)            → { selected, toggle, selectAll, clear, isSelected }
useKeyboardShortcut(key, fn)
useDocumentTitle(title)
useLockBodyScroll(locked)
useMediaQuery(query)
useIdempotencyKey(resetOn)     → stable across retries (Lesson 07)

// ── Composed convenience ──
function usePaymentsTable() {
  const { filters, setFilter } = useFilters();
  const deferred = useDeferredValue(filters);
  const query = usePayments(deferred);
  const selection = useSelection(query.data?.items ?? []);
  return { filters, setFilter, query, selection, isStale: filters !== deferred };
}
```
**The convenience hook composes the small ones** rather than reimplementing them — so each piece is still independently usable and testable, and `usePaymentsTable` is 6 lines rather than 60.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Name starts with `use`** | The linter's Rules-of-Hooks detection depends on it |
| **One concern per hook** | A god hook re-renders every consumer on every change |
| **Name after what it gives you, not how it works** | `useDebouncedValue`, not `useTimeoutStateThing` |
| **Extract at two call sites — or at one if the logic is *tricky*** | Reuse isn't the only value; getting it right once is |
| **Tuple for 1–2 returns (with `as const`), object for 3+** | Renameable vs order-independent |
| **Return stable references** | A fresh object breaks every consumer's memo and effect deps |
| **Hooks share logic, never state** | For shared state: lift, context, or a store |
| **Never call hooks conditionally, even inside a custom hook** | Positional storage on the fiber |
| **Name hooks with side effects honestly** | A hook that opens a socket should say so |
| **Validate anything read from storage or the URL** | It was written by an older version of your code |
| **Prefer testing the component over the hook** | A hook is an implementation detail |

---

## 9. Interview traps

**Q1. "What's a custom hook?"**
A function that calls hooks, used to share *stateful logic*. **Then the key property:** each caller gets an independent copy — hooks share logic, not state. Two components calling `useCounter` have two separate counters.

**Q2. "How do you share state between components with a hook?"**
You don't — that's not what hooks do. Lift the state, use context, or use an external store. The hook can *wrap* the context (`useAuth`), but the sharing comes from the context, not the hook.

**Q3. "Why must a custom hook start with `use`?"**
It's how the ESLint plugin knows to enforce the Rules of Hooks inside it, and how React's tooling identifies hooks. Without the prefix, a function calling `useState` conditionally wouldn't be flagged.

**Q4. "When do you extract a custom hook?"**
Two call sites with the same logic — or one call site where the logic is tricky enough to be worth getting right once (a race-safe fetch, an effect with a subtle cleanup, a focus trap). **Naming the second case shows judgement rather than rule-following.**

**Q5. "Tuple or object return?"**
Tuple for one or two values, so callers can rename (`const [isOpen, toggleOpen]`) — with `as const`, or the types collapse into a union array. Object for three or more, so callers don't memorise an order and can destructure only what they need.

**Q6. "Write `useDebouncedValue`."**
Six lines: state initialised to the value, an effect with `setTimeout` and a `clearTimeout` cleanup, deps `[value, delay]`. **The insight to volunteer:** the cleanup *is* the debounce — each new value cancels the pending timer, which is the synchronization model doing exactly its job.

**Q7. "What's wrong with a hook that returns 19 values?"**
Every consumer re-renders whenever any of them changes, nothing is independently reusable or testable, and the hook has no single reason to change. Split by concern and compose.

**Q8. "How do you test a custom hook?"**
`renderHook` + `act` from Testing Library, with fake timers for anything time-based. **But prefer testing the component** — a hook is an implementation detail, and hook tests can pass while the component is broken. Test hooks directly for genuinely reusable library code or complex timing logic.

**Q9. "Why can't you call hooks conditionally, even inside a custom hook?"**
Hooks are stored positionally on the fiber and matched by call order. Skipping one shifts every subsequent hook's identity, so `useState` #2 reads #3's slot. Pass the condition into the hook (`enabled: false`) instead of returning early.

**Q10. "A hook returns `{ filters: { status } }` and a consumer's effect loops forever. Why?"**
The returned object is created fresh each render, so `useEffect([filters])` sees a new reference every time. Memoize the return value — **a hook's referential stability is part of its contract.**

---

## 10. Build & break

### Build — Ledger's hook library
Implement, with tests: `useDebouncedValue`, `useLocalStorage` (with schema validation), `useMediaQuery` (on `useSyncExternalStore`), `useEventCallback`, `useKeyboardShortcut` (with the input guard), `useDocumentTitle`, `useLockBodyScroll` (restoring the previous value), `useSelection`, `useIdempotencyKey`.

For each, write down: what it returns, whether the return is referentially stable, and what its cleanup does.

### Build — compose, don't accumulate
Write `usePaymentsTable` by composing `useFilters`, `useDeferredValue`, `usePayments` and `useSelection`. Then write the god-hook version that does all of it inline. Compare: line count, testability, and how many consumers re-render when the selection changes.

### Break — five experiments
1. **Shared state misconception.** Call `useCounter()` in two components and expect them to sync. They don't. Then lift it to context and watch them sync.
2. **Missing `as const`.** Return `[value, setter]` without it, destructure, and try to call the setter. Read the type error.
3. **Unstable return.** Return `{ filters: { status } }` unmemoized and put `filters` in a consumer's effect deps. Watch the infinite loop.
4. **Conditional hook.** Early-return before a `useQuery` inside a custom hook. Read the Rules-of-Hooks error, then watch the runtime corruption if you suppress it.
5. **Eager `localStorage`.** Write `useLocalStorage` without the lazy initialiser and log inside. Count the reads per render.

### Explain out loud (60 seconds)
1. What a custom hook shares, and what it doesn't.
2. Why the `use` prefix is required.
3. When to extract — including the non-reuse reason.
4. Tuple vs object returns, and the `as const` trap.
5. Why a hook's return must be referentially stable.

---

## Module 3 complete — checkpoint

- [ ] What context is for, and the one thing it lacks
- [ ] The two distinct causes of consumer re-renders
- [ ] Split by change frequency; split state from dispatch
- [ ] Why `memo` can't stop a context re-render
- [ ] The two purposes of memoization
- [ ] Three reasons `memo` silently fails
- [ ] Four cases where memoization is correctness, not performance
- [ ] The four-step optimisation order
- [ ] What the React Compiler does and doesn't fix
- [ ] What concurrent rendering is — and what it doesn't speed up
- [ ] Transition vs debounce
- [ ] `useTransition` vs `useDeferredValue`, and the `memo` requirement
- [ ] Tearing, and `useSyncExternalStore`
- [ ] Why `useId` exists
- [ ] What custom hooks share (logic, not state)
- [ ] When to extract, and the tricky-logic exception

---

## What's next

Module 4 is composition and patterns: how to structure components that other people use — compound components, slots, render props — plus forms, error boundaries and Suspense, and the app architecture that survives 200 components.

Next → **[Lesson 13: Component design & composition](../04-composition/13-composition-patterns.md)**
