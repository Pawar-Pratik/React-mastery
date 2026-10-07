# Lesson 13 — Component design & composition

> **Why this lesson exists:** the difference between a codebase that stays pleasant at 200 components and one that becomes a prop-soup nightmare is almost entirely composition discipline. These patterns — compound components, slots, render props, headless hooks — aren't academic: they're how every serious design system (Radix, Headless UI, MUI, shadcn) is built, and *"design a reusable X component"* is a standard interview question.

**Time:** ~65 minutes · **Prereq:** Lessons 02, 09

---

## 1. The idea in one sentence

> **A component should let consumers arrange its parts rather than configure its behaviour — because arrangement is infinitely flexible and configuration always runs out.**

---

## 2. The configuration trap, and how to recognise it early

```tsx
// Month 1
<Modal title="Refund" body="Are you sure?" onConfirm={fn} />

// Month 3 — "we need an icon"
<Modal title="Refund" icon="warning" ... />

// Month 5 — "the footer needs a checkbox"
<Modal ... footerContent={<Checkbox />} showFooterDivider />

// Month 9
<Modal
  title body icon iconColor size showClose closeOnOverlay closeOnEscape
  footerContent footerAlign headerAlign bodyPadding maxHeight scrollable
  onConfirm onCancel confirmLabel cancelLabel confirmVariant loading disabled ... />
```

**The diagnostic:** every new requirement adds a prop, and props are never removed. A component with 20 props has usually absorbed the *layout decisions* of its consumers — which is precisely the work it shouldn't be doing.

```tsx
// ✅ Composition: the consumer arranges, the component coordinates
<Modal open={open} onClose={close}>
  <Modal.Header icon={<WarningIcon />}>Refund payment</Modal.Header>
  <Modal.Body>
    <p>This refunds {formatMoney(payment.money)} to the customer.</p>
    <Checkbox checked={notify} onChange={setNotify}>Email the customer</Checkbox>
  </Modal.Body>
  <Modal.Footer>
    <Button variant="ghost" onClick={close}>Cancel</Button>
    <Button variant="danger" onClick={refund} loading={isPending}>Refund</Button>
  </Modal.Footer>
</Modal>
```

**The test to apply before adding any prop:** *"could the consumer express this by arranging children instead?"* If yes, don't add the prop.

**And the honest counter-case:** configuration is right when the option set is genuinely closed and the component's job is to *constrain* — `<Button variant="primary" size="md">` should not let consumers compose arbitrary button internals. Design-system primitives configure; layout containers compose.

---

## 3. Pattern 1 — Compound components

Related components sharing implicit state through context.

```tsx
type TabsContextValue = { value: string; setValue: (v: string) => void; baseId: string };
const TabsContext = createContext<TabsContextValue | null>(null);

function useTabs(component: string) {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error(`<${component}> must be used within <Tabs>`);
  return ctx;
}

export function Tabs({ defaultValue, value: controlled, onValueChange, children }: TabsProps) {
  const [uncontrolled, setUncontrolled] = useState(defaultValue);
  const value = controlled ?? uncontrolled;                      // controlled OR uncontrolled (§6)
  const baseId = useId();

  const setValue = useCallback((v: string) => {
    if (controlled === undefined) setUncontrolled(v);
    onValueChange?.(v);
  }, [controlled, onValueChange]);

  const ctx = useMemo(() => ({ value, setValue, baseId }), [value, setValue, baseId]);
  return <TabsContext value={ctx}>{children}</TabsContext>;
}

Tabs.List = function TabsList({ children }: { children: React.ReactNode }) {
  return <div role="tablist">{children}</div>;
};

Tabs.Trigger = function TabsTrigger({ value, children }: TriggerProps) {
  const { value: active, setValue, baseId } = useTabs("Tabs.Trigger");
  const selected = active === value;
  return (
    <button
      role="tab"
      id={`${baseId}-trigger-${value}`}
      aria-selected={selected}
      aria-controls={`${baseId}-panel-${value}`}
      tabIndex={selected ? 0 : -1}                     // roving tabindex — a11y
      onClick={() => setValue(value)}
    >{children}</button>
  );
};

Tabs.Content = function TabsContent({ value, children }: ContentProps) {
  const { value: active, baseId } = useTabs("Tabs.Content");
  if (active !== value) return null;
  return (
    <div role="tabpanel" id={`${baseId}-panel-${value}`}
         aria-labelledby={`${baseId}-trigger-${value}`} tabIndex={0}>
      {children}
    </div>
  );
};
```

**What this buys:**
- Consumers control layout completely — wrap triggers in a scroll container, add a divider, put content anywhere
- No prop drilling; the context is scoped to the widget
- Each part handles its own ARIA wiring, using one `useId` for the pairing
- A clear error if a part is used outside its parent

**The trade-offs, honestly:** the parts are coupled (a `Tabs.Trigger` is useless alone), it's more code than a configured component, and the API surface is less discoverable — you can't see the options in one type. **Worth it for widgets with layout variation; overkill for a badge.**

> **Interview note:** don't over-claim `Tabs.Trigger = ...` as "namespacing." Attaching subcomponents is convenient for discoverability and import ergonomics, but named exports (`export { Tabs, TabsTrigger }`) tree-shake better. Radix uses named exports for that reason. Either is defensible; knowing *why* is the signal.

---

## 4. Pattern 2 — Slots

When you need specific, named regions rather than free-form children.

```tsx
type PageProps = {
  title: React.ReactNode;
  actions?: React.ReactNode;
  breadcrumbs?: React.ReactNode;
  children: React.ReactNode;
};

function Page({ title, actions, breadcrumbs, children }: PageProps) {
  return (
    <div className="page">
      {breadcrumbs && <nav aria-label="Breadcrumb">{breadcrumbs}</nav>}
      <header>
        <h1>{title}</h1>
        {actions && <div className="page__actions">{actions}</div>}
      </header>
      <main>{children}</main>
    </div>
  );
}

<Page
  title="Payments"
  breadcrumbs={<Breadcrumbs items={[...]} />}
  actions={<><Button variant="ghost">Export</Button><Button>New payment</Button></>}
>
  <PaymentsTable />
</Page>
```

**Slots vs compound components:**

| | Slots | Compound |
|---|---|---|
| Regions | Fixed and named | Arbitrary arrangement |
| Shared state | None needed | Context |
| Discoverability | **High** — the props type lists them | Lower |
| Flexibility | Bounded | High |
| Best for | Layouts, page shells, cards | Interactive widgets |

**Slots are underrated.** For a page layout, `title`/`actions`/`children` as props is clearer than `<Page.Title>`, and the type tells you exactly what regions exist. **Use slots when the structure is fixed; compound when the structure varies.**

---

## 5. Pattern 3 — Render props and headless components

Share *behaviour* while letting the consumer own *rendering*.

```tsx
// The modern form: a headless HOOK rather than a render-prop component
export function useDisclosure(initial = false) {
  const [isOpen, setIsOpen] = useState(initial);
  return {
    isOpen,
    open: useCallback(() => setIsOpen(true), []),
    close: useCallback(() => setIsOpen(false), []),
    toggle: useCallback(() => setIsOpen(o => !o), []),
    triggerProps: useMemo(() => ({ "aria-expanded": isOpen, onClick: () => setIsOpen(o => !o) }), [isOpen]),
  };
}

// Consumers render whatever they like
const { isOpen, triggerProps, close } = useDisclosure();
<button {...triggerProps}>Filters</button>
{isOpen && <FilterPanel onClose={close} />}
```

**Hooks replaced most render props.** A hook gives the same behaviour-sharing with no extra component in the tree, no nesting, and better composability (you can call three hooks; you can't easily nest three render props without a pyramid).

**Where render props still win** — when the behaviour needs to *own* a DOM element or a lifecycle position:

```tsx
// Virtualization genuinely needs to control rendering
<Virtualizer count={rows.length} estimateSize={() => 48}>
  {(virtualItem) => <PaymentRow payment={rows[virtualItem.index]!} />}
</Virtualizer>

// So does anything measuring its own children
<Measure>{({ ref, width }) => <div ref={ref}>{width > 600 ? <Wide/> : <Narrow/>}</div>}</Measure>
```

**And the "headless component library" pattern** (Radix, Headless UI) combines both: unstyled components that handle behaviour, ARIA, focus management and keyboard interaction, leaving all visual decisions to you. **That's the right default for a design system in 2026** — building an accessible combobox or dialog from scratch is weeks of work you shouldn't repeat.

---

## 6. Controlled vs uncontrolled — the API decision

A reusable component should support both, and the pattern is the same every time.

```tsx
function useControllableState<T>({ value, defaultValue, onChange }: {
  value?: T; defaultValue: T; onChange?: (v: T) => void;
}) {
  const [uncontrolled, setUncontrolled] = useState(defaultValue);
  const isControlled = value !== undefined;
  const current = isControlled ? value : uncontrolled;

  const setValue = useCallback((next: T) => {
    if (!isControlled) setUncontrolled(next);
    onChange?.(next);                       // ← fires in BOTH modes
  }, [isControlled, onChange]);

  return [current, setValue] as const;
}
```

**The rules that make it correct:**
- `value !== undefined` decides the mode — **not** `value != null`, because `null` can be a legitimate controlled value.
- `onChange` fires in both modes, so consumers can observe without controlling.
- A component must not **switch modes** mid-life. React warns about this for inputs; do the same:
  ```tsx
  const wasControlled = useRef(isControlled);
  if (process.env.NODE_ENV !== "production" && wasControlled.current !== isControlled) {
    console.error("Component is changing from controlled to uncontrolled (or vice versa).");
  }
  ```

> **The interview framing:** *"uncontrolled by default with an optional `value`/`onChange` — so simple cases need no state and complex cases get full control. That's how every good component library does it, and the subtle part is that `onChange` must fire in both modes."*

---

## 7. Pattern 4 — Polymorphic components (`as`)

```tsx
type PolymorphicProps<E extends React.ElementType, P> =
  P & { as?: E } & Omit<React.ComponentPropsWithoutRef<E>, keyof P | "as">;

function Text<E extends React.ElementType = "span">(
  { as, size = "md", ...rest }: PolymorphicProps<E, { size?: "sm" | "md" | "lg" }>
) {
  const Component = as ?? "span";
  return <Component data-size={size} {...rest} />;
}

<Text>plain</Text>
<Text as="h1" size="lg">Heading</Text>
<Text as="a" href="/payments">Link</Text>       // ✅ href is typed because it's an anchor
<Text as="button" href="/x" />                   // ❌ compile error
```

Genuinely valuable in a design system — one `Text` component serving `<p>`, `<span>`, `<h1>` and `<a>` with correct semantics and correct types. **Real cost:** the types are gnarly, error messages get worse, and the compiler works harder. **Build it once in a shared package; don't reach for it in feature code** ([TS Lesson 17](../../TypeScript/05-ecosystem/17-typescript-with-react.md)).

A lighter alternative worth knowing — Radix's `asChild`:
```tsx
<Button asChild><Link to="/payments">Payments</Link></Button>
// Button clones its child and merges props onto it instead of rendering its own element
```
Simpler types, and it composes better with routing libraries. The trade is that it requires exactly one valid element child.

---

## 8. Anti-patterns

### Anti-pattern 1 — Props that are really components
```tsx
// ❌ You've invented a worse JSX
<Table columns={[{ header: "Amount", renderCell: (r) => ..., width: 120, align: "right" }]} />
// ✅
<Table><Table.Column header="Amount" align="right" width={120} cell={r => ...} /></Table>
```
Though note: **the config-array form is legitimate** when columns are dynamic (user-configurable, or generated from a schema). Judge by whether the consumer writes them literally or computes them.

### Anti-pattern 2 — Boolean explosion
```tsx
<Button primary large outline loading disabled fullWidth rounded />   // 2⁷ combinations
<Button variant="outline" size="lg" loading fullWidth />               // ✅
```

### Anti-pattern 3 — Leaky abstractions
```tsx
// ❌ Implementation in the API
<DataTable useVirtualization virtualizationOverscan={5} memoizeRows />
// ✅ Those are the component's decisions, not the consumer's
<DataTable rows={payments} />
```

### Anti-pattern 4 — HOCs where a hook would do
```tsx
// ❌ Wrapper hell, lost types, mystery props
export default withAuth(withTheme(withRouter(Component)));
// ✅
const { principal } = useAuth();
```
HOCs still have niches — error boundaries (which must be class components), and framework-level wrappers like `React.memo` or `forwardRef`. For logic sharing, hooks won.

### Anti-pattern 5 — `cloneElement` for implicit props
```tsx
// ❌ Fragile, invisible, breaks with fragments and conditional children
React.Children.map(children, child => cloneElement(child, { selected: true }));
// ✅ Context, so children opt in explicitly
```
`cloneElement` is deprecated in React 19's docs for good reason: it silently fails when children are fragments, arrays, strings or conditionals, and consumers can't see where the prop came from.

---

## 9. Ledger Console's component layer

```
components/
├── ui/                    # primitives — configured, closed option sets
│   ├── button.tsx         # variant, size, loading
│   ├── badge.tsx
│   ├── input.tsx
│   └── skeleton.tsx
├── patterns/              # compound + slots
│   ├── modal.tsx          # Modal.Header / Body / Footer
│   ├── data-table.tsx     # DataTable.Column
│   ├── page.tsx           # slots: title, actions, breadcrumbs
│   └── async-boundary.tsx # one `state` prop, exhaustive switch
├── features/              # domain components — composed from the above
│   ├── payments/
│   │   ├── payments-table.tsx
│   │   ├── payment-detail.tsx
│   │   └── refund-modal.tsx
│   └── webhooks/
└── hooks/
```

```tsx
// The shape a feature component ends up as — thin, composed, readable
export function RefundModal({ payment, open, onClose }: RefundModalProps) {
  const [state, dispatch] = useReducer(refundReducer, { step: "closed" });   // Lesson 06
  const refund = useRefundPayment();                                          // Lesson 18

  return (
    <Modal open={open} onClose={onClose} size="md">
      <Modal.Header icon={<WarningIcon />}>Refund {payment.id}</Modal.Header>
      <Modal.Body>
        {state.step === "amount" && <AmountStep ... />}
        {state.step === "confirm" && <ConfirmStep ... />}
        {state.step === "failed" && <Alert variant="error">{state.error.message}</Alert>}
      </Modal.Body>
      <Modal.Footer>
        <Button variant="ghost" onClick={onClose}>Cancel</Button>
        <Button variant="danger" loading={state.step === "submitting"}
                onClick={() => dispatch({ type: "submit" })}>Refund</Button>
      </Modal.Footer>
    </Modal>
  );
}
```

**The three-layer split is the architecture point:** primitives are constrained (configuration), patterns are flexible (composition), and features are thin compositions of both. Feature components should be *arrangement plus domain logic* — if one is 300 lines of layout, a pattern is missing.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Before adding a prop, ask if children could express it** | Configuration always runs out; arrangement doesn't |
| **Compound components for widgets with layout variation; slots for fixed structure** | Slots are more discoverable; compound is more flexible |
| **Configuration for primitives with closed option sets** | Their job is to constrain |
| **Support controlled *and* uncontrolled; `onChange` fires in both** | Simple cases need no state; complex cases get control |
| **Never switch controlled/uncontrolled mid-life — warn in dev** | React warns for inputs; yours should too |
| **Headless hooks over render props** | Same behaviour sharing, no extra tree node, composes better |
| **Use a headless library (Radix/Headless UI) for dialogs, comboboxes, menus** | Accessible focus management is weeks of work |
| **Every compound part throws if used outside its parent** | Clear error beats a silent null |
| **`useId` once per widget; derive ARIA id suffixes** | Server/client stable, and correct pairing |
| **Avoid `cloneElement`; use context** | Silently breaks on fragments, arrays and conditionals |
| **HOCs only for error boundaries and framework wrappers** | Hooks won for logic sharing |

---

## 11. Interview traps

**Q1. "Design a reusable Modal."**
Lead with composition: `Modal` + `Header`/`Body`/`Footer`, context for `onClose`, `open` as a controlled prop. Then volunteer what a good answer includes and a mediocre one doesn't: **focus trap, focus restore on close, `Escape` to close, `aria-modal` + `role="dialog"` + `aria-labelledby`, scroll lock with restore, and a portal so it escapes `overflow: hidden`.** Finish with *"in practice I'd build on Radix Dialog rather than reimplement focus management."*

**Q2. "Compound components vs slots vs render props?"**
Compound: context-shared state, arbitrary arrangement — interactive widgets. Slots: named regions as props, fixed structure, **more discoverable** — layouts and cards. Render props: consumer controls rendering while the component owns behaviour — now mostly replaced by headless hooks, except when the behaviour must own a DOM node (virtualization, measurement).

**Q3. "Your Button has 15 props. What do you do?"**
Diagnose first: how many are derivable, booleans that should be a union, layout options that should be children, or modes that should be separate components? A `Button` should be configured (closed option set); if it has layout props, those belong to the consumer.

**Q4. "Controlled or uncontrolled?"**
Support both. `value !== undefined` selects the mode (not `!= null`, since `null` can be a real value), `onChange` fires in both, and never switch modes mid-life. That's how every good library does it.

**Q5. "Why did hooks replace HOCs and render props?"**
No wrapper components in the tree, no prop-name collisions, no lost types, no pyramid nesting, and they compose linearly — you can call three hooks as easily as one. HOCs survive for error boundaries (class-only) and framework wrappers.

**Q6. "When is `cloneElement` wrong?"**
Almost always. It breaks on fragments, arrays, strings and conditional children; the injected props are invisible at the call site; and it couples the parent to the child's prop names. Use context so children opt in.

**Q7. "How do you make a component polymorphic?"**
A generic `as` prop with `ComponentPropsWithoutRef<E>` merged in, or Radix's `asChild` which clones a single element child. Worth it in a design system, not in feature code — the types get gnarly and error messages degrade.

**Q8. "How do you handle accessibility in a compound component?"**
One `useId` per widget instance, derived suffixes for each part's id, correct roles (`tablist`/`tab`/`tabpanel`), `aria-selected`/`aria-controls`/`aria-labelledby` wiring across parts, **roving tabindex** for keyboard navigation, and arrow-key handling. **Mentioning roving tabindex unprompted is a strong signal** — it's the part people forget.

**Q9. "Subcomponents as properties (`Tabs.Trigger`) or named exports?"**
Properties are discoverable and read nicely; named exports tree-shake better (Radix chose them for that reason). Either is fine — knowing the trade-off is the point.

**Q10. "How do you structure a component library?"**
Three layers: primitives (configured, closed options), patterns (composed — modal, table, page shell), features (thin arrangements plus domain logic). If a feature component is 300 lines of layout, a pattern is missing.

---

## 12. Build & break

### Build — the Tabs component, completely
Implement §3 with: controlled/uncontrolled support via `useControllableState`, full ARIA wiring from one `useId`, roving tabindex, arrow-key navigation (Left/Right, Home/End), and errors when parts are used outside `<Tabs>`.

Then test it with a screen reader (VoiceOver on macOS: ⌘F5; NVDA on Windows). **Most React developers have never done this, and it changes how you build components permanently.**

### Build — refactor the configuration monster
```tsx
<DataTable
  data={payments} columns={["id","amount","status"]} columnLabels={{...}}
  columnWidths={{...}} columnAlign={{...}} renderCell={{...}} sortable={["amount"]}
  onSort={fn} selectable selectedIds={ids} onSelectionChange={fn}
  emptyMessage="No payments" loadingRows={10} isLoading={false} error={null}
  onRowClick={fn} rowClassName={fn} stickyHeader virtualized maxHeight={600}
/>
```
Produce a composed version with `DataTable.Column`, an `AsyncBoundary` for the loading/error/empty states, and the virtualization decision moved inside. Count the props you removed.

### Build — the three-layer structure
Set up Ledger Console's `ui/`, `patterns/`, `features/` folders with at least two components in each. Write a one-paragraph `README.md` in each explaining what belongs there — that document is what keeps the structure alive after you.

### Break — four experiments
1. **Compound part outside its parent.** Render `<Tabs.Trigger>` alone. With the throwing hook you get a clear message; without it, a confusing `null` crash.
2. **Mode switching.** Render an `<Input>` with `value={undefined}` then `value="x"`. Read React's controlled/uncontrolled warning, then add the same warning to your own component.
3. **`cloneElement` fragility.** Build a component that clones children to inject a prop, then pass it a fragment, then a conditional, then a string. Watch it fail three different ways.
4. **Missing focus trap.** Open a modal without one, press Tab repeatedly, and watch focus escape to the page behind it. Then add `inert` on the background or a focus trap and repeat.

### Explain out loud (60 seconds)
1. The configuration trap, and the test before adding a prop.
2. Compound vs slots vs render props, with a use case each.
3. The controlled/uncontrolled pattern and its two subtleties.
4. What a good Modal includes beyond rendering.

---

## What's next

Forms are where composition, state, validation and accessibility all meet — and where React's newest APIs (Actions, `useActionState`, `useOptimistic`) changed the idiomatic answer. Next: controlled vs uncontrolled at scale, validation, and large-form performance.

Next → **[Lesson 14: Forms, properly](14-forms.md)**
