# Lesson 14 — Forms, properly

> **Why this lesson exists:** forms are where everything meets — state, validation, accessibility, async submission, error handling, and performance. They're also where React's story changed most in v19, with Actions and `useActionState` making the idiomatic answer genuinely different from what most tutorials teach. And *"build a form with validation"* is one of the most common machine-coding round prompts.

**Time:** ~70 minutes · **Prereq:** Lessons 05, 13 · [TS Lesson 15](../../TypeScript/04-real-code/15-the-boundary-and-parsing.md)

---

## 1. The idea in one sentence

> **A form is a boundary — untrusted strings in, validated domain values out — so the schema that validates it should also be the type that describes it.**

That's "parse, don't validate" ([TS Lesson 15](../../TypeScript/04-real-code/15-the-boundary-and-parsing.md)) applied to the UI, and it's why schema-driven forms won.

---

## 2. Controlled vs uncontrolled

```tsx
// Controlled — React owns the value
const [email, setEmail] = useState("");
<input value={email} onChange={e => setEmail(e.target.value)} />

// Uncontrolled — the DOM owns the value; React reads it when needed
const ref = useRef<HTMLInputElement>(null);
<input ref={ref} defaultValue="" />
// …later: ref.current?.value

// Uncontrolled via the form itself — no refs at all
<form onSubmit={e => {
  e.preventDefault();
  const data = Object.fromEntries(new FormData(e.currentTarget));
}}>
  <input name="email" defaultValue="" />
</form>
```

| | Controlled | Uncontrolled |
|---|---|---|
| Re-renders per keystroke | **Yes** | No |
| Instant validation / formatting | ✅ | Awkward |
| Conditional fields based on values | ✅ | Awkward |
| Performance on a 50-field form | Needs care | **Free** |
| Works with React 19 Actions | Yes | **Natively** |

**The received wisdom — "always use controlled inputs" — is wrong**, and the pendulum has swung back. `FormData` is a perfectly good state container for most forms, and it's what React 19's Actions are built around.

> **The decision rule:** *"controlled when you need the value **during** typing — live validation, character counts, dependent fields, formatting as you type. Uncontrolled when you only need it at submit. Most forms only need it at submit."*

**The hybrid that most real forms end up as:** uncontrolled by default, controlled for the two or three fields that genuinely need live behaviour.

---

## 3. React 19 Actions — the new default

```tsx
function RefundForm({ payment }: { payment: Refundable }) {
  const [state, formAction, isPending] = useActionState(
    async (_prev: FormState, formData: FormData): Promise<FormState> => {
      const parsed = RefundSchema.safeParse(Object.fromEntries(formData));
      if (!parsed.success) return { status: "invalid", errors: fieldErrors(parsed.error) };

      const result = await api.createRefund(payment.id, parsed.data, idempotencyKey);
      return result.ok
        ? { status: "success", refund: result.value }
        : { status: "failed", error: result.error };
    },
    { status: "idle" },
  );

  return (
    <form action={formAction}>
      <Field label="Amount" name="amount" defaultValue={payment.money.amountMinor / 100}
             error={state.status === "invalid" ? state.errors.amount : undefined} />
      <Field label="Reason" name="reason" as="select" options={REASONS} />
      <button disabled={isPending}>{isPending ? "Refunding…" : "Refund"}</button>
      {state.status === "failed" && <Alert variant="error">{state.error.message}</Alert>}
    </form>
  );
}
```

**What `useActionState` gives you for free:**
- `isPending` — no manual `isSubmitting` state to forget to reset
- Automatic form reset on success (for uncontrolled inputs)
- The action runs in a transition, so the UI stays responsive
- The same code works with Server Actions in Next.js ([Lesson 24](../07-server-react/24-nextjs-app-router.md))
- **It works without JavaScript** when used with Server Actions — genuine progressive enhancement

```tsx
// useFormStatus — a child reads the enclosing form's pending state, no prop drilling
function SubmitButton({ children }: { children: React.ReactNode }) {
  const { pending } = useFormStatus();       // ← reads the PARENT <form>
  return <button disabled={pending}>{pending ? "Saving…" : children}</button>;
}
// Must be rendered INSIDE the <form> — it reads context from the form element.
```

```tsx
// useOptimistic — show the result before the server confirms (Lesson 18)
const [optimisticRefunds, addOptimistic] = useOptimistic(
  refunds,
  (current, pending: Refund) => [...current, { ...pending, status: "pending" as const }],
);

async function submit(formData: FormData) {
  const refund = buildRefund(formData);
  addOptimistic(refund);                    // instant UI
  await api.createRefund(...);               // React reverts automatically if this throws
}
```
**`useOptimistic` automatically reverts** when the action completes and the real state arrives — you don't write rollback logic. That's genuinely new and worth knowing.

---

## 4. Validation: schema-driven

```tsx
const RefundSchema = z.object({
  amount: z.string()
    .regex(/^\d+(\.\d{1,2})?$/, "Enter a valid amount")
    .transform(s => Math.round(parseFloat(s) * 100))         // string → minor units (API L05)
    .refine(n => n >= 50, "Minimum refund is 0.50"),
  reason: z.enum(["requested_by_customer", "duplicate", "fraudulent"]),
  notify: z.coerce.boolean().default(false),                  // checkboxes give "on" | undefined
});

type FormInput = z.input<typeof RefundSchema>;      // { amount: string; ... }   ← the <input> shape
type FormOutput = z.output<typeof RefundSchema>;    // { amount: number; ... }   ← the API body
```

**The `z.input`/`z.output` split is the thing to internalise.** The DOM gives you strings; your API wants integers in minor units; the transformation and its validation belong in **one declaration**. Without it you write the conversion twice and they drift.

### Where validation runs
```
1. HTML attributes      required, type="email", min, max, pattern   ← free, works without JS
2. Client-side schema    instant feedback, good UX
3. Server-side schema    ← THE ONLY ONE THAT'S SECURITY
```
**Client validation is UX; server validation is security.** Anyone can `curl` your endpoint. Never skip #3 ([API Lesson 16](../../API/03-security/16-owasp-and-hardening.md)).

### When to validate — the UX rule
```tsx
// ❌ Validating on every keystroke from the start: errors appear while you're still typing
// ✅ The standard, and it's what users expect:
//    - validate on BLUR for the first time
//    - then validate on CHANGE once the field has been touched
//    - validate everything on SUBMIT
//    - clear the error as soon as it becomes valid
```
Showing "invalid email" after the user has typed `a` is hostile. **Validate late, clear early.**

### With React Hook Form
```tsx
const { register, handleSubmit, formState: { errors, isSubmitting } } =
  useForm<FormInput, unknown, FormOutput>({
    resolver: zodResolver(RefundSchema),
    mode: "onTouched",                       // ← validate on blur, then on change
  });

const onSubmit = handleSubmit(async data => {     // data is FormOutput — already transformed
  const result = await api.createRefund(payment.id, data, key);
  if (!result.ok) setError("root", { message: result.error.message });
});
```
RHF keeps inputs **uncontrolled** (registering via refs), so typing doesn't re-render the form — which is why it's the default choice for large forms. The three-parameter `useForm<In, Ctx, Out>` is the piece people miss.

**RHF vs Actions:** Actions for simple forms and anything wanting progressive enhancement; RHF for complex client-side forms with field arrays, dependent fields, wizards, and per-field validation modes. They're not competitors so much as different scales.

---

## 5. Accessibility — the half that's usually missing

```tsx
function Field({ label, name, error, hint, required, ...rest }: FieldProps) {
  const id = useId();
  const errorId = `${id}-error`, hintId = `${id}-hint`;

  return (
    <div className="field">
      <label htmlFor={id}>
        {label} {required && <span aria-hidden="true">*</span>}
      </label>
      {hint && <p id={hintId} className="field__hint">{hint}</p>}
      <input
        id={id}
        name={name}
        required={required}
        aria-invalid={error ? true : undefined}
        aria-describedby={[hint && hintId, error && errorId].filter(Boolean).join(" ") || undefined}
        {...rest}
      />
      {error && <p id={errorId} role="alert" className="field__error">{error}</p>}
    </div>
  );
}
```

The checklist, and every item is commonly missed:
```
[ ] <label htmlFor> paired with input id (NOT placeholder-as-label)
[ ] aria-describedby linking hint AND error
[ ] aria-invalid when in error
[ ] role="alert" on the error so screen readers announce it
[ ] Errors are TEXT, not just a red border (colour alone fails WCAG)
[ ] A submit-time error summary at the top, focusable, linking to each field
[ ] Focus moves to the first invalid field (or the summary) on failed submit
[ ] type="email" / inputMode="numeric" / autoComplete="cc-number" for real keyboards and autofill
[ ] The form is submittable with Enter
[ ] Disabled submit buttons announce WHY (aria-describedby), or aren't disabled at all
```

```tsx
// Focus management on failed submit — the detail that separates good from adequate
const onSubmit = handleSubmit(onValid, (fieldErrors) => {
  const first = Object.keys(fieldErrors)[0];
  if (first) document.getElementsByName(first)[0]?.focus();
});
```

> **`placeholder` is not a label.** It disappears on focus, fails contrast requirements, and is invisible to some screen readers. It's a *hint*, and the most common accessibility failure in forms.

---

## 6. Performance on large forms

```tsx
// ❌ 50 controlled fields in one component: every keystroke re-renders all 50
function BigForm() {
  const [values, setValues] = useState(initial);
  return FIELDS.map(f => (
    <input key={f.name} value={values[f.name]}
           onChange={e => setValues(v => ({ ...v, [f.name]: e.target.value }))} />
  ));
}
```

Three fixes, in order of preference:
1. **Go uncontrolled** — RHF or `FormData`. Typing causes zero re-renders. *This is the real fix.*
2. **Isolate each field** into its own component owning its own state ([Lesson 03](../01-mental-model/03-rendering-and-commit.md)).
3. **Debounce** what the value *drives* (a preview, a search), not the input itself.

```tsx
// Field arrays — dynamic lists of inputs
const { fields, append, remove } = useFieldArray({ control, name: "lineItems" });
{fields.map((field, i) => (
  <div key={field.id}>        {/* ← field.id, NOT the index (Lesson 04) */}
    <input {...register(`lineItems.${i}.description`)} />
    <button type="button" onClick={() => remove(i)}>Remove</button>
  </div>
))}
```
**`key={field.id}` not `key={i}`** — removing a middle row with index keys moves every subsequent row's value up by one. That's the index-key bug, in the place it does the most visible damage.

---

## 7. The details that make forms feel professional

```tsx
// 1. Prevent double submit — three layers, because each can fail alone
<button disabled={isPending}>                        // UI affordance
// + the reducer has no `submit` in the submitting state   (Lesson 06)
// + an Idempotency-Key on the request                     (API Lesson 18)

// 2. Warn before losing unsaved work
useEffect(() => {
  if (!isDirty) return;
  const handler = (e: BeforeUnloadEvent) => { e.preventDefault(); };
  window.addEventListener("beforeunload", handler);
  return () => window.removeEventListener("beforeunload", handler);
}, [isDirty]);
// Plus a router-level block for client-side navigation, which beforeunload doesn't cover

// 3. Autofill and mobile keyboards — genuinely improves completion rates
<input name="email" type="email" autoComplete="email" inputMode="email" />
<input name="cardNumber" autoComplete="cc-number" inputMode="numeric" />
<input name="postcode" autoComplete="postal-code" />

// 4. Server errors mapped back to FIELDS, not just a banner
if (!result.ok && result.error.code === "validation_failed") {
  for (const e of result.error.errors) {
    setError(e.field as keyof FormInput, { message: e.detail });   // ← API Lesson 10's field paths
  }
}
```

**That last one is the payoff from the API track.** Your error contract returns `errors: [{ field, code, detail }]` with dotted paths — which means the client can highlight the exact input. Designing the API error shape and consuming it in the form is the same job, done at both ends.

---

## 8. Ledger Console's forms

| Form | Approach | Why |
|---|---|---|
| **Login** | Actions + `useActionState` | Simple, benefits from progressive enhancement |
| **Refund** | Reducer state machine + Actions | Multi-step with illegal transitions ([Lesson 06](../02-state-effects/06-reducers-and-state-machines.md)) |
| **Create payment** | RHF + Zod | Dependent fields (currency changes the amount's minor-unit exponent) |
| **Webhook endpoint** | RHF + Zod + field array | Dynamic list of subscribed events |
| **Search / filter bar** | Controlled + debounced → URL | Needs the value while typing ([Lesson 03](../01-mental-model/03-rendering-and-commit.md)) |
| **Settings** | Uncontrolled + `FormData` | 20 fields, submit-only |

```tsx
// The currency-dependent amount field — why "controlled when you need it while typing" matters
function AmountField({ currency }: { currency: Currency }) {
  const exponent = MINOR_UNITS[currency];            // usd → 2, jpy → 0
  const [display, setDisplay] = useState("");

  return (
    <Field
      label={`Amount (${currency.toUpperCase()})`}
      value={display}
      inputMode="decimal"
      onChange={e => setDisplay(formatAsCurrency(e.target.value, exponent))}  // live formatting
      hint={exponent === 0 ? "This currency has no decimal places" : undefined}
    />
  );
}
```

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Uncontrolled by default; controlled only for fields needing live behaviour** | Most forms only need values at submit |
| **One schema for validation *and* types (`z.input`/`z.output`)** | The string→domain conversion lives in one place |
| **Client validation is UX; server validation is security** | Anyone can `curl` your endpoint |
| **Validate on blur first, then on change once touched; clear early** | Errors while still typing are hostile |
| **`<label htmlFor>` always. `placeholder` is not a label** | The most common a11y failure in forms |
| **`aria-describedby` + `aria-invalid` + `role="alert"`** | Errors must be announced, not just red |
| **Move focus to the first invalid field on failed submit** | Otherwise keyboard users are stranded |
| **`key={field.id}` in field arrays, never the index** | Removing a row shifts every value up |
| **Map server field errors back to inputs** | Your API returns field paths — use them |
| **Prevent double submit at three layers: UI, state machine, idempotency key** | Each can fail independently |
| **`autoComplete` and `inputMode` on every field** | Measurably improves completion, especially on mobile |
| **Warn on unsaved changes — `beforeunload` *and* a router block** | `beforeunload` doesn't cover client-side navigation |

---

## 10. Interview traps

**Q1. "Controlled or uncontrolled?"**
Both, by field. Controlled when you need the value **during** typing — live validation, character counts, dependent fields, formatting. Uncontrolled when you only need it at submit, which is most fields. **Correct the received wisdom explicitly:** "always controlled" is out of date, and React 19's Actions are built around `FormData`.

**Q2. "How do you validate a form?"**
A schema (Zod) as the single source of truth, with `z.input` for the form state and `z.output` for the submit payload so the string→domain transform lives in one place. HTML attributes for free baseline validation, client schema for UX, **server schema for security.** Validate on blur then on change once touched.

**Q3. "How do you handle a 50-field form that lags?"**
Go uncontrolled (RHF or `FormData`) so typing causes no re-renders — that's the actual fix. If it must be controlled, isolate each field into its own component so state is local. Debounce what the value *drives*, not the input.

**Q4. "What's `useActionState`?"**
React 19's form-action hook: takes an async action and initial state, returns `[state, formAction, isPending]`. Wires to `<form action={formAction}>`, gives you pending state for free, runs in a transition, resets uncontrolled inputs on success, and **works without JavaScript** when paired with Server Actions.

**Q5. "What's `useFormStatus` and where must it be used?"**
It reads the enclosing `<form>`'s pending state, so a `SubmitButton` component knows it's submitting without prop drilling. **It must be rendered inside the `<form>`** — it reads from the form element's context, so calling it in the component that renders the form doesn't work.

**Q6. "List the accessibility requirements for a form field."**
`label htmlFor`/`id`, `aria-describedby` for hint and error, `aria-invalid`, `role="alert"` on the error, text errors not just colour, focus moved to the first invalid field on submit, `autoComplete`, and `inputMode`. **And placeholder is not a label** — it disappears on focus and fails contrast.

**Q7. "How do you prevent a double submit?"**
Three layers, and say why each alone is insufficient: a disabled button (defeatable by keyboard/race/second code path), a state machine with no `submit` transition from `submitting` (structural), and an **`Idempotency-Key` on the request** (because the network can duplicate it even if your UI can't). **The third layer is the one that shows cross-stack thinking.**

**Q8. "Server returns field-level validation errors. What do you do with them?"**
Map them back to the inputs using the field paths from the error contract — `setError(e.field, { message: e.detail })` — so the user sees the error on the field rather than a generic banner. That's why the API error format includes a `field` per error.

**Q9. "Why `key={field.id}` in a field array?"**
Index keys mean removing a middle row shifts every subsequent row's *value* up by one, because React reuses the fiber at each position. The most visible version of the index-key bug ([Lesson 04](../01-mental-model/04-keys-and-identity.md)).

**Q10. "React Hook Form or React 19 Actions?"**
Different scales. Actions for simple forms and anywhere progressive enhancement matters (they work without JS with Server Actions). RHF for complex client-side forms — field arrays, dependent fields, wizards, per-field validation modes, and 50-field performance. Not mutually exclusive.

---

## 11. Build & break

### Build — the refund form, three ways
Implement the same form with: (1) plain controlled state, (2) React Hook Form + Zod, (3) React 19 Actions + `useActionState`. For each, record: re-renders per keystroke (Profiler), lines of code, and whether it works with JavaScript disabled.

**That comparison table is the lesson**, and it's a strong thing to be able to quote.

### Build — the accessible `Field` component
Implement §5 completely, then verify:
1. Tab through with the keyboard only — can you complete the form?
2. Turn on a screen reader (VoiceOver ⌘F5 / NVDA) and submit with errors. **Is the error announced?**
3. Zoom to 200% — does the layout survive?
4. Check error contrast with DevTools' accessibility panel.

**Most React developers have never done step 2.** Do it once and your forms change permanently.

### Build — server error mapping
Wire Ledger's `422 validation_failed` response (with `errors: [{field, code, detail}]`) into RHF's `setError`, so a server-side rejection highlights the exact input. Then break the client validation deliberately and confirm the server's errors still land correctly.

### Break — five experiments
1. **50 controlled fields.** Build it, type, and watch the Profiler. Convert to RHF and compare.
2. **Index keys in a field array.** Add three rows with values, remove the middle one, and watch the last row's value jump up.
3. **Placeholder as label.** Build a form using only placeholders, then tab through it with a screen reader. The fields are unlabelled.
4. **Missing `role="alert"`.** Submit an invalid form with a screen reader on. Nothing is announced.
5. **Double submit.** Remove the disabled state and click submit rapidly with a slow network (throttle to 3G). Count the requests. Then add the idempotency key and count the *refunds*.

### Explain out loud (90 seconds)
1. Controlled vs uncontrolled, and the modern default.
2. Where validation runs, and which layer is security.
3. Five accessibility requirements for a field.
4. Three layers of double-submit prevention.
5. What `useActionState` gives you for free.

---

## What's next

Forms fail. Requests fail. Components throw. Next: the boundaries that contain failure and manage loading — error boundaries, Suspense, and portals — so a broken widget doesn't take down the page.

Next → **[Lesson 15: Error boundaries, Suspense & portals](15-boundaries-suspense-portals.md)**
