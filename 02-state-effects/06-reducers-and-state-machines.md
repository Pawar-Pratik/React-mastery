# Lesson 06 — useReducer & state machines

> **Why this lesson exists:** `useState` is fine for independent values and becomes a liability the moment state has *rules*. Most React codebases express those rules as scattered `if` statements across a dozen handlers, and the bugs that result — a spinner that never stops, a form you can submit twice, a modal showing stale data — are all the same bug: **state that permits combinations nobody intended.** A reducer plus a discriminated union makes those combinations unrepresentable.

**Time:** ~60 minutes · **Prereq:** Lesson 05 · [TS Lesson 09](../../TypeScript/03-type-level/09-illegal-states-unrepresentable.md)

---

## 1. The idea in one sentence

> **A reducer moves state logic *out* of your handlers and into one pure function where all the transitions live together — which is the only place you can see, test, and constrain them.**

---

## 2. When `useState` stops being enough

Four signals. If you hit two, switch.

```tsx
// Signal 1 — multiple states that must change together
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<ApiError | null>(null);
const [data, setData] = useState<Payment[] | null>(null);
// Every handler must remember to set all three. Forget one and you get a stuck spinner.

// Signal 2 — the next state depends on several current values
function handleNext() {
  if (step === "amount" && amount > 0 && !isSubmitting) setStep("confirm");
  else if (step === "confirm" && agreed) setStep("submitting");
  // ...transition rules scattered across handlers
}

// Signal 3 — the same update happens from several places
// setIsLoading(false); setError(e); appears in 6 handlers, slightly differently each time

// Signal 4 — you want to test the logic without rendering
```

A reducer fixes all four: one pure function, all transitions in one place, testable in isolation, and dispatch sites that say *what happened* rather than *what to set*.

---

## 3. The mechanics

```tsx
type State =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; payments: Payment[]; fetchedAt: number }
  | { status: "error"; error: ApiError };

type Action =
  | { type: "fetch_started" }
  | { type: "fetch_succeeded"; payments: Payment[] }
  | { type: "fetch_failed"; error: ApiError }
  | { type: "reset" };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "fetch_started":   return { status: "loading" };
    case "fetch_succeeded": return { status: "success", payments: action.payments, fetchedAt: Date.now() };
    case "fetch_failed":    return { status: "error", error: action.error };
    case "reset":           return { status: "idle" };
    default:                return assertNever(action);      // ← exhaustive (TS L07)
  }
}

const [state, dispatch] = useReducer(reducer, { status: "idle" });
```

Three properties that matter:

**1. The reducer is pure.** Same `(state, action)` → same result, always. No fetching, no `Date.now()` in a way that matters for tests, no mutation. That's what makes it testable and what makes React safe to call it twice in StrictMode.

**2. `dispatch` is referentially stable.** React guarantees it never changes, so it's safe in dependency arrays and in `memo`'d children with **no `useCallback`**. That's an underrated ergonomic win over passing several `setX` functions down.

**3. Actions describe *events*, not *setters*.**
```tsx
// ❌ setter-style actions — you've reinvented useState with extra steps
dispatch({ type: "set_loading", value: true });
dispatch({ type: "set_error", value: e });

// ✅ event-style actions — one dispatch, all consequences in one place
dispatch({ type: "fetch_failed", error: e });
```
Event-style is the whole point: the *component* says what happened, the *reducer* decides what that means. When the meaning changes, you edit one function instead of six handlers.

---

## 4. State machines: making transitions explicit

A reducer is already halfway to a state machine. Going the rest of the way means **the reducer only allows legal transitions.**

```tsx
// Ledger's refund flow
type RefundState =
  | { step: "closed" }
  | { step: "amount"; payment: Refundable; amountMinor: number }
  | { step: "confirm"; payment: Refundable; amountMinor: number; reason: RefundReason }
  | { step: "submitting"; payment: Refundable; amountMinor: number; reason: RefundReason }
  | { step: "done"; refund: Refund }
  | { step: "failed"; payment: Refundable; amountMinor: number; error: ApiError };

type RefundAction =
  | { type: "open"; payment: Refundable }
  | { type: "amount_changed"; amountMinor: number }
  | { type: "proceed"; reason: RefundReason }
  | { type: "back" }
  | { type: "submit" }
  | { type: "succeeded"; refund: Refund }
  | { type: "failed"; error: ApiError }
  | { type: "close" };

function refundReducer(state: RefundState, action: RefundAction): RefundState {
  // Global transitions first
  if (action.type === "close") return { step: "closed" };

  switch (state.step) {
    case "closed":
      return action.type === "open"
        ? { step: "amount", payment: action.payment, amountMinor: action.payment.money.amountMinor }
        : state;                                    // ← any other action is IGNORED, not an error

    case "amount":
      if (action.type === "amount_changed") return { ...state, amountMinor: action.amountMinor };
      if (action.type === "proceed")        return { ...state, step: "confirm", reason: action.reason };
      return state;

    case "confirm":
      if (action.type === "back")   return { step: "amount", payment: state.payment, amountMinor: state.amountMinor };
      if (action.type === "submit") return { ...state, step: "submitting" };
      return state;

    case "submitting":
      // ★ No "submit" case here — double-submit is IMPOSSIBLE, not merely guarded against
      if (action.type === "succeeded") return { step: "done", refund: action.refund };
      if (action.type === "failed")    return { step: "failed", payment: state.payment,
                                                amountMinor: state.amountMinor, error: action.error };
      return state;

    case "done":
    case "failed":
      return state;

    default:
      return assertNever(state);
  }
}
```

**Switch on the *state* first, then the action.** That's the structural difference between a reducer and a state machine, and it buys you:

| Property | How |
|---|---|
| **Double-submit is impossible** | `submitting` has no `submit` case. Not a disabled button — a structural impossibility |
| **Every state carries exactly its data** | No `refund?: Refund` that's undefined in five of six states |
| **Unknown actions are ignored, not crashes** | A late response after `close` returns `state` unchanged |
| **Adding a step is a compile error until handled** | `assertNever` on both the state and action unions |
| **You can draw it** | The reducer *is* the diagram |

> **Say this in an interview:** *"I switch on state first, then action — so an action that's illegal in the current state simply doesn't have a case, which means double-submit isn't something I guard against with a disabled button, it's something that can't happen. A disabled button is a UI affordance; the reducer is the invariant."*

And note the connection to the API track: [API Lesson 09](../../API/02-rest-design/09-writes-patch-and-bulk.md) puts the same guard in SQL (`WHERE status = 'requires_capture'`). **Same invariant, enforced at both ends, neither trusting the other.**

### Rendering a state machine
```tsx
function RefundModal({ payment }: { payment: Refundable }) {
  const [state, dispatch] = useReducer(refundReducer, { step: "closed" });

  switch (state.step) {
    case "closed":     return null;
    case "amount":     return <AmountStep amount={state.amountMinor} max={state.payment.money.amountMinor}
                                          onChange={a => dispatch({ type: "amount_changed", amountMinor: a })}
                                          onNext={r => dispatch({ type: "proceed", reason: r })} />;
    case "confirm":    return <ConfirmStep {...state} onBack={() => dispatch({ type: "back" })}
                                          onSubmit={() => dispatch({ type: "submit" })} />;
    case "submitting": return <ConfirmStep {...state} submitting />;   // ← no onSubmit; it can't fire
    case "done":       return <SuccessStep refund={state.refund} />;
    case "failed":     return <ErrorStep error={state.error} onRetry={() => dispatch({ type: "submit" })} />;
    default:           return assertNever(state);
  }
}
```
Every branch has exactly the data it needs, guaranteed by the type. No `state.refund!`, no optional chaining, no defensive checks.

---

## 5. Reducers and side effects

**A reducer must be pure**, so where does the `fetch` go?

```tsx
// ✅ The standard shape: dispatch around the async work
async function submit() {
  dispatch({ type: "submit" });                       // → submitting
  const result = await api.createRefund(payment.id, { amountMinor, reason }, idempotencyKey);
  if (result.ok) dispatch({ type: "succeeded", refund: result.value });
  else           dispatch({ type: "failed", error: result.error });
}
```

The reducer never knows about `fetch`. It just receives events. That separation is what makes it testable:

```tsx
test("cannot submit twice", () => {
  let s: RefundState = { step: "closed" };
  s = refundReducer(s, { type: "open", payment });
  s = refundReducer(s, { type: "proceed", reason: "requested_by_customer" });
  s = refundReducer(s, { type: "submit" });
  expect(s.step).toBe("submitting");
  s = refundReducer(s, { type: "submit" });           // second submit
  expect(s.step).toBe("submitting");                  // ← unchanged. No refund duplicated.
});
```
**No React, no rendering, no mocking — just a function.** That's the second-biggest reason to use a reducer, after the invariants.

### When the effects get complex — XState
```tsx
// A real state machine library gives you: entry/exit actions, guards, delays,
// parallel states, nested states, invoked promises, and a visualiser.
```
**When it's worth it:** genuinely complex flows (multi-step checkout, media players, onboarding wizards with branching, anything with timeouts and retries baked into the states). **When it isn't:** a four-state modal. `useReducer` covers most cases; know XState exists and can name when you'd reach for it.

---

## 6. Reducer patterns

```tsx
// Lazy initialisation — the third argument
const [state, dispatch] = useReducer(reducer, initialArg, init);
// `init(initialArg)` runs once. Useful for reading localStorage or computing from props.

function init(payment: Payment): RefundState {
  return { step: "amount", payment, amountMinor: payment.money.amountMinor };
}
const [state, dispatch] = useReducer(refundReducer, payment, init);
```

```tsx
// Passing dispatch down — no useCallback needed, dispatch is stable
const RefundContext = createContext<React.Dispatch<RefundAction> | null>(null);
// Children dispatch without the parent re-rendering them (Lesson 09's split-context pattern)
```

```tsx
// Action creators — optional, but they give you one place to build payloads
const actions = {
  open: (payment: Refundable) => ({ type: "open" as const, payment }),
  submit: () => ({ type: "submit" as const }),
};
dispatch(actions.open(payment));
```

```tsx
// Undo/redo — nearly free with a reducer
type Undoable<T> = { past: T[]; present: T; future: T[] };

function undoable<T, A>(reducer: (s: T, a: A) => T) {
  return (state: Undoable<T>, action: A | { type: "undo" } | { type: "redo" }): Undoable<T> => {
    if ((action as any).type === "undo") {
      const previous = state.past.at(-1);
      if (!previous) return state;
      return { past: state.past.slice(0, -1), present: previous, future: [state.present, ...state.future] };
    }
    if ((action as any).type === "redo") {
      const next = state.future[0];
      if (!next) return state;
      return { past: [...state.past, state.present], present: next, future: state.future.slice(1) };
    }
    const present = reducer(state.present, action as A);
    return present === state.present ? state : { past: [...state.past, state.present], present, future: [] };
  };
}
```
**Undo/redo as a reducer wrapper is the classic demonstration** of why reducers compose and `useState` doesn't. It's also a good machine-coding round answer ([Lesson 30](../09-interview/30-machine-coding-and-design.md)).

---

## 7. `useReducer` vs `useState` vs a store

| | `useState` | `useReducer` | External store (Zustand/Redux) |
|---|---|---|---|
| Independent values | ✅ | overkill | overkill |
| Related values, rules | awkward | ✅ | ✅ |
| Logic testable without React | ❌ | ✅ | ✅ |
| Shared across distant components | lift/context | lift/context | ✅ |
| Stable setter reference | `setX` is stable too | `dispatch` is stable | ✅ |
| Devtools time-travel | ❌ | ❌ (Redux has it) | ✅ (Redux) |
| Survives unmount | ❌ | ❌ | ✅ |

**Note both `setX` and `dispatch` are referentially stable** — that's a common myth to correct. The reducer's advantage isn't stability, it's *centralised transitions*.

**When to reach for a store:** state shared by many distant components that changes often, state that must survive a component unmounting, or when you need selector-based subscriptions to avoid re-render cascades ([Lesson 09](../03-hooks/09-context.md)). **Not** for server data — that's a query cache ([Lesson 17](../05-data/17-server-state.md)).

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Switch to a reducer when 2+ states change together, or transitions have rules** | Scattered `if`s across handlers is where the bugs are |
| **Actions describe events, not setters** | The component says what happened; the reducer decides what it means |
| **Switch on state first, then action** | Illegal transitions become structurally impossible, not guarded |
| **State is a discriminated union; each variant carries exactly its data** | No `refund?: Refund` undefined in five of six states |
| **`assertNever` on both the state and action unions** | Adding a variant is a compile error until handled |
| **Reducers stay pure — no fetch, no randomness, no mutation** | Testable, and safe under StrictMode double-invocation |
| **Unknown actions return `state` unchanged, never throw** | Late async responses shouldn't crash the UI |
| **`dispatch` is stable — don't wrap it in `useCallback`** | React guarantees it |
| **Test the reducer as a plain function** | No React, no mocks, no rendering |
| **Reach for XState only for genuinely complex flows** | `useReducer` covers most cases |

---

## 9. Interview traps

**Q1. "`useState` or `useReducer`?"**
The four signals: multiple states changing together, transitions depending on several current values, the same update from many places, or wanting to test logic without rendering. **Then the real argument:** a reducer puts all transitions in one pure function, which is the only place you can constrain them.

**Q2. "How do you prevent a double submit?"**
Weak: disable the button. Strong: *"a disabled button is a UI affordance — someone can still fire the handler via keyboard, a race, or a second code path. I'd model it as a state machine where the `submitting` state has no `submit` transition, so a second dispatch is a no-op by construction. And the server needs an idempotency key regardless, because the network can duplicate the request even if my UI can't."* **That last clause connects front-end and back-end thinking and is a strong signal.**

**Q3. "Why must a reducer be pure?"**
React may call it twice in StrictMode to surface impurity, and purity is what makes the state predictable and the logic testable without React. Side effects live in the component or an effect, around the dispatch.

**Q4. "Where do you put the API call?"**
Outside the reducer: `dispatch("submit")` → `await api.x()` → `dispatch("succeeded"|"failed")`. The reducer only receives events. If the flow is complex enough that this becomes unwieldy, that's the signal for XState's `invoke` or a query library's mutation.

**Q5. "How do you type actions?"**
A discriminated union on `type`, with each variant carrying exactly its payload. Every `dispatch` call site is then checked, and `assertNever` in the reducer's default makes adding an action a compile error until handled ([TS L09](../../TypeScript/03-type-level/09-illegal-states-unrepresentable.md)).

**Q6. "Is `dispatch` stable?"**
Yes — React guarantees the identity never changes, so it's safe in dependency arrays and `memo`'d children without `useCallback`. (And so is `setState` from `useState`, which people often don't realise.)

**Q7. "Implement undo/redo."**
A reducer wrapper holding `{past, present, future}`: normal actions push `present` onto `past` and clear `future`; `undo` pops from `past`; `redo` pops from `future`. Mention that it's nearly free with a reducer and awkward with `useState` — which is itself the argument for reducers.

**Q8. "When would you use Redux over `useReducer`?"**
Shared state across distant parts of the tree that changes often, state that must survive unmounts, selector-based subscriptions to limit re-renders, or when you genuinely want time-travel devtools and middleware. **Not for server data** — that's a query cache, and conflating the two is the most common state-management mistake.

**Q9. "What's the difference between a reducer and a state machine?"**
A reducer maps `(state, action) → state` with no constraint on which transitions exist. A state machine additionally declares which actions are legal *in each state* — so you switch on state first. The practical payoff is that illegal transitions become unrepresentable rather than something you remember to guard.

---

## 10. Build & break

### Build — Ledger's refund state machine
Implement §4 fully: the state union, the action union, the reducer switching on state first, and the rendering switch. Then write the tests:
```tsx
test("cannot submit twice", ...);
test("cannot go back from submitting", ...);
test("close from any state returns to closed", ...);
test("a late 'succeeded' after close is ignored", ...);
test("amount cannot exceed the payment amount", ...);
test("adding a new step fails the build until handled", ...);   // remove a case, see assertNever fire
```
**Notice that none of these tests render anything.** That's the payoff.

### Build — draw it, then code it
Before writing the reducer, draw the state diagram on paper: boxes for states, arrows for actions. Then write the reducer from the diagram. **They should be the same shape** — if your reducer has transitions the diagram doesn't, you've found a bug in one of them.

### Build — convert a `useState` mess
Take a form with `isLoading`, `error`, `data`, `isDirty`, `touched`, `submitCount` and convert it to a reducer with a discriminated union. Count: states before (2⁶ = 64 representable), states after, and the number of `if` statements you deleted from handlers.

### Break — four experiments
1. **The impossible double submit.** With the machine, click submit twice rapidly (or dispatch twice in a row). Nothing happens. Now add a `submit` case to `submitting` and watch two refunds fire.
2. **Impure reducer.** Put a `fetch` inside the reducer. Run under StrictMode and watch it fire twice.
3. **Mutating reducer.** `state.items.push(x); return state;` — nothing re-renders, because the reference is unchanged.
4. **Late response.** Dispatch `close`, then dispatch `succeeded`. With `default: return state` it's ignored; with a `throw` it crashes. That's why unknown actions must be silent.

### Explain out loud (60 seconds)
1. The four signals to move from `useState` to `useReducer`.
2. Why switching on state first matters.
3. Where side effects go, and why.
4. How you'd prevent a double submit — at both layers.

---

## What's next

Reducers and state cover everything *inside* React's model. Next: the escape hatches for when you need to step outside it — DOM measurement, imperative APIs, values that shouldn't trigger renders — and the rules that keep refs from becoming a mess.

Next → **[Lesson 07: Refs & escape hatches](07-refs-and-escape-hatches.md)**
