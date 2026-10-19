# Lesson 28 — Testing React

> **Why this lesson exists:** most React test suites are expensive to maintain and catch little, because they test *implementation* — that a hook returned a value, that a component called a prop — rather than *behaviour*. The result is tests that break on every refactor and pass while the app is broken. This lesson is about writing the other kind, and it's also a reliable interview differentiator: how someone tests reveals what they've been burned by.

**Time:** ~65 minutes · **Prereq:** Modules 1–7

---

## 1. The idea in one sentence

> **Test what a user can do, not how the component does it — because behaviour is the contract and implementation is the thing you want to be free to change.**

Kent C. Dodds' framing, and it's right: *"the more your tests resemble the way your software is used, the more confidence they give you."*

---

## 2. The testing shape for a React app

```
        ╱  E2E  ╲           few    Playwright — real browser, real flows
      ╱───────────╲
    ╱  Component   ╲       MANY   ★ RTL + MSW — the valuable layer
  ╱─────────────────╲
╲   Unit (pure fns)  ╱     many   reducers, formatters, schemas — fast and precise
  ╲─────────────────╱
```

**The React-specific correction:** the valuable layer is **component tests**, not unit tests. Most React bugs live in the seams — wiring, state, effects, conditional rendering, the interaction between a form and its validation — and a unit test of a hook proves none of that.

| Layer | Test | Don't test |
|---|---|---|
| **Unit** | Reducers, formatters, schemas, pure domain logic | Anything needing a render |
| **Component** | What the user sees and does; a feature end-to-end with a mocked network | Internal state, hook return values |
| **E2E** | 3–5 critical journeys against a real stack | Everything else — slow and flaky |
| **Visual** | Chromatic/Percy if you have a design system | Layout in unit tests |

---

## 3. Testing Library: the principles

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

test("issuing a refund shows a confirmation", async () => {
  const user = userEvent.setup();
  render(<PaymentDetail paymentId={"pi_1" as PaymentId} />, { wrapper: TestProviders });

  await user.click(await screen.findByRole("button", { name: /refund/i }));
  await user.type(screen.getByLabelText(/amount/i), "49.99");
  await user.click(screen.getByRole("button", { name: /confirm refund/i }));

  expect(await screen.findByText(/refund issued/i)).toBeInTheDocument();
});
```

**Note what the test never mentions:** component names, props, state, hooks, CSS classes, or the fact that a reducer exists. It describes what a user does and sees — so you can rewrite the entire implementation and the test still passes.

### Query priority — and why it matters more than it looks

```
1. getByRole          ← ALWAYS prefer. It's how assistive tech sees the page
2. getByLabelText     ← form fields
3. getByPlaceholderText
4. getByText          ← non-interactive content
5. getByDisplayValue
6. getByAltText / getByTitle
7. getByTestId        ← LAST RESORT
```

**Querying by role tests accessibility as a side effect.** If `getByRole("button", { name: /refund/i })` fails, the element isn't a button or has no accessible name — which is a real bug for keyboard and screen-reader users, found for free by your test suite. **That's the strongest argument for the priority order**, and it's worth saying in an interview.

```tsx
// ❌ Tests implementation; breaks on any refactor; proves nothing about a11y
container.querySelector(".refund-btn");
screen.getByTestId("refund-button");

// ✅ Tests the contract
screen.getByRole("button", { name: /refund/i });
```

`data-testid` is legitimate for things with no accessible identity — a chart container, a virtualized viewport — but **each one is a small admission that the element isn't identifiable to users either.**

### `getBy` / `queryBy` / `findBy`
```tsx
screen.getByRole(...)        // throws if not found — asserting presence
screen.queryByRole(...)      // returns null — the ONLY way to assert absence
screen.findByRole(...)       // async, retries until timeout — anything after an await
expect(screen.queryByRole("alert")).not.toBeInTheDocument();   // ✅ absence
```

### `userEvent`, not `fireEvent`
```tsx
await user.click(button);          // ✅ pointerdown → mousedown → focus → pointerup → click
fireEvent.click(button);            // ❌ a single synthetic event
```
`userEvent` simulates the full browser sequence — focus changes, hover, key events, and it respects `disabled` and `pointer-events: none`. **`fireEvent` will happily "click" a disabled button**, which is precisely the bug you wanted to catch.

---

## 4. MSW: mock the network, not your modules

```tsx
// src/test/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("*/v1/payments/:id", ({ params }) =>
    HttpResponse.json(buildPayment({ id: params.id as string }))),

  http.post("*/v1/payments/:id/refunds", async ({ request }) => {
    // Assert the contract the API actually requires (API Lesson 18)
    if (!request.headers.get("idempotency-key"))
      return HttpResponse.json({ code: "idempotency_key_required" }, { status: 400 });
    return HttpResponse.json(buildRefund(), { status: 201 });
  }),
];

// src/test/setup.ts
const server = setupServer(...handlers);
beforeAll(() => server.listen({ onUnhandledRequest: "error" }));   // ← unmocked calls FAIL
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

**Why MSW beats mocking `fetch` or your API module:**
- Your real client code runs — including error mapping, schema parsing, retry and refresh logic
- The same handlers work in tests, Storybook, and local development
- `onUnhandledRequest: "error"` catches requests you didn't know you were making
- Changing the client's internals doesn't break the tests

```tsx
// Override per test for error paths — the ones people skip
test("shows the decline reason when the card is declined", async () => {
  server.use(http.post("*/v1/payments/:id/refunds", () =>
    HttpResponse.json({ code: "card_declined", detail: "Insufficient funds" }, { status: 402 })));
  // …
  expect(await screen.findByText(/insufficient funds/i)).toBeInTheDocument();
});
```

**Generate handlers from your OpenAPI spec** where you can — then a backend contract change breaks your tests rather than production ([API L25](https://github.com/Pawar-Pratik/API-mastery/blob/main/06-craft/25-openapi-contract-first.md)).

---

## 5. The test wrapper

```tsx
// src/test/utils.tsx — every component test uses this
export function TestProviders({ children }: { children: React.ReactNode }) {
  const qc = new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },   // ← no retries: a failure should fail FAST
      mutations: { retry: false },
    },
  });
  return (
    <QueryClientProvider client={qc}>
      <MemoryRouter>
        <AuthProvider initialPrincipal={testPrincipal}>{children}</AuthProvider>
      </MemoryRouter>
    </QueryClientProvider>
  );
}

export function renderApp(ui: React.ReactElement, { route = "/" } = {}) {
  window.history.pushState({}, "", route);
  return { user: userEvent.setup(), ...render(ui, { wrapper: TestProviders }) };
}
```

**`retry: false` is essential.** With the default retry policy, a test asserting an error state waits for three retries with backoff and times out — a classic source of "why is my test slow and flaky?"

**A fresh `QueryClient` per test** prevents cache leakage between tests, which otherwise produces tests that pass alone and fail in sequence.

---

## 6. Testing the hard parts

### Async state and loading
```tsx
test("shows a skeleton, then the payment", async () => {
  renderApp(<PaymentDetail paymentId={id} />);
  expect(screen.getByTestId("skeleton")).toBeInTheDocument();
  expect(await screen.findByText("pi_1")).toBeInTheDocument();
  expect(screen.queryByTestId("skeleton")).not.toBeInTheDocument();
});
```

### Reducers — as pure functions, no rendering at all
```tsx
test("cannot submit twice", () => {
  let s: RefundState = { step: "confirm", payment, amountMinor: 4999, reason: "duplicate" };
  s = refundReducer(s, { type: "submit" });
  expect(s.step).toBe("submitting");
  s = refundReducer(s, { type: "submit" });      // the second submit
  expect(s.step).toBe("submitting");              // ← unchanged. No refund duplicated.
});
```
**This is the payoff of the state machine** ([Lesson 06](../02-state-effects/06-reducers-and-state-machines.md)): the most important invariant in the app is tested in four lines with no React, no mocks and no async.

### Idempotency — asserting the *effect*, not the status
```tsx
test("double-clicking refund sends one request", async () => {
  const requests: Request[] = [];
  server.use(http.post("*/refunds", async ({ request }) => {
    requests.push(request);
    await delay(300);                                    // simulate a slow network
    return HttpResponse.json(buildRefund(), { status: 201 });
  }));

  const { user } = renderApp(<RefundModal payment={payment} open />);
  const btn = screen.getByRole("button", { name: /confirm refund/i });
  await user.click(btn);
  await user.click(btn);                                  // second click while pending

  await waitFor(() => expect(requests).toHaveLength(1));   // ← THE assertion
});
```

### Race conditions
```tsx
test("a slow response for an old query is discarded", async () => {
  server.use(http.get("*/v1/payments", async ({ request }) => {
    const q = new URL(request.url).searchParams.get("q");
    await delay(q === "a" ? 500 : 50);                    // "a" is slow, deliberately
    return HttpResponse.json({ data: [buildPayment({ description: `results for ${q}` })] });
  }));

  const { user } = renderApp(<PaymentsPage />);
  await user.type(screen.getByRole("searchbox"), "abc");

  expect(await screen.findByText(/results for abc/i)).toBeInTheDocument();
  await delay(600);
  expect(screen.queryByText(/results for a$/i)).not.toBeInTheDocument();   // ← never appears
});
```
**That last assertion is the whole test** — and it's the bug from [Lesson 08](../02-state-effects/08-effects.md) encoded so it can't regress.

### Accessibility, automatically
```tsx
import { axe } from "jest-axe";

test("the payments page has no a11y violations", async () => {
  const { container } = renderApp(<PaymentsPage />);
  await screen.findByRole("grid");
  expect(await axe(container)).toHaveNoViolations();
});
```
**`axe` catches roughly 30–40% of accessibility issues** — missing labels, contrast, invalid ARIA, bad nesting. The rest needs keyboard and screen-reader testing by hand. Automate what you can; be honest that it isn't everything.

### Virtualized lists
```tsx
test("renders only the visible window", async () => {
  renderApp(<PaymentsTable rows={buildPayments(10_000)} />);
  const rows = await screen.findAllByRole("row");
  expect(rows.length).toBeLessThan(50);                      // windowed
  expect(screen.getByRole("grid")).toHaveAttribute("aria-rowcount", "10000");   // ← real total
});
```
**jsdom has no layout**, so virtualizers see zero height and render nothing. Mock the scroll element's dimensions, or use `initialRect` — and test the *scrolling* behaviour in Playwright, which has a real browser.

---

## 7. What not to test

| Don't test | Why | Instead |
|---|---|---|
| Hook return values via `renderHook` | A hook is an implementation detail; the test passes while the component is broken | Test the component that uses it |
| Internal state | It's not the contract | Test what it renders |
| That a prop callback "was called" | Tests wiring, not behaviour | Test the resulting UI change |
| Implementation details (class names, structure) | Breaks on every refactor | Roles and text |
| Third-party libraries | They have their own tests | Trust them |
| Every permutation of a presentational component | Diminishing returns | One happy path + the interesting states |

```tsx
// ❌ Passes even if the button doesn't render, isn't reachable, or is disabled
test("calls onRefund", () => {
  const onRefund = vi.fn();
  render(<RefundButton onRefund={onRefund} />);
  fireEvent.click(screen.getByTestId("btn"));
  expect(onRefund).toHaveBeenCalled();
});

// ✅ Tests what actually matters
test("refunding shows a confirmation", async () => {
  const { user } = renderApp(<PaymentDetail paymentId={id} />);
  await user.click(await screen.findByRole("button", { name: /refund/i }));
  await user.click(screen.getByRole("button", { name: /confirm/i }));
  expect(await screen.findByText(/refund issued/i)).toBeInTheDocument();
});
```

**`renderHook` is legitimate** for genuinely reusable library-grade hooks with complex timing (debounce, retry backoff) — where a component test would be convoluted. For app hooks, test the component.

---

## 8. E2E with Playwright

```tsx
test("a merchant can refund a payment", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("merchant@test.dev");
  await page.getByLabel("Password").fill("correct-horse");
  await page.getByRole("button", { name: "Sign in" }).click();

  await page.getByRole("searchbox").fill("pi_3Nx8");
  await page.getByRole("row", { name: /pi_3Nx8/ }).click();
  await page.getByRole("button", { name: "Refund" }).click();
  await page.getByLabel("Amount").fill("49.99");
  await page.getByRole("button", { name: "Confirm refund" }).click();

  await expect(page.getByText("Refund issued")).toBeVisible();
  await expect(page.getByRole("status")).toContainText("Refunded");   // the row updated
});
```

**E2E rules that keep it from becoming a liability:**
- **3–5 journeys only** — login, the core flow, a payment path. Not coverage.
- **Real backend, seeded database** — the point is integration
- **Same role/label queries** as component tests
- **No arbitrary waits.** `await expect(...).toBeVisible()` auto-retries; `waitForTimeout` is how suites become flaky
- **Run in CI on a schedule and before deploys**, not on every PR — they're slow

```tsx
// Worth having: a11y and visual checks in the real browser
await expect(page).toHaveScreenshot("payments.png", { maxDiffPixels: 100 });
const results = await new AxeBuilder({ page }).analyze();
expect(results.violations).toEqual([]);
```

---

## 9. Flakiness — treat it as a bug

A flaky test is worse than no test: it trains the team to re-run CI instead of reading failures.

| Cause | Fix |
|---|---|
| `fireEvent` instead of `userEvent` | Missing focus/hover steps produce inconsistent results |
| Arbitrary `waitForTimeout` | Use `findBy*` / `toBeVisible`, which retry |
| Shared state between tests | Fresh `QueryClient`; `server.resetHandlers()` |
| Real timers with animations | `vi.useFakeTimers()`, or disable animations in tests |
| Retry-on-failure enabled | `retry: false` in the test query client |
| Test order dependence | Randomise order in CI so you *find* it |
| A genuine race in the code | **The flake is telling the truth. Fix the code** |

**Never add a blanket `retry: 3` to your test runner.** It hides real races — exactly the bugs your concurrency tests exist to find.

---

## 10. Ledger Console's test suite

```
test/
├── setup.ts              MSW server, jest-dom, axe
├── handlers/             generated from openapi.yaml + overrides
├── utils.tsx             TestProviders, renderApp, builders
└── builders.ts           buildPayment(), buildRefund() — override only what matters

src/features/payments/__tests__/
├── payments-page.test.tsx        filters ↔ URL, empty, error, loading
├── payments-table.test.tsx        virtualization window, aria-rowcount, keyboard nav
└── refund-flow.test.tsx           happy path, double-click, 409, 422, focus trap

src/features/refunds/refund-reducer.test.ts     pure, no React
src/lib/api-client.test.ts                       401 → single-flight refresh → retry
e2e/
├── auth.spec.ts          login → protected route → logout
├── refund.spec.ts        the money path, end to end
└── a11y.spec.ts          axe on each main route
```

**The tests that earn their place** — each encodes a bug that would otherwise recur:
```tsx
test("double-clicking refund sends one request", …);              // API L18
test("a slow response for an old query is discarded", …);          // React L08
test("filters round-trip through the URL", …);                     // React L16
test("a malformed URL renders defaults instead of crashing", …);    // TS L15
test("concurrent 401s trigger exactly one refresh", …);             // API L13
test("ticking row 2 then prepending keeps the tick on the same payment", …);   // React L04
test("the reducer rejects a second submit from submitting", …);     // React L06
test("aria-rowcount reports the total, not the window", …);         // React L21
test("an optimistic toggle rolls back on failure", …);              // React L18
```

**That list is the artefact.** It's a regression suite where every test corresponds to a specific, named failure mode — which is a far better answer to "how do you test?" than a coverage percentage.

---

## 11. Production rules

| Rule | Why |
|---|---|
| **Test behaviour, not implementation** | Implementation should be free to change |
| **`getByRole` first; `getByTestId` last** | Querying by role tests accessibility for free |
| **`userEvent`, never `fireEvent`** | `fireEvent` will "click" a disabled button |
| **MSW at the network boundary, not module mocks** | Your real client code runs, including error mapping and parsing |
| **`onUnhandledRequest: "error"`** | Catches requests you didn't know about |
| **`retry: false` in the test query client** | Otherwise error tests wait through backoff and time out |
| **A fresh `QueryClient` per test** | Cache leakage makes tests order-dependent |
| **`queryBy*` is the only way to assert absence** | `getBy*` throws |
| **Assert the *effect*, not the status** (`requests.length === 1`) | A 201 doesn't tell you how many refunds exist |
| **Test error paths, not just the happy path** | That's where the bugs are |
| **`axe` in component tests; keyboard and screen reader by hand** | Automation catches ~30–40% |
| **3–5 E2E journeys, no arbitrary waits** | More makes the suite slow and flaky |
| **Fix flakes; never retry them away** | A flake is often a real race |

---

## 12. Interview traps

**Q1. "How do you test React components?"**
Testing Library with `getByRole`, `userEvent`, and MSW for the network — testing what a user can do and see. **Then the framing:** component tests are the valuable layer for React, because the bugs live in the seams — wiring, state, effects, conditional rendering — which unit tests of hooks don't touch.

**Q2. "Why `getByRole` over `getByTestId`?"**
Because it tests accessibility as a side effect: if the query fails, the element isn't a button or has no accessible name, which is a real bug for keyboard and screen-reader users. `data-testid` breaks the connection between your test and how the app is actually used.

**Q3. "`userEvent` vs `fireEvent`?"**
`userEvent` simulates the full browser sequence — pointer events, focus, hover — and respects `disabled` and `pointer-events: none`. **`fireEvent` will happily click a disabled button**, which is exactly the bug you were testing for.

**Q4. "Should you test custom hooks directly?"**
Usually not — a hook is an implementation detail, and `renderHook` tests pass while the component is broken. Test the component that uses it. **Exception:** genuinely reusable library-grade hooks with complex timing (debounce, retry backoff), where a component test would be convoluted.

**Q5. "Why MSW instead of mocking fetch?"**
Your real client code runs — error mapping, schema parsing, retry, token refresh. The same handlers serve tests, Storybook and local dev. And `onUnhandledRequest: "error"` surfaces requests you didn't know you were making. Mocking your API module skips all the code most likely to be wrong.

**Q6. "What do you NOT test?"**
Hook return values, internal state, "the callback was called", class names and DOM structure, third-party libraries, and every permutation of a presentational component. **The test: would this break if I refactored without changing behaviour?** If yes, it's testing implementation.

**Q7. "How do you test that double-clicking doesn't double-charge?"**
Capture requests in an MSW handler with an artificial delay, click twice, and assert `requests.length === 1`. **Assert the effect, not the status** — a 201 tells you nothing about how many refunds exist.

**Q8. "How do you test a race condition?"**
Make MSW return different latencies per query, type quickly, then assert both that the *correct* result appears **and that the stale one never does**. That second assertion is the test.

**Q9. "How do you test accessibility?"**
Three layers: `getByRole` queries (which fail when accessibility is broken), `jest-axe` in component tests, and **manual keyboard and screen-reader passes**. Automation catches roughly 30–40% — be honest about that rather than claiming coverage you don't have.

**Q10. "A test is flaky. What do you do?"**
Investigate, never retry. Check in order: `fireEvent` instead of `userEvent`, arbitrary waits, shared state between tests, real timers, retries enabled in the query client, test-order dependence — and then the important one: **a genuine race in the code, where the flake is telling the truth.**

**Q11. "How much coverage do you need?"**
Reframe it: *"I care which behaviours are covered, not the percentage. 100% line coverage with no error paths, no concurrency tests and no accessibility checks is a worse suite than 60% that has all three. The coverage I'd enforce is that every critical user journey has a test."*

---

## 13. Build & break

### Build — the suite for Ledger Console
1. `setup.ts` with MSW (handlers generated from `openapi.yaml`), `jest-dom`, `jest-axe`.
2. `TestProviders` + `renderApp` with `retry: false` and a fresh `QueryClient`.
3. Builders: `buildPayment()`, `buildRefund()` — override only what the test cares about.
4. The nine tests from §10, each mapped to a named failure mode.
5. Three Playwright journeys.
6. `axe` on every main route.

### Build — the regression-test exercise
For each of the five bugs you deliberately created while learning — the index-key bug, the race condition, the stale closure, the double submit, the missing rollback — **write the test that would have caught it.** Then break the code again and watch the test fail.

**That exercise is the whole point of a test suite**, and it produces a suite where every test has a story.

### Break — five experiments
1. **`fireEvent` on a disabled button.** It "succeeds". Switch to `userEvent` and watch it correctly fail.
2. **Retries on.** Write a test asserting an error state with the default retry policy. Watch it take 7 seconds and then time out.
3. **Shared cache.** Use one `QueryClient` across tests. Watch a test pass alone and fail in sequence.
4. **Implementation test.** Write a `renderHook` test for `usePayments`, then refactor the hook's internals without changing behaviour. Watch a passing app with a failing test.
5. **jsdom and virtualization.** Test a virtualized list without mocking dimensions. Zero rows render.

### Explain out loud (90 seconds)
1. Why component tests are React's valuable layer.
2. Why `getByRole` first.
3. Why MSW beats mocking your API module.
4. Three things you deliberately don't test.
5. How you'd test that a double-click doesn't double-charge.

---

## Module 8 complete — checkpoint

- [ ] The Console's state model, and the three decisions you'd defend
- [ ] The five hard parts, and what each demonstrates
- [ ] The build order, and why Phase 4 is the one to build first if time is short
- [ ] Why component tests are the valuable layer
- [ ] Query priority, and why `getByRole` tests a11y for free
- [ ] `userEvent` vs `fireEvent`
- [ ] Why MSW, and `onUnhandledRequest: "error"`
- [ ] What not to test
- [ ] How to test idempotency and race conditions

---

## What's next

The final module converts everything into interview performance: a rapid-fire question bank covering the whole track, then machine coding and frontend design rounds.

Next → **[Lesson 29: Rapid-fire master Q&A](../09-interview/29-rapid-fire-master-qa.md)**
