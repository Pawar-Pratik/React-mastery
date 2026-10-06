# Lesson 09 — Context, properly

> **Why this lesson exists:** context is the most commonly over-reached-for tool in React. People use it as a state manager, discover their app re-renders constantly, and conclude "context is slow." Context isn't slow — **it just has no selector mechanism**, which means every consumer re-renders when any part of the value changes. Once you know that precisely, you know exactly what context is for and when to reach for something else.

**Time:** ~60 minutes · **Prereq:** Lesson 03

---

## 1. The idea in one sentence

> **Context is a *dependency injection* mechanism, not a state manager — it solves "how do I get this value down there" and says nothing about "how do I avoid re-rendering when it changes."**

---

## 2. The problem it solves: prop drilling

```tsx
// ❌ `principal` threaded through four components that don't use it
<App principal={principal}>
  <Layout principal={principal}>
    <Sidebar principal={principal}>
      <UserMenu principal={principal} />
    </Sidebar>
  </Layout>
</App>
```

Prop drilling is *tedious*, but be precise about its actual costs — it's not automatically wrong:

| Real cost | Not a cost |
|---|---|
| Intermediate components must know about data they don't use | Performance — passing props is free |
| Refactoring requires touching every level | Correctness — it's explicit and traceable |
| Component APIs get noisy | Testability — explicit props are easier to test |

**And composition often eliminates it entirely, with no context at all:**
```tsx
// ✅ Pass the rendered element down instead of the data
<Layout sidebar={<Sidebar userMenu={<UserMenu principal={principal} />} />} />
// Layout and Sidebar never see `principal`
```
**Try composition before reaching for context.** It's the same `children`-passing technique from [Lesson 03](../01-mental-model/03-rendering-and-commit.md), and it also avoids the re-render cascade.

---

## 3. The mechanics

```tsx
type AuthContextValue = {
  principal: Principal | null;
  login: (email: Email, password: string) => Promise<Result<Principal, ApiError>>;
  logout: () => void;
};

// ★ undefined default + a hook that throws — so consumers never handle undefined
const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (ctx === undefined) throw new Error("useAuth must be used within <AuthProvider>");
  return ctx;
}

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [principal, setPrincipal] = useState<Principal | null>(null);
  const login = useCallback(async (email: Email, password: string) => { /* ... */ }, []);
  const logout = useCallback(() => setPrincipal(null), []);

  const value = useMemo(() => ({ principal, login, logout }), [principal, login, logout]);
  return <AuthContext value={value}>{children}</AuthContext>;   // React 19: no .Provider needed
}
```

Three things in there that matter:

**1. The undefined-default + throwing-hook pattern.** The alternative — a fake default object — means a missing provider renders silently broken UI instead of failing loudly at first use. Always do it this way ([TS Lesson 17](../../TypeScript/05-ecosystem/17-typescript-with-react.md)).

**2. `useMemo` on the value is mandatory.** Without it, `{ principal, login, logout }` is a new object every render of the provider, so **every consumer re-renders on every provider render** regardless of whether anything changed.

**3. React 19 dropped `.Provider`.** `<AuthContext value={v}>` works directly. Also `use(AuthContext)` can be called conditionally, unlike `useContext`.

---

## 4. The re-render problem — precisely

```tsx
const AppContext = createContext<AppState | undefined>(undefined);

function Provider({ children }) {
  const [principal, setPrincipal] = useState(null);
  const [theme, setTheme] = useState("light");
  const [toasts, setToasts] = useState([]);          // ← changes on every toast

  const value = useMemo(() => ({ principal, theme, toasts, setTheme, setToasts }),
                        [principal, theme, toasts]);
  return <AppContext value={value}>{children}</AppContext>;
}
```

**Show a toast → `toasts` changes → `value` is a new object → EVERY consumer re-renders**, including components that only read `theme`.

**Context has no selector.** `useContext(C)` subscribes to the *whole* value. There is no `useContext(C, c => c.theme)`. That single limitation is the entire performance story, and stating it precisely is what separates "context is slow" from an actual understanding.

Note the two distinct causes, because people conflate them:
- **Provider re-renders** → new `value` object → all consumers re-render (fixed by `useMemo`)
- **The value genuinely changed** → all consumers re-render, even ones that don't care about the changed part (fixed only by splitting or a store)

### Fix 1 — Split contexts by change frequency
```tsx
const PrincipalContext = createContext<Principal | null>(null);   // changes rarely
const ThemeContext = createContext<Theme>("light");                // changes rarely
const ToastContext = createContext<Toast[]>([]);                   // changes often
```
Now a toast re-renders only toast consumers. **This is the primary fix and it's free** — group values by *how often they change*, not by what feature they belong to.

### Fix 2 — Split state from dispatch (the underrated one)
```tsx
const StateContext = createContext<State | undefined>(undefined);
const DispatchContext = createContext<React.Dispatch<Action> | undefined>(undefined);

function Provider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, initial);
  return (
    <DispatchContext value={dispatch}>          {/* dispatch is STABLE — never changes */}
      <StateContext value={state}>{children}</StateContext>
    </DispatchContext>
  );
}
```
**Components that only dispatch never re-render**, because `dispatch` is referentially stable forever ([Lesson 06](../02-state-effects/06-reducers-and-state-machines.md)). A "add to cart" button, a "dismiss toast" button, a form submit — none of them read state, so none re-render. That's often most of your consumers.

### Fix 3 — Memo the consumer's children
```tsx
function ThemedPanel() {
  const theme = useContext(ThemeContext);
  return <div className={theme}><ExpensiveChildren /></div>;
}
// ❌ ExpensiveChildren re-renders whenever theme changes

function ThemedPanel({ children }: { children: React.ReactNode }) {
  const theme = useContext(ThemeContext);
  return <div className={theme}>{children}</div>;     // ✅ children created by the parent — stable
}
```
Composition again. `memo` **cannot** prevent a re-render caused by context — a memoized component still re-renders when a context it consumes changes. Passing children through is the way around it.

### Fix 4 — An external store with selectors
When a value changes frequently and many components read *different parts* of it, context is structurally the wrong tool:

```tsx
// Zustand — selector-based subscriptions
const useStore = create<AppState>()(set => ({
  principal: null, theme: "light", toasts: [],
  addToast: t => set(s => ({ toasts: [...s.toasts, t] })),
}));

function ThemeToggle() {
  const theme = useStore(s => s.theme);       // ← re-renders ONLY when theme changes
}
```
Under the hood this is `useSyncExternalStore` ([Lesson 11](11-concurrent-hooks.md)) with a per-component selector — which is exactly the mechanism context lacks.

> **The decision rule:** *"context for values that change rarely — auth, theme, locale, feature flags, a stable dispatch. An external store with selectors for values that change often and are read in different slices. And a query cache for server data, which isn't either of those."*

---

## 5. What context is genuinely good for

```tsx
// ✅ Rarely-changing app-wide values
<AuthProvider>      {/* login/logout only */}
<ThemeProvider>     {/* user preference */}
<LocaleProvider>    {/* i18n */}
<FeatureFlagProvider>

// ✅ A stable dispatch or API object
<ToastDispatchContext>     {/* { show, dismiss } — stable functions */}
<ModalContext>              {/* { open, close } */}

// ✅ Implicit compound-component communication — one of the best uses
<Tabs defaultValue="payments">
  <Tabs.List>
    <Tabs.Trigger value="payments">Payments</Tabs.Trigger>
    <Tabs.Trigger value="refunds">Refunds</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="payments"><PaymentsTable /></Tabs.Content>
</Tabs>
```

```tsx
// The compound-component pattern (Lesson 13) — context is the right tool here because
// the scope is small and the value changes only on user interaction within that widget
const TabsContext = createContext<{ value: string; setValue: (v: string) => void } | null>(null);

export function Tabs({ defaultValue, children }: TabsProps) {
  const [value, setValue] = useState(defaultValue);
  const ctx = useMemo(() => ({ value, setValue }), [value]);
  return <TabsContext value={ctx}>{children}</TabsContext>;
}
Tabs.Trigger = function Trigger({ value, children }: TriggerProps) {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error("Tabs.Trigger must be used within <Tabs>");
  return <button aria-selected={ctx.value === value} onClick={() => ctx.setValue(value)}>{children}</button>;
};
```
**Why this is a good use:** the consumers are few and physically nearby, the value changes only on a deliberate user action, and the alternative (prop drilling through `Tabs.List` into every `Trigger`) would make the component API worse.

### Nested providers — a real technique
```tsx
<ThemeContext value="dark">
  <Page>
    <ThemeContext value="light">      {/* ← an island that overrides */}
      <PrintPreview />
    </ThemeContext>
  </Page>
</ThemeContext>
```
`useContext` reads the **nearest** provider above it. Genuinely useful for scoped overrides — a light-themed print preview inside a dark app, a form-level `disabled` that cascades, a nested `<FormProvider>` for a sub-form.

---

## 6. Context anti-patterns

### Anti-pattern 1 — The god context
```tsx
// ❌ Everything in one provider → everything re-renders on any change
<AppContext value={{ user, theme, cart, notifications, settings, modals, toasts }} />
```
Split by change frequency.

### Anti-pattern 2 — Server data in context
```tsx
// ❌ You've hand-rolled a worse query cache
const PaymentsContext = createContext<Payment[]>([]);
```
Server data needs caching, revalidation, deduplication, background refetch, retry and per-query loading states. **That's a query library, not context** ([Lesson 17](../05-data/17-server-state.md)).

### Anti-pattern 3 — Forgetting to memo the value
```tsx
<Ctx value={{ a, b }}>       // ❌ new object every render → every consumer re-renders
```
The most common context performance bug there is.

### Anti-pattern 4 — Context for two levels of drilling
```tsx
// ❌ A provider, a hook, and a new file to avoid passing one prop twice
```
Context has a real cost: indirection, harder testing, harder to trace where a value comes from. **Two or three levels of drilling is fine.** Reach for context when it's five levels, or many consumers, or genuinely app-wide.

### Anti-pattern 5 — Assuming `memo` protects consumers
```tsx
const Memoized = memo(function Child() {
  const theme = useContext(ThemeContext);      // ← memo does NOT stop this re-rendering
  return <div className={theme} />;
});
```
`memo` compares props. Context isn't a prop. A memoized component still re-renders when a consumed context changes — a surprise worth knowing.

---

## 7. Ledger Console's context map

| Value | Mechanism | Why |
|---|---|---|
| `principal`, `login`, `logout` | **Context** | Changes on login/logout only |
| Theme, locale | **Context** | User preference; changes rarely |
| Feature flags | **Context** | Fetched once at boot |
| `toast.show/dismiss` (the API) | **Context** (dispatch only) | Stable functions; consumers never re-render |
| Toast **list** | **Separate context** or store | Changes often; only `<ToastViewport>` reads it |
| Tabs / Accordion / Select internals | **Context, scoped to the widget** | Few nearby consumers, small value |
| Payments, customers, balance | **Query cache** | Server data — caching, dedup, revalidation |
| Filters, sort, pagination | **URL** | Shareable, back/forward, refresh-safe |
| Table column widths, sidebar collapsed | **localStorage + store** | Preferences that survive reloads |

```tsx
// The split-context toast system — the pattern worth stealing
const ToastListContext = createContext<Toast[]>([]);
const ToastApiContext = createContext<ToastApi | null>(null);

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);

  // The API object is created ONCE — consumers of it never re-render
  const api = useMemo<ToastApi>(() => ({
    show: t => setToasts(prev => [...prev, { ...t, id: crypto.randomUUID() }]),
    dismiss: id => setToasts(prev => prev.filter(t => t.id !== id)),
  }), []);

  return (
    <ToastApiContext value={api}>
      <ToastListContext value={toasts}>{children}</ToastListContext>
    </ToastApiContext>
  );
}

export const useToast = () => {
  const api = useContext(ToastApiContext);
  if (!api) throw new Error("useToast must be used within <ToastProvider>");
  return api;
};
```
**Every component that *shows* a toast uses `useToast()` and never re-renders when toasts change.** Only `<ToastViewport>` reads the list. That's the split-context pattern earning its keep.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Try composition before context** | Passing `children` or elements down often removes the need entirely |
| **`undefined` default + a throwing hook** | A missing provider fails loudly instead of rendering broken UI |
| **Always `useMemo` the context value** | Otherwise every provider render re-renders every consumer |
| **Split contexts by change frequency, not by feature** | A toast shouldn't re-render theme consumers |
| **Split state from dispatch** | Dispatch-only consumers never re-render |
| **Context for rarely-changing values; a store with selectors for frequent ones** | Context has no selector mechanism |
| **Never put server data in context** | You're hand-rolling a worse query cache |
| **`memo` does not protect against context changes** | It compares props; context isn't a prop |
| **Two or three levels of drilling is fine** | Context costs indirection and traceability |
| **Nested providers for scoped overrides** | `useContext` reads the nearest one |

---

## 9. Interview traps

**Q1. "What's context for?"**
Dependency injection — getting a value to distant descendants without threading it through every level. **Not a state manager:** it has no selector, so every consumer re-renders when any part of the value changes. That's the sentence that shows you understand the limitation rather than repeating "context is slow."

**Q2. "A context value changes — who re-renders?"**
Every component calling `useContext` on that context, **regardless of which part of the value they use** — and `memo` doesn't help, because context isn't a prop. Fixes: split by change frequency, split state from dispatch, pass children through, or move to a store with selectors.

**Q3. "Why do all my consumers re-render even when nothing changed?"**
The provider's value is a new object each render. `useMemo` it with the right dependencies. **The single most common context bug.**

**Q4. "How do you avoid re-rendering components that only dispatch?"**
Two contexts: one for state, one for `dispatch`. `dispatch` from `useReducer` is referentially stable forever, so dispatch-only consumers never re-render. Often that's most of them.

**Q5. "Context or Redux/Zustand?"**
Context for rarely-changing app-wide values — auth, theme, locale, flags, a stable API object. A store for frequently-changing state read in different slices, because stores give you **selector-based subscriptions**, which is precisely what context lacks. And neither for server data.

**Q6. "Is context slow?"**
No — the propagation itself is efficient. What's slow is the *consequence*: no selectors means unnecessary re-renders when a large value changes. Being precise here is the differentiator.

**Q7. "How do you type a context so consumers never handle `undefined`?"**
Default to `undefined`, export a hook that throws if it's missing. The throw narrows the type, so consumers get a non-nullable value and a missing provider fails at first use with a clear message.

**Q8. "Can you nest the same context?"**
Yes — `useContext` reads the nearest provider above. Useful for scoped overrides: a light-themed island in a dark app, a nested form provider, a `disabled` cascade.

**Q9. "When would you NOT use context?"**
Two or three levels of drilling (just pass the prop); server data (query cache); frequently-changing state with many partial readers (store); and anything composition solves by passing elements down.

**Q10. "How would you build a toast system?"**
Split contexts: a stable `{show, dismiss}` API object memoized with `[]`, and a separate list context that only the viewport reads. Every component that shows a toast is a consumer of the API only and never re-renders. **Describing this unprompted is a strong signal** — it shows you design for re-render behaviour, not just for function.

---

## 10. Build & break

### Build — the context re-render lab
```tsx
const AppContext = createContext<any>(null);

function Provider({ children }) {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState("light");
  const value = { count, theme, setCount, setTheme };   // ❌ no memo, deliberately
  return <AppContext value={value}>{children}</AppContext>;
}

function CountReader() { const { count } = useContext(AppContext); console.log("count reader"); return <b>{count}</b>; }
function ThemeReader() { const { theme } = useContext(AppContext); console.log("theme reader"); return <b>{theme}</b>; }
```
1. Change `count`. **Both** log. That's the no-selector problem.
2. Add `useMemo`. Changing `count` still re-renders `ThemeReader`, because the value genuinely changed. **This is the key observation** — memo fixes the provider-render case, not the value-changed case.
3. Split into two contexts. Now only the relevant reader logs.
4. Wrap `ThemeReader` in `memo` and go back to one context — it still re-renders. Prove to yourself that `memo` doesn't help.

### Build — Ledger's providers
Implement `AuthProvider`, `ThemeProvider`, and the split `ToastProvider` from §7. Then verify with the Profiler: showing a toast must re-render **only** `ToastViewport`, not the page.

### Break — four experiments
1. **Missing provider.** Call `useAuth()` outside `AuthProvider`. With the throwing hook you get a clear error; with a fake default you get silently broken UI. Try both.
2. **Unmemoized value.** Add a `console.log` to every consumer and click something unrelated in the provider. Watch them all fire.
3. **God context.** Put ten values in one provider and change one. Count the re-renders with "highlight updates" on.
4. **Nested override.** Nest a `ThemeContext` with a different value and confirm descendants read the nearest one.

### Explain out loud (60 seconds)
1. What context is for, and the one thing it lacks.
2. The two distinct causes of consumer re-renders.
3. Four ways to limit them.
4. When you'd reach for a store instead.

---

## What's next

You've seen three different re-render problems now, and reached for `memo` and `useMemo` without a proper account of when they help. Next: the memoization hooks — what they actually do, the three reasons `useMemo` doesn't help, and why the React Compiler may make most of this obsolete.

Next → **[Lesson 10: memo, useMemo, useCallback](10-memoization.md)**
