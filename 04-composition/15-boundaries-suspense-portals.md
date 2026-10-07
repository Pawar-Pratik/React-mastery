# Lesson 15 — Error boundaries, Suspense & portals

> **Why this lesson exists:** these three features share one job — **controlling where something happens in the tree.** Error boundaries decide where a crash stops, Suspense decides where a loading state appears, portals decide where DOM is rendered. Get them right and one broken widget doesn't white-screen your dashboard; get them wrong and a failed revenue chart takes down the payments table with it.

**Time:** ~65 minutes · **Prereq:** Lessons 03, 13

---

## 1. The idea in one sentence

> **Boundaries are placement decisions: where failure stops, where loading appears, and where DOM lands — and choosing those positions well is an architecture skill, not an API one.**

---

## 2. Error boundaries

```tsx
// Still must be a class component — there's no hook equivalent
class ErrorBoundary extends React.Component<
  { fallback: (error: Error, reset: () => void) => React.ReactNode; children: React.ReactNode;
    onError?: (error: Error, info: React.ErrorInfo) => void },
  { error: Error | null }
> {
  state = { error: null as Error | null };

  static getDerivedStateFromError(error: Error) {
    return { error };                                    // render phase: update state to show a fallback
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    this.props.onError?.(error, info);                   // commit phase: side effects (logging)
    logger.error({ err: error, componentStack: info.componentStack }, "react_error_boundary");
  }

  render() {
    if (this.state.error) {
      return this.props.fallback(this.state.error, () => this.setState({ error: null }));
    }
    return this.props.children;
  }
}
```

**The two methods do different jobs:** `getDerivedStateFromError` runs during render (so it must be pure — no logging) and returns the new state. `componentDidCatch` runs after commit and is where side effects like reporting belong.

### What error boundaries catch — and the four things they don't

```
✅ Errors thrown during render
✅ Errors in lifecycle methods
✅ Errors in constructors of the subtree

❌ Event handlers            ← the big one
❌ Async code (setTimeout, promises, fetch callbacks)
❌ Server-side rendering
❌ Errors thrown in the boundary itself
```

**Event handlers are the most important exclusion**, because that's where most errors actually are:
```tsx
// ❌ NOT caught by any boundary — React isn't rendering when this runs
<button onClick={() => { throw new Error("boom"); }} />

// ✅ Handle it yourself
<button onClick={async () => {
  const result = await refund(id);                   // Result, not throw (TS Lesson 16)
  if (!result.ok) setError(result.error);
}} />

// ✅ Or route it into a boundary deliberately
const [, setError] = useState();
onClick={() => { try { risky(); } catch (e) { setError(() => { throw e; }); } }}
// setting a thrower into state makes it throw during the NEXT render, where a boundary sees it
```
That last trick is what `react-error-boundary`'s `useErrorBoundary().showBoundary(error)` does, and it's worth knowing — it's how you get async errors into a boundary.

### Placement is the whole decision

```tsx
// ❌ One boundary at the root: any error blanks the entire app
<ErrorBoundary fallback={<FullPageError />}>
  <App />
</ErrorBoundary>

// ✅ Layered — failure is contained at the smallest sensible unit
<ErrorBoundary fallback={<FullPageError />}>            {/* last resort */}
  <Layout>
    <ErrorBoundary fallback={<RouteError />}>            {/* per route */}
      <PaymentsPage>
        <ErrorBoundary fallback={<WidgetError name="Revenue chart" />}>
          <RevenueChart />                                {/* a broken chart ≠ a broken page */}
        </ErrorBoundary>
        <PaymentsTable />                                 {/* still works */}
      </PaymentsPage>
    </ErrorBoundary>
  </Layout>
</ErrorBoundary>
```

**The placement rule:** *"put a boundary wherever the surrounding UI is still useful without this piece."* A revenue chart failing shouldn't hide the payments table. A single table row failing shouldn't kill the table.

### Recovery
```tsx
<ErrorBoundary
  fallback={(error, reset) => (
    <div role="alert">
      <p>Couldn't load the chart.</p>
      <button onClick={reset}>Try again</button>
      {import.meta.env.DEV && <pre>{error.stack}</pre>}     {/* dev only */}
    </div>
  )}
/>
```
**A reset that re-renders the same failing component just fails again.** Real recovery needs the *cause* to change too:
```tsx
// Reset when the route changes — a key change remounts the boundary (Lesson 04)
<ErrorBoundary key={location.pathname} fallback={...}>
// Or expose resetKeys, as react-error-boundary does:
<ErrorBoundary resetKeys={[paymentId]} onReset={() => query.refetch()}>
```

> **Use the `react-error-boundary` library rather than hand-rolling.** It gives you `resetKeys`, `onReset`, `useErrorBoundary().showBoundary()` for async errors, and a `withErrorBoundary` HOC — all things you'd otherwise reimplement badly.

**And report to your monitoring:** `componentDidCatch` is where Sentry/Datadog integration goes, with the `componentStack` attached. A boundary that silently swallows errors is worse than a crash, because nobody finds out.

---

## 3. Suspense

```tsx
<Suspense fallback={<TableSkeleton rows={10} />}>
  <PaymentsTable />          {/* any component inside may "suspend" */}
</Suspense>
```

**The mechanism:** a component signals it isn't ready by throwing a promise (conceptually — React 19 uses `use`). React walks up to the nearest `<Suspense>`, shows its fallback, and retries when the promise resolves. **It's `try/catch` for loading states**, with the boundary deciding where the fallback appears.

### What actually suspends today
```
✅ React.lazy() — code splitting
✅ use(promise) in React 19
✅ Server Components awaiting data (Lesson 23)
✅ Framework data loaders — Next.js, Remix, TanStack Router
✅ Libraries opting in — TanStack Query with `useSuspenseQuery`, Relay

❌ A plain fetch in useEffect         ← does NOT suspend
❌ A promise created during render     ← infinite suspend loop
```
That second exclusion catches people: writing `use(fetch(url))` inside a Client Component creates a new promise every render, so it never stabilises.

### Code splitting — the universal use
```tsx
const RevenueChart = lazy(() => import("./RevenueChart"));       // a separate bundle chunk

<Suspense fallback={<ChartSkeleton />}>
  <RevenueChart data={data} />
</Suspense>
```
```tsx
// Route-level splitting — the highest-value split (Lesson 22)
const PaymentsPage = lazy(() => import("./pages/Payments"));
const SettingsPage = lazy(() => import("./pages/Settings"));
```

**Pair every `lazy` with an error boundary**, because a chunk can fail to load (network, or a stale chunk after a deploy):
```tsx
<ErrorBoundary fallback={(e, reset) => <ChunkLoadError onRetry={reset} />}>
  <Suspense fallback={<ChartSkeleton />}><RevenueChart /></Suspense>
</ErrorBoundary>
```
**Stale chunks after a deploy are a real production issue:** a user with the old HTML requests a chunk hash that no longer exists. The standard fix is a full reload on chunk-load failure.

### Placement, again
```tsx
// ❌ One boundary: the whole page waits for the slowest thing
<Suspense fallback={<PageSkeleton />}>
  <Header /><PaymentsTable /><RevenueChart /><RecentActivity />
</Suspense>

// ✅ Independent boundaries: each part appears as it's ready
<Header />
<Suspense fallback={<TableSkeleton />}><PaymentsTable /></Suspense>
<Suspense fallback={<ChartSkeleton />}><RevenueChart /></Suspense>
<Suspense fallback={<ActivitySkeleton />}><RecentActivity /></Suspense>
```
**One boundary per independently-loadable region**, and each fallback should be roughly the *shape and size* of the real content — otherwise you get layout shift when it swaps in, which is a CLS problem ([Lesson 22](../06-performance/22-loading-performance.md)).

### Suspense + transitions: avoiding the fallback flash
```tsx
// ❌ Navigating replaces content with a skeleton — jarring if the data is fast
setTab("refunds");

// ✅ In a transition, React keeps the OLD content visible while the new one loads
startTransition(() => setTab("refunds"));
```
**This is the most important Suspense interaction to know** ([Lesson 11](../03-hooks/11-concurrent-hooks.md)): an update inside a transition doesn't re-show an already-revealed Suspense fallback. You keep the previous screen (dimmed via `isPending`) instead of flashing a skeleton for 80ms.

```tsx
// SuspenseList existed in experimental builds to orchestrate reveal order.
// It's NOT in stable React — don't claim it in an interview. Achieve ordering with
// boundary placement and streaming order instead (Lesson 25).
```

---

## 4. Portals

```tsx
import { createPortal } from "react-dom";

function Modal({ open, onClose, children }: ModalProps) {
  if (!open) return null;
  return createPortal(
    <div className="overlay" onClick={onClose}>
      <div className="modal" role="dialog" aria-modal="true" onClick={e => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body,             // ← rendered HERE in the DOM…
  );
}
```

**The key property:** a portal moves the *DOM position* but keeps the *React tree position*. So:
- Context still flows through ✅
- **Events still bubble through the React tree**, not the DOM tree ✅
- Error boundaries above still catch ✅
- But CSS `overflow: hidden`, `z-index` stacking contexts and `transform` on ancestors no longer trap it ✅

**Why that matters:** a dropdown inside a scrollable, `overflow: hidden` table gets clipped. A modal inside a `transform`ed ancestor creates a new containing block and can't be centred on the viewport. Portals are the fix for both, and that's genuinely what they're for.

```tsx
// The event-bubbling surprise, and it's a real one:
<div onClick={() => console.log("parent")}>       {/* React tree parent */}
  <Modal>{/* portalled to document.body */}
    <button onClick={() => console.log("button")} />
  </Modal>
</div>
// Clicking the button logs: "button", then "parent"
// …even though in the DOM they're in completely different subtrees.
```
This bites when you attach a document-level click-outside listener: the portal's click bubbles through React to the trigger's ancestors *and* reaches `document`, so a naive outside-click handler closes the modal immediately. Fix with `event.stopPropagation()` in the portal, or check `containerRef.current.contains(e.target)`.

### What a modal actually needs
A portal is about 5% of a correct modal. The rest:
```
[ ] Focus trap — Tab cycles within the modal
[ ] Focus moves INTO the modal on open (the first focusable, or the dialog itself)
[ ] Focus RETURNS to the trigger on close
[ ] Escape closes it
[ ] Background is inert — `inert` attribute or aria-hidden on siblings
[ ] Scroll lock on body, restoring the PREVIOUS value (Lesson 08)
[ ] role="dialog" aria-modal="true" aria-labelledby pointing at the title
[ ] Click-outside to close (optional, and must not fire on drag-release)
```
**Use `<dialog>` or Radix Dialog.** Hand-rolling focus management correctly takes days and you'll still miss cases. The native `<dialog showModal()>` gives you focus trap, `Escape`, inertness and the top layer for free — it's the modern answer and worth knowing.

```tsx
// Other genuine portal uses
createPortal(<Toast />, document.getElementById("toast-root")!);
createPortal(<Tooltip />, document.body);          // escapes overflow:hidden
createPortal(<Dropdown />, document.body);          // escapes table scroll containers
```

---

## 5. Composing all three

```tsx
function PaymentsPage() {
  return (
    <ErrorBoundary fallback={(e, reset) => <PageError error={e} onRetry={reset} />}>
      <Page title="Payments" actions={<ExportButton />}>

        {/* Independent boundary: the chart can fail or load slowly on its own */}
        <ErrorBoundary fallback={<WidgetError name="Revenue" />}>
          <Suspense fallback={<ChartSkeleton height={240} />}>
            <RevenueChart />
          </Suspense>
        </ErrorBoundary>

        {/* The table is the primary content — its own boundaries */}
        <ErrorBoundary fallback={(e, reset) => <TableError error={e} onRetry={reset} />}>
          <Suspense fallback={<TableSkeleton rows={10} />}>
            <PaymentsTable />
          </Suspense>
        </ErrorBoundary>

      </Page>

      {/* Portalled, so it escapes any overflow/transform on the layout */}
      <ToastViewport />
    </ErrorBoundary>
  );
}
```

**The ordering matters:** `ErrorBoundary` **outside** `Suspense`. If a lazy chunk fails to load, the error must be caught — and a boundary inside the Suspense would be part of the not-yet-loaded subtree.

---

## 6. Production rules

| Rule | Why |
|---|---|
| **Error boundaries layered: root, route, and per independent widget** | A broken chart shouldn't hide the table |
| **Place a boundary wherever the surrounding UI is still useful without this piece** | The placement rule that decides everything |
| **Error boundaries do NOT catch event handlers or async code** | Handle those yourself, or push them into a boundary deliberately |
| **`componentDidCatch` reports to monitoring with the component stack** | A boundary that silently swallows is worse than a crash |
| **Reset needs the cause to change — `resetKeys` or a `key`** | Re-rendering the same failing component just fails again |
| **Use `react-error-boundary`, not a hand-rolled class** | `resetKeys`, `showBoundary`, and the HOC are all things you'd get wrong |
| **One Suspense boundary per independently-loadable region** | Otherwise the page waits for the slowest part |
| **Fallbacks should match the real content's shape and size** | Prevents layout shift (CLS) |
| **Wrap every `lazy` in an error boundary** | Chunks fail, especially after a deploy |
| **Navigate inside a transition to avoid fallback flashes** | React keeps the old content visible |
| **ErrorBoundary outside Suspense** | So chunk-load failures are caught |
| **Portals keep React-tree events and context** | Outside-click handlers must account for it |
| **Use `<dialog>` or Radix for modals** | Focus management is days of work to get right |

---

## 7. Interview traps

**Q1. "What do error boundaries catch?"**
Render, lifecycle and constructor errors in their subtree. **Not** event handlers, async code, SSR, or errors in the boundary itself. **Event handlers being excluded is the important half** — that's where most errors are, so you handle them yourself or push them into a boundary by setting a thrower into state (which is what `showBoundary` does).

**Q2. "Can an error boundary be a function component?"**
No — `getDerivedStateFromError` and `componentDidCatch` have no hook equivalents, so it must be a class. It's the main remaining reason class components exist. In practice you use `react-error-boundary` and never write the class.

**Q3. "Where do you put error boundaries?"**
Layered: root (last resort), per route, and around any widget whose failure shouldn't take down its siblings. **The rule:** wherever the surrounding UI is still useful without this piece.

**Q4. "What is Suspense, mechanically?"**
A component signals it isn't ready (conceptually by throwing a promise); React walks up to the nearest boundary, renders its fallback, and retries on resolution. It's `try/catch` for loading, with the boundary choosing where the fallback appears. Today it's triggered by `lazy`, `use`, Server Components, framework loaders, and opted-in libraries — **not by a plain `useEffect` fetch.**

**Q5. "How do you avoid a loading flash when navigating?"**
Wrap the navigation in `startTransition`. React keeps the already-revealed content visible instead of reverting to the fallback, and you show `isPending` as a subtle dim or progress bar. **The most valuable Suspense interaction to know.**

**Q6. "Error boundary inside or outside Suspense?"**
Outside. A failed lazy chunk must be caught, and a boundary inside the Suspense is part of the subtree that never loaded.

**Q7. "What's a portal and when do you need one?"**
Renders DOM elsewhere (usually `document.body`) while keeping the React tree position — so context and event bubbling still work through React. Needed when an ancestor's `overflow: hidden`, `z-index` stacking context or `transform` would clip or mis-position the content: modals, tooltips, dropdowns inside scroll containers.

**Q8. "Where do portal events bubble?"**
Through the **React tree**, not the DOM tree. So a click inside a portal fires handlers on the component's React ancestors, even though they're elsewhere in the DOM. This surprises people implementing click-outside — the click reaches both the React parent and `document`.

**Q9. "What does a correct modal need beyond a portal?"**
Focus trap, focus-in on open, focus-return on close, `Escape`, inert background, scroll lock with restore, and `role="dialog"` + `aria-modal` + `aria-labelledby`. **Then say you'd use `<dialog>` or Radix rather than reimplement it**, because focus management is days of work and you'd still miss cases.

**Q10. "A chunk fails to load after a deploy. What happens and what do you do?"**
The user has old HTML referencing a chunk hash that no longer exists, so `lazy` rejects. Catch it in an error boundary and offer a reload (or reload automatically once, guarded against loops). Longer-term: keep old chunks available for a grace period, or version the app shell.

---

## 8. Build & break

### Build — the boundary layer for Ledger Console
1. `ErrorBoundary` via `react-error-boundary`, reporting to your logger with the component stack.
2. Boundaries at root, per route (keyed on `location.pathname` so navigating resets), and around the revenue chart.
3. Suspense boundaries with skeletons **matched to the real content's dimensions**.
4. `ToastViewport` and `Modal` via portals.
5. A `lazy`-loaded route with a chunk-error fallback offering a reload.

Then break each deliberately: throw in the chart's render and confirm the table still works; reject a lazy import and confirm the reload prompt appears.

### Build — the flash comparison
Build tabs where one tab's content suspends for ~400ms. Switch tabs without a transition (skeleton flashes) and with one (old content stays, dimmed). **Record both with a screen recording** — the difference is obvious and it's a great thing to be able to describe from experience.

### Break — five experiments
1. **Event-handler error.** Throw inside `onClick` with a boundary above. Nothing catches it — it hits `window.onerror`. Then use `showBoundary` and watch the boundary catch it.
2. **Async error.** Throw inside a `setTimeout`. Same result.
3. **Reset without a cause change.** Call `reset()` on a boundary wrapping a component that always throws. It immediately re-throws. Add `resetKeys` and change the key.
4. **Portal click-outside.** Implement a document-level click-outside handler for a portalled dropdown. It closes immediately on the opening click, because the click reaches `document`. Fix it.
5. **Missing focus trap.** Open a hand-rolled modal, press Tab six times, and watch focus land behind the overlay. Then compare with `<dialog showModal()>`.

### Explain out loud (60 seconds)
1. What error boundaries catch and what they don't.
2. The placement rule for both kinds of boundary.
3. How Suspense works mechanically.
4. Why transitions prevent fallback flashes.
5. What a portal preserves and what it escapes.

---

## What's next

You can build components, forms and boundaries. The last piece of Module 4 is the layer above: routing — including why the URL is the best state container you have — and the folder architecture that survives 200 components.

Next → **[Lesson 16: Routing & app architecture](16-routing-and-architecture.md)**
