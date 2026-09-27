# Lesson 02 — JSX, components & props

> **Why this lesson exists:** props are a component's API, and most React codebases degrade because nobody treats them that way — options accumulate, booleans multiply, and a component ends up with 23 props and four mutually-exclusive modes. This lesson covers JSX's real rules (including the ones that bite) and, more importantly, **how to design a component API** using the same discipline you learned for HTTP endpoints.

**Time:** ~60 minutes · **Prereq:** Lesson 01

---

## 1. The idea in one sentence

> **A component's props are its public contract — design them so that wrong usage is impossible, not merely discouraged.**

That's [API Lesson 01](../../API/01-foundations/01-what-an-api-really-is.md) at component scale, and it's the frame that makes every decision below obvious.

---

## 2. JSX: the rules that actually bite

```tsx
// 1. One root element — or a Fragment
return <><A /><B /></>;                    // shorthand
return <React.Fragment key={id}><A /></React.Fragment>;   // needed when you want a key

// 2. Attributes are camelCase, because they're JS properties
<div className="x" htmlFor="y" tabIndex={0} onClick={fn} aria-label="z" data-id="1" />
//   ^class      ^for          ^tabindex               ^ aria-* and data-* stay kebab

// 3. {} embeds an expression — an EXPRESSION, not a statement
<div>{cond ? <A /> : <B />}</div>          // ✅
<div>{if (cond) ...}</div>                  // ❌ not an expression

// 4. Self-close void elements
<img src="x" />  <br />  <input />

// 5. Capitalised = component, lowercase = host element
<Badge />        → type: Badge (the function)
<badge />        → type: "badge" (an unknown HTML tag — silently renders nothing useful)
```

That last rule causes a real, silent bug: a lowercase component name produces a DOM element React doesn't recognise, and nothing errors.

### Conditional rendering and the falsy traps
```tsx
{isLoading && <Spinner />}                 // ✅ false renders nothing
{count && <Badge count={count} />}          // ❌ count === 0 renders the NUMBER 0 on screen
{count > 0 && <Badge count={count} />}      // ✅
{items.length > 0 && <List items={items} />} // ✅ — .length is the classic offender
```
**`0` is falsy but React renders it.** `false`, `null`, `undefined` and `true` render nothing; `0` and `NaN` render as text. This is the single most common JSX bug, and it's the same family as `||` vs `??` from [TS Lesson 03](../../TypeScript/01-foundations/03-types-as-sets.md).

```tsx
// Multi-branch: prefer an early return or a lookup over nested ternaries
function View({ state }: { state: Async<Payment> }) {
  if (state.status === "loading") return <Skeleton />;
  if (state.status === "error")   return <ErrorView error={state.error} />;
  return <Detail payment={state.data} />;         // ← flat, readable, exhaustive
}
```

### `dangerouslySetInnerHTML`
```tsx
<div dangerouslySetInnerHTML={{ __html: sanitize(userContent) }} />
```
The name is deliberately ugly. It bypasses React's escaping and is a direct XSS vector. **If you use it, the input must be a `SafeHtml` branded type** ([TS Lesson 09](../../TypeScript/03-type-level/09-illegal-states-unrepresentable.md)) produced by a sanitiser — so the type system proves it was sanitised.

### What JSX doesn't do
```tsx
<div style="color: red" />                 // ❌ style takes an object
<div style={{ color: "red", marginTop: 8 }} />   // ✅ camelCase; numbers get "px"
<div style={{ lineHeight: 1.5, zIndex: 10 }} />  // ✅ unitless properties stay unitless
```

---

## 3. Props: the mechanics

```tsx
type Props = {
  payment: Payment;
  onRefund: (id: PaymentId) => void;
  compact?: boolean;
};

function PaymentRow({ payment, onRefund, compact = false }: Props) { /* ... */ }
```

**Props are read-only. Always.**
```tsx
function Bad({ payment }: Props) {
  payment.status = "succeeded";            // ❌ mutating a prop
  return <span>{payment.status}</span>;     // may not even re-render — you changed nothing React tracks
}
```
Mutation breaks the diff (React compares references), breaks memoization, and breaks the one-way data flow that makes the whole model predictable. In TypeScript, `readonly` on props fields is cheap insurance.

### `children` is just a prop
```tsx
<Card>hello</Card>                        // props.children === "hello"
<Card children="hello" />                  // identical, and never write it this way
```
And it can be anything — including a function, which is what a render prop is ([Lesson 13](../04-composition/13-composition-patterns.md)).

### Spreading props
```tsx
<Button {...rest} />                       // forwarding
<Button {...a} {...b} />                    // later wins
```
Forwarding is essential for wrapper components and dangerous when unfiltered:
```tsx
// ❌ leaks internal props to the DOM → React warns about unknown attributes
function Button({ variant, loading, ...rest }: ButtonProps) {
  return <button {...rest} variant={variant} />;   // "variant" isn't a DOM attribute
}
// ✅ extract your own props, forward the rest, use data-* for styling hooks
function Button({ variant, loading, ...rest }: ButtonProps) {
  return <button {...rest} data-variant={variant} disabled={rest.disabled || loading} />;
}
```

### Default values
```tsx
function Button({ variant = "primary" }: Props) {}          // ✅ destructuring default
Button.defaultProps = { variant: "primary" };                // ❌ deprecated for functions in React 19
```
Note the subtlety from [TS Lesson 05](../../TypeScript/02-type-system/05-functions-and-overloads.md): a destructuring default fires on `undefined`, **not** on `null`. `<Button variant={null} />` gives you `null`, not `"primary"`.

---

## 4. Designing a component API

This is the part that matters. Six rules.

### Rule 1 — Prefer composition over configuration
```tsx
// ❌ The configuration trap. Every new requirement adds a prop, forever.
<Modal
  title="Refund payment"
  showCloseButton
  footerButtons={[{ label: "Cancel", variant: "ghost" }, { label: "Refund", variant: "danger" }]}
  size="md"
  icon="warning"
  bodyText="This cannot be undone."
/>

// ✅ Composition. The consumer arranges the parts.
<Modal size="md">
  <Modal.Header icon={<WarningIcon />}>Refund payment</Modal.Header>
  <Modal.Body>This cannot be undone.</Modal.Body>
  <Modal.Footer>
    <Button variant="ghost" onClick={close}>Cancel</Button>
    <Button variant="danger" onClick={refund}>Refund</Button>
  </Modal.Footer>
</Modal>
```
**The test:** when a new requirement arrives ("the footer needs a checkbox"), does the component need changing? With configuration, yes — forever. With composition, no. That's the same "can you remove anything?" minimality test from [API Lesson 01](../../API/01-foundations/01-what-an-api-really-is.md).

**When configuration *is* right:** a genuinely closed set of options (`variant="primary" | "danger"`), and design-system primitives where constraint is the point.

### Rule 2 — Make illegal prop combinations unrepresentable
```tsx
// ❌ 2³ = 8 combinations; several are nonsense
type Props = { isLoading?: boolean; error?: string; data?: Payment[] };

// ✅ 3 states, each carrying exactly its data
type Props = { state: Async<Payment[]> };

// ❌ href AND onClick both allowed — what should it render?
type Props = { href?: string; onClick?: () => void };

// ✅ XOR — exactly one
type Props = XOR<{ href: string }, { onClick: () => void }>;
```
Straight from [TS Lesson 09](../../TypeScript/03-type-level/09-illegal-states-unrepresentable.md). **Count representable prop combinations vs meaningful ones** — the gap is where support tickets come from.

### Rule 3 — Boolean props don't scale
```tsx
// ❌ 2⁴ = 16 combinations, of which ~4 make sense
<Button primary secondary large small />

// ✅ closed unions
<Button variant="primary" size="lg" />
```
**The rule:** more than two related booleans means you wanted a union. And a boolean that's almost always `true` (`showIcon`) is usually a sign the consumer should pass the icon instead.

### Rule 4 — Name props for *what they are*, not *what they do internally*
```tsx
// ❌ leaks implementation
<Table useVirtualization enableMemoization rowRendererStrategy="fast" />
// ✅ describes intent
<Table rows={payments} renderRow={r => <PaymentRow payment={r} />} />
```
Implementation-shaped props are the component equivalent of exposing your database schema in an API response.

### Rule 5 — Callbacks are `onSomethingHappened`, not `handleSomething`
```tsx
type Props = {
  onRefund: (id: PaymentId) => void;         // ✅ the component reports an event
  onSelectionChange: (ids: PaymentId[]) => void;
  handleClick: () => void;                    // ❌ `handle*` is the HANDLER's name, not the prop's
};
```
Convention: `on*` for props you receive, `handle*` for functions you define. Consistent naming in a 200-component codebase is worth more than it looks.

And pass **the data, not the event**, unless the consumer needs the event:
```tsx
onChange: (value: string) => void;                // ✅ usually
onChange: (e: React.ChangeEvent<HTMLInputElement>) => void;   // only if they need preventDefault etc.
```

### Rule 6 — The minimal-props test
For every prop, ask: **can this be derived?**
```tsx
// ❌ two sources of truth that can disagree
<PaymentList payments={payments} count={payments.length} isEmpty={payments.length === 0} />
// ✅
<PaymentList payments={payments} />
```
Derived props are the component version of denormalised data: they can drift, and there's no compiler to catch it.

---

## 5. The component-size question

**When do you split a component?** The useful answers, in order of reliability:

| Split when | Why |
|---|---|
| **A part needs its own state and the parent doesn't care** | Isolating state also isolates re-renders ([Lesson 20](../06-performance/20-measuring.md)) |
| **A part is reused** | The obvious one |
| **A part re-renders on a different cadence** | An input that updates 30×/s shouldn't re-render a table |
| **You need a `key` to control identity** | Lists, tabs, conditional forms |
| **You can name it honestly** | If the best name is `<TopSectionPart2>`, it isn't a component yet |

**Don't split because a file hit 200 lines.** A long component with one coherent job is easier to read than five files you must hold in your head simultaneously. Premature extraction is as real a problem as premature optimisation, and it costs more because it's harder to undo.

> The strongest signal for splitting is **different re-render cadence** — it's the one with a measurable payoff, and it's the one juniors never cite.

---

## 6. Ledger Console's component API

```tsx
// ── Primitive: constrained by design ────────────────────────────
type ButtonProps = React.ComponentPropsWithoutRef<"button"> & {
  variant?: "primary" | "secondary" | "ghost" | "danger";
  size?: "sm" | "md" | "lg";
  loading?: boolean;
};

// ── Compound: composition, not configuration ────────────────────
<DataTable rows={payments} getRowId={p => p.id}>
  <DataTable.Column<Payment> header="ID"     cell={p => <Mono>{p.id}</Mono>} />
  <DataTable.Column<Payment> header="Amount" cell={p => formatMoney(p.money)} align="right" />
  <DataTable.Column<Payment> header="Status" cell={p => <StatusBadge status={p.status} />} />
  <DataTable.Empty>No payments yet</DataTable.Empty>
</DataTable>

// ── Async boundary: one state prop, exhaustively handled ────────
function PaymentDetail({ id }: { id: PaymentId }) {
  const query = usePayment(id);
  switch (query.state) {
    case "loading": return <DetailSkeleton />;
    case "error":   return <ErrorState error={query.error} onRetry={query.refetch} />;
    case "success": return <PaymentView payment={query.data} />;
    default:        return assertNever(query);
  }
}

// ── Action component: XOR so it can't be both ───────────────────
type ActionProps = XOR<{ href: string }, { onClick: () => void }> & {
  children: React.ReactNode;
  variant?: ButtonProps["variant"];         // ← derive, don't duplicate the union
};
```

That last line is worth noticing: `ButtonProps["variant"]` means the two components can't drift apart. Same principle as deriving types from a schema.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Props are read-only. Never mutate them** | Breaks the diff, memoization and one-way flow |
| **`{count > 0 && ...}`, never `{count && ...}`** | `0` is falsy but renders as text |
| **Composition over configuration for anything with a layout** | New requirements shouldn't mean new props forever |
| **Closed unions, not multiple booleans** | 4 booleans is 16 combinations |
| **Make illegal prop combinations unrepresentable (`XOR`, discriminated unions)** | Count representable vs meaningful |
| **Never pass derived data as a prop** | Two sources of truth that can disagree |
| **`on*` for props, `handle*` for your functions** | Consistency at 200 components is worth real money |
| **Extract your props before spreading the rest onto a DOM element** | Otherwise React warns about unknown attributes |
| **Capitalise component names** | Lowercase silently becomes an unknown HTML tag |
| **`dangerouslySetInnerHTML` only with a branded sanitised type** | It's a direct XSS vector |
| **Split by state ownership and re-render cadence, not line count** | Premature extraction costs more than a long file |

---

## 8. Interview traps

**Q1. "What does `{items.length && <List/>}` render when the list is empty?"**
The number **`0`**, visibly, on the page. `0` is falsy so `&&` returns it, and React renders numbers. Use `items.length > 0 &&`. The most common JSX bug there is.

**Q2. "How would you design a Modal component's API?"**
Composition: `Modal.Header/Body/Footer` as children, with only genuinely-closed options (`size`) as props. **Then give the test:** when a new requirement arrives — "the footer needs a checkbox" — a configuration-based modal needs a code change and a composition-based one doesn't.

**Q3. "This component has 23 props. What do you do?"**
Diagnose before refactoring: how many are (a) derivable from others, (b) booleans that should be a union, (c) layout options that should be children, (d) mutually exclusive modes that should be separate components? Usually it's a component doing three jobs — and the fix is splitting, not tidying the props list.

**Q4. "Why can't you mutate props?"**
React compares references to decide what changed; mutation means the value changed but the reference didn't, so React sees nothing. It also breaks memoization and the one-way data flow that makes the model predictable. In TypeScript, mark props `readonly`.

**Q5. "When do you split a component?"**
State ownership, reuse, **different re-render cadence**, needing a `key`, and being able to name it honestly. **Not line count.** The re-render-cadence answer is the one that shows performance awareness.

**Q6. "What's the difference between `<Foo />` and `Foo()`?"**
`<Foo />` creates an element whose `type` is the function — React manages it, so it gets its own fiber, state, hooks and a place in DevTools. `Foo()` calls it immediately and inlines the result into the parent, so it has none of those. Looks identical, breaks the moment `Foo` uses a hook.

**Q7. "How do you type a component that wraps a native element?"**
`React.ComponentPropsWithoutRef<"button"> & { yourProps }`, then destructure yours and spread the rest. `Omit` first if you're overriding a native prop's type ([TS Lesson 17](../../TypeScript/05-ecosystem/17-typescript-with-react.md)).

**Q8. "Two props are mutually exclusive. How do you enforce it?"**
`XOR<A, B>` at the type level so it's a compile error at the call site — better than a runtime `if` or a PropTypes warning, because it's caught where the mistake is made.

**Q9. "Where does XSS come from in React?"**
React escapes interpolated text by default, so the usual vectors are: `dangerouslySetInnerHTML`, `href={userInput}` (`javascript:` URLs), spreading unvalidated props onto an element, and rendering server-provided content into a `<script>`-adjacent context. The defence is validation at the boundary plus a branded `SafeHtml` type.

---

## 9. Build & break

### Build — refactor a configuration monster
Take this and fix it:
```tsx
<PaymentCard
  payment={payment}
  showRefundButton
  showCaptureButton
  showCustomer
  compact
  highlighted
  onRefundClick={fn}
  onCaptureClick={fn}
  footerText="Processed by Ledger"
  badgeColor="green"
  isLoading={false}
  hasError={false}
  errorMessage=""
/>
```
Produce: a composition-based API, closed unions instead of booleans, one `state` prop for the async triple, derived values removed, and `XOR` where two props conflict. **Write down how many props you eliminated** — that number is the exercise.

### Build — Ledger Console's primitives
`Button`, `Badge`, `Action` (XOR), `Modal` (compound), `AsyncBoundary` (one `state` prop, exhaustive switch). Type each with `ComponentPropsWithoutRef`. Add `@ts-expect-error` tests for the illegal usages:
```tsx
// @ts-expect-error — can't be both a link and a button
<Action href="/x" onClick={fn}>Go</Action>
// @ts-expect-error — invalid variant
<Button variant="rainbow">Hi</Button>
```

### Break — five experiments
1. **The `0` bug.** Render `{payments.length && <List/>}` with an empty array. Find the stray `0` on the page.
2. **Prop mutation.** Mutate a prop object in a child and watch the parent not re-render — then watch it "work" inconsistently when something else triggers a render. That inconsistency is the real danger.
3. **Lowercase component.** Rename `<Badge/>` to `<badge/>`. It renders nothing, silently, with no error.
4. **Unfiltered spread.** Spread a props object containing `variant` onto a `<button>`. Read React's unknown-attribute warning.
5. **`null` vs `undefined` defaults.** `function B({v = "primary"})` called as `<B v={null}/>`. The default doesn't fire.

### Explain out loud (60 seconds)
1. The falsy-render trap and its fix.
2. Composition vs configuration, with the new-requirement test.
3. Three ways to make illegal prop combinations unrepresentable.
4. When you split a component — and the reason juniors don't cite.

---

## What's next

You can write and design components. Now the question that underlies every React performance problem and half its bugs: **when exactly does React re-render, and what happens when it does?**

Next → **[Lesson 03: Rendering — render vs commit](03-rendering-and-commit.md)**
