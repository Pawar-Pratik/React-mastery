# Lesson 18 — Mutations, optimistic updates & races

> **Why this lesson exists:** reads are forgiving — a stale read corrects itself on the next fetch. **Writes aren't.** A mutation that double-fires charges a customer twice; an optimistic update without rollback shows a refund that never happened. This lesson covers the write path end to end, and it's where the React track and the API track meet most directly: idempotency keys, error contracts, and rollback all cross the boundary.

**Time:** ~70 minutes · **Prereq:** Lesson 17 · [API Lesson 18](../../API/04-production/18-reliability-and-idempotency.md)

---

## 1. The idea in one sentence

> **A mutation changes server state, so afterwards your cache is *wrong* — and the whole design is about deciding when to correct it, how to show progress meanwhile, and what to do when the write fails.**

---

## 2. The basic mutation

```tsx
function useRefundPayment() {
  const qc = useQueryClient();

  return useMutation({
    mutationFn: (input: RefundInput) =>
      api.createRefund(input.paymentId, input, input.idempotencyKey),

    onSuccess: (refund, input) => {
      // The cache is now stale for anything this touched
      qc.invalidateQueries({ queryKey: paymentKeys.detail(input.paymentId) });
      qc.invalidateQueries({ queryKey: paymentKeys.lists() });
      qc.invalidateQueries({ queryKey: ["balance"] });
      toast.success(`Refunded ${formatMoney(refund.money)}`);
    },

    onError: (error) => {
      if (error instanceof ApiError && error.code === "already_refunded") {
        toast.error("This payment was already refunded");
        qc.invalidateQueries({ queryKey: paymentKeys.all });    // our cache was wrong
      } else {
        toast.error("Refund failed. Please try again.");
      }
    },
  });
}

// In the component
const refund = useRefundPayment();
<Button loading={refund.isPending} onClick={() => refund.mutate(input)}>Refund</Button>
```

**Three decisions in there:**

**1. Invalidate everything the write touched.** A refund changes the payment, every list containing it, and the balance. Forgetting the balance means the user sees an unchanged number and thinks the refund didn't work. **Write down the blast radius of every mutation** — it's the part people miss.

**2. Branch on the error `code`, not the message.** `already_refunded` is a specific, recoverable condition from your API contract ([API Lesson 10](../../API/02-rest-design/10-errors-and-problem-details.md)) — and it tells you your cache was stale, so refetch.

**3. `mutate` vs `mutateAsync`:**
```tsx
refund.mutate(input);                      // fire-and-forget; errors go to onError
await refund.mutateAsync(input);           // returns a promise; YOU must catch it
```
`mutateAsync` rejects on failure, so an uncaught one becomes an unhandled rejection ([TS Lesson 16](../../TypeScript/04-real-code/16-async-and-errors.md)). **Prefer `mutate` unless you genuinely need to await** — for example to close a modal only after success.

---

## 3. Idempotency — the part that isn't optional

```tsx
function RefundModal({ payment, open }: Props) {
  // ONE key per logical operation, stable across retries, reset when the user starts over
  const idempotencyKey = useIdempotencyKey(payment.id + String(open));

  const refund = useRefundPayment();
  return <Button onClick={() => refund.mutate({ paymentId: payment.id, amountMinor, idempotencyKey })} />;
}
```

**Why it's mandatory here, not a nice-to-have:**
```
User clicks Refund → request sent → network stalls → user clicks again
  Without a key: TWO refunds. Real money, gone twice.
  With a key:    the server recognises the retry and replays the first response.
```

Three layers of protection, and **each can fail alone** ([Lesson 14](../04-composition/14-forms.md)):
1. `disabled={isPending}` — a UI affordance; defeated by keyboard, a race, or a second code path
2. The state machine has no `submit` transition from `submitting` — structural ([Lesson 06](../02-state-effects/06-reducers-and-state-machines.md))
3. **`Idempotency-Key` on the request** — because the *network* can duplicate it even when your UI can't

```tsx
// The key must be stable across retries and fresh per operation
export function useIdempotencyKey(resetOn: unknown): IdempotencyKey {
  const keyRef = useRef<IdempotencyKey | null>(null);
  const prevRef = useRef(resetOn);
  if (keyRef.current === null || prevRef.current !== resetOn) {
    keyRef.current = crypto.randomUUID() as IdempotencyKey;
    prevRef.current = resetOn;
  }
  return keyRef.current;
}
```
A **ref**, not state — it must not cause a re-render, and it must survive re-renders unchanged ([Lesson 07](../02-state-effects/07-refs-and-escape-hatches.md)).

> **The interview point:** *"the retry that duplicates the charge often isn't the user clicking twice — it's the client library, a service worker, or a proxy retrying a request whose response was lost. That's why the key goes on the request, not just a disabled button."*

---

## 4. Optimistic updates

Show the result immediately; correct it if the server disagrees.

```tsx
function useRefundPayment() {
  const qc = useQueryClient();

  return useMutation({
    mutationFn: (input: RefundInput) => api.createRefund(...),

    onMutate: async (input) => {
      // 1. Cancel in-flight refetches, or one could overwrite our optimistic value
      await qc.cancelQueries({ queryKey: paymentKeys.detail(input.paymentId) });

      // 2. Snapshot for rollback
      const previous = qc.getQueryData<Payment>(paymentKeys.detail(input.paymentId));

      // 3. Optimistically update
      qc.setQueryData<Payment>(paymentKeys.detail(input.paymentId), old =>
        old ? { ...old, status: "refunded", refunded: { amountMinor: input.amountMinor,
                                                        currency: old.money.currency } } : old);

      return { previous };                              // → passed to onError/onSettled
    },

    onError: (_err, input, ctx) => {
      // 4. Roll back to the snapshot
      if (ctx?.previous) qc.setQueryData(paymentKeys.detail(input.paymentId), ctx.previous);
      toast.error("Refund failed — the payment was not refunded");
    },

    onSettled: (_data, _err, input) => {
      // 5. ALWAYS reconcile with the server, success or failure
      qc.invalidateQueries({ queryKey: paymentKeys.detail(input.paymentId) });
      qc.invalidateQueries({ queryKey: ["balance"] });
    },
  });
}
```

**The five steps are not optional, and each exists for a reason:**

| Step | If you skip it |
|---|---|
| `cancelQueries` | An in-flight refetch resolves with old data and overwrites your optimistic update |
| Snapshot | You have nothing to roll back to |
| `setQueryData` | No optimistic update at all |
| Rollback in `onError` | A failed refund stays visible as "refunded" — **the worst possible outcome** |
| `invalidate` in `onSettled` | Your optimistic guess diverges from the server's actual result |

**Step 5 matters even on success**, because your optimistic value is a *guess*. The server may have computed a fee, adjusted the amount, or set a different status. Always reconcile.

### React 19's `useOptimistic`
```tsx
const [optimisticRefunds, addOptimistic] = useOptimistic(
  refunds,
  (current, pending: Refund) => [...current, { ...pending, status: "pending" as const }],
);

async function submit(formData: FormData) {
  addOptimistic(buildRefund(formData));      // instant
  await api.createRefund(...);                // React reverts automatically when this settles
}
```
**`useOptimistic` reverts automatically** when the surrounding action completes — the optimistic value only exists during the transition. Simpler than the five-step dance, but it's scoped to one action rather than a shared cache. **Use it for local, form-scoped optimism; use the cache version when many components read the same data.**

### When *not* to be optimistic
```
❌ The operation can fail for reasons you can't predict (card declined, insufficient funds)
❌ The server computes something you can't (fees, tax, an assigned ID, a new balance)
❌ Rollback would be confusing or alarming (money appearing then vanishing)
❌ The operation is slow AND rare — a spinner is honest and fine
✅ The result is predictable and the failure rate is low: toggles, likes, reordering,
   marking read, adding a tag, optimistic list insertion
```

> **The judgement to express:** *"I'd be optimistic about a toggle or a tag, not about a refund. Money appearing and then disappearing is worse than a two-second spinner — an optimistic update is a promise to the user, and you shouldn't make promises you can't keep."*

---

## 5. Cache updates: invalidate vs setQueryData

```tsx
// A) Invalidate — mark stale, refetch. Simple, always correct, costs a round trip.
onSuccess: () => qc.invalidateQueries({ queryKey: paymentKeys.lists() })

// B) setQueryData — write the server's response into the cache. No extra request.
onSuccess: (updated) => qc.setQueryData(paymentKeys.detail(updated.id), updated)

// C) Both — write the authoritative response, invalidate what you can't compute
onSuccess: (refund, input) => {
  qc.setQueryData(paymentKeys.detail(input.paymentId), refund.payment);  // exact
  qc.invalidateQueries({ queryKey: paymentKeys.lists() });                // can't recompute ordering
  qc.invalidateQueries({ queryKey: ["balance"] });                        // server-computed
}
```

**The rule:** `setQueryData` when the server's response *is* the new state of that entity. `invalidateQueries` when the result depends on server-side computation you can't replicate — list ordering, aggregates, counts, balances, pagination.

```tsx
// Updating an entity inside every cached list — the fiddly case
qc.setQueriesData<Paginated<Payment>>({ queryKey: paymentKeys.lists() }, old =>
  old ? { ...old, data: old.data.map(p => p.id === updated.id ? updated : p) } : old);
```
**The trap:** if the update changes something the list is *filtered or sorted by* (status, amount, date), patching in place leaves the row in the wrong position or in a list it no longer belongs to. **Invalidate instead when the mutation could change membership or ordering.**

---

## 6. Race conditions in mutations

Reads race ([Lesson 08](../02-state-effects/08-effects.md)); writes race differently and worse.

```tsx
// ── Race 1: two mutations to the same resource ──
// User edits a note, saves, edits again, saves. Request A is slow, B is fast.
//   B commits → A commits with the OLDER value → the user's latest edit is lost.
```
```tsx
// Fixes, in order of strength:
// (a) Serialize with a mutation scope — TanStack runs same-scope mutations in order
useMutation({ mutationFn, scope: { id: `payment-${paymentId}` } });

// (b) Optimistic concurrency — send the version you read; the server 412s on a stale write
await api.updatePayment(id, patch, { ifMatch: payment.version });   // API Lesson 09

// (c) Disable the control while pending — weakest, but often enough
```
**(b) is the correct answer for anything that matters**, and it's the client half of the lost-update problem from [API Lesson 09](../../API/02-rest-design/09-writes-patch-and-bulk.md): the server rejects a write based on stale data with a `412`, and the client re-reads and retries.

```tsx
onError: (err, input) => {
  if (err instanceof ApiError && err.code === "stale_version") {
    toast.warning("This payment changed while you were editing. Reloading…");
    qc.invalidateQueries({ queryKey: paymentKeys.detail(input.paymentId) });
  }
}
```

```tsx
// ── Race 2: a refetch overwrites an optimistic update ──
// Solved by cancelQueries in onMutate (§4, step 1)

// ── Race 3: a mutation completes after the component unmounts ──
// Fine — mutations aren't bound to component lifetime, and the cache update still applies.
// But guard any navigation/toast that assumes the component still exists.
```

---

## 7. Error handling in mutations

```tsx
onError: (error, input, ctx) => {
  rollback(ctx);

  if (!(error instanceof ApiError)) { toast.error("Something went wrong"); return; }

  switch (error.code) {
    case "validation_failed":
      // Map field errors back to the form (Lesson 14 / API Lesson 10)
      for (const e of error.fields ?? []) setFormError(e.field, e.detail);
      break;
    case "already_refunded":
      toast.error("Already refunded");
      qc.invalidateQueries({ queryKey: paymentKeys.detail(input.paymentId) });   // we were stale
      break;
    case "insufficient_balance":
      toast.error("Not enough balance to issue this refund");
      break;
    case "rate_limit_exceeded":
      toast.error(`Too many requests. Retry in ${error.retryAfter}s`);           // API Lesson 19
      break;
    default:
      toast.error(error.retryable
        ? "Temporary problem — please try again"
        : "Refund failed. Contact support with reference " + error.requestId);   // API Lesson 20
  }
}
```

**Three things this does that most error handling doesn't:**
- Branches on **`code`**, never on message text — the stable contract
- Uses **`retryable`** to decide whether to suggest retrying
- Surfaces **`requestId`** for unrecoverable errors, so a support ticket is answerable in one query

That last one is the observability loop closing: the API returns `X-Request-Id` on every response ([API Lesson 20](../../API/04-production/20-observability.md)), and the UI shows it so the user can quote it.

### Retries on mutations
```tsx
useMutation({
  mutationFn,
  retry: (count, error) =>
    error instanceof ApiError && error.retryable && count < 2,     // 5xx/429 only
});
```
**Only retry a mutation if it's idempotent** — which, with an idempotency key, it is. Without one, an automatic retry is how you double-charge. The two features are a pair: **the key is what makes the retry safe.**

---

## 8. Ledger Console's mutations

| Mutation | Optimistic? | Invalidates | Notes |
|---|---|---|---|
| **Refund payment** | ❌ | payment, lists, balance, events | Money — a wrong optimistic state is unacceptable |
| **Capture payment** | ❌ | payment, lists, balance | Same |
| **Update payment metadata** | ✅ | payment | Predictable; `If-Match` for concurrency |
| **Create customer** | ❌ | customer lists | Server assigns the ID |
| **Toggle webhook endpoint** | ✅ | endpoints | Simple boolean, low failure rate |
| **Retry webhook delivery** | ✅ (status → "pending") | deliveries | Reconciled on settle |
| **Revoke API key** | ✅ (remove from list) | keys | Rollback is clean and obvious |

```tsx
// The optimistic toggle — the clean case
export function useToggleEndpoint() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, enabled }: { id: string; enabled: boolean }) =>
      api.updateEndpoint(id, { enabled }),
    onMutate: async ({ id, enabled }) => {
      await qc.cancelQueries({ queryKey: webhookKeys.lists() });
      const previous = qc.getQueryData<WebhookEndpoint[]>(webhookKeys.lists());
      qc.setQueryData<WebhookEndpoint[]>(webhookKeys.lists(), old =>
        old?.map(e => e.id === id ? { ...e, enabled } : e));
      return { previous };
    },
    onError: (_e, _v, ctx) => {
      if (ctx?.previous) qc.setQueryData(webhookKeys.lists(), ctx.previous);
      toast.error("Couldn't update the endpoint");
    },
    onSettled: () => qc.invalidateQueries({ queryKey: webhookKeys.lists() }),
  });
}
```

**Notice the pattern in the table:** anything touching money is never optimistic. That's not conservatism — it's that the *cost of being wrong* differs. A toggle flipping back is a minor annoyance; a refund appearing and vanishing is a support call and a loss of trust.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Every money-moving mutation carries an `Idempotency-Key`** | The network can duplicate a request even when your UI can't |
| **The key lives in a ref, stable across retries, reset per operation** | State would re-render; a fresh key defeats the purpose |
| **Write down each mutation's blast radius and invalidate all of it** | Forgetting the balance makes a successful refund look failed |
| **Optimistic updates need all five steps** | Skipping rollback leaves a failed refund showing as succeeded |
| **`cancelQueries` before an optimistic write** | An in-flight refetch will otherwise overwrite it |
| **`onSettled` invalidation even on success** | Your optimistic value is a guess; the server is authoritative |
| **Never be optimistic about money or server-computed values** | Money appearing then vanishing is worse than a spinner |
| **Don't patch cached lists when the update changes filtering or ordering** | The row ends up in the wrong place or the wrong list |
| **Branch on error `code`, use `retryable`, surface `requestId`** | The API contract exists to be consumed |
| **Only auto-retry mutations that carry an idempotency key** | Otherwise retries duplicate side effects |
| **Use `If-Match`/version for concurrent edits** | The client half of the lost-update problem |
| **Prefer `mutate` over `mutateAsync`** | `mutateAsync` rejects, and an uncaught rejection is a crash |

---

## 10. Interview traps

**Q1. "A user clicks Refund twice. What happens?"**
The three layers, and **why each alone is insufficient**: a disabled button is defeatable by keyboard, race or a second code path; a state machine prevents *your* duplicate dispatch but not a network-level one; an `Idempotency-Key` is what actually guarantees the server processes it once. Then the real insight: *"the duplicate is often not the user — it's a client library, service worker or proxy retrying a request whose response was lost."*

**Q2. "Walk me through an optimistic update."**
The five steps: cancel in-flight queries, snapshot for rollback, apply the optimistic change, roll back in `onError`, reconcile in `onSettled`. **For each, say what breaks without it** — that's what shows you've shipped one rather than read about one.

**Q3. "When would you NOT do an optimistic update?"**
When the operation can fail unpredictably (card declined), when the server computes something you can't (fees, IDs, balances), and when rollback would alarm the user. *"I'd be optimistic about a toggle, not a refund — an optimistic update is a promise, and money appearing then vanishing breaks trust worse than a spinner."*

**Q4. "`invalidateQueries` vs `setQueryData`?"**
`setQueryData` when the server's response *is* the new entity state — no extra request. `invalidate` when the result depends on server computation you can't replicate: ordering, aggregates, counts, balances. Often both. **And don't patch a list in place if the update changes what the list is filtered or sorted by.**

**Q5. "Two saves race and the older one wins. Fix it."**
Three options: serialize mutations by scope, use optimistic concurrency (`If-Match` with the version you read, server returns `412` on a stale write), or disable the control while pending. **(2) is the correct answer for anything that matters** — it's the client half of the lost-update problem, and the server must enforce it regardless.

**Q6. "Should mutations retry automatically?"**
Only if they're idempotent — which means only if they carry an idempotency key. Otherwise a retry duplicates the side effect. Retry on `retryable` errors (5xx, 429) with backoff; never on 4xx. **The key and the retry are a pair.**

**Q7. "The server says `already_refunded`. What does that tell you?"**
Two things: the operation genuinely can't proceed, **and your cache was stale** — so you invalidate and refetch rather than just showing an error. Reading a conflict error as a cache-staleness signal is the answer that stands out.

**Q8. "What's `useOptimistic` and when is it better?"**
React 19's hook: an optimistic value derived from real state plus a pending action, which **reverts automatically** when the surrounding transition settles. Simpler than the five-step cache dance, but scoped to one action. Use it for form-local optimism; use the cache version when several components read the same data.

**Q9. "A mutation succeeds but the UI doesn't update."**
Almost always missing or wrong invalidation: the key doesn't match what the component queries (a key factory prevents this), or you invalidated the detail but not the list, or `staleTime` is high enough that the refetch didn't happen. Check the Devtools cache — the query is there with the old data, unmarked as stale.

**Q10. "How do you show the user something actionable when a mutation fails?"**
Branch on the error `code`: map `validation_failed` field errors back onto the form inputs, use `retryable` to decide whether to offer a retry, honour `retryAfter` on 429, and show the `requestId` for unrecoverable errors so support can look it up in one query. **All four come from the API error contract**, which is why designing that contract well pays off here.

---

## 11. Build & break

### Build — the refund mutation, completely
Implement `useRefundPayment` with: an idempotency key from a ref, the full blast-radius invalidation, code-based error branching with field mapping, `retryable`-aware retry, and `requestId` surfaced in the failure toast.

Then verify behaviour against a mock API:
- Succeed → payment, list and balance all update
- Return `409 already_refunded` → error toast **and** a refetch
- Return `422 validation_failed` → errors land on the form fields
- Return `500` → retried twice with backoff, then a retryable message
- Click twice rapidly with a 3s delay → **one refund**, confirmed by the request log

### Build — an optimistic toggle with induced failure
Implement the webhook toggle from §8, then make the API fail 50% of the time. Watch it flip, then flip back. **Then remove the rollback and watch it stay wrong** — that's the failure mode you're protecting against, and seeing it once makes the five steps memorable.

### Build — the mutation race
Two saves to the same resource, the first artificially slow. Observe the older value winning. Then add `If-Match` with the version, have the mock return `412` on a stale write, and handle it by refetching and telling the user.

### Break — five experiments
1. **No `cancelQueries`.** Trigger an optimistic update while a refetch is in flight. Watch the refetch overwrite your optimistic value.
2. **No `onSettled` invalidate.** Optimistically set `status: "refunded"` while the server returns `"pending"` (async settlement). The UI is permanently wrong.
3. **Wrong invalidation key.** Invalidate `["payment", id]` while the query uses `["payments", "detail", id]`. Nothing updates. **This is the argument for a key factory.**
4. **Patching a filtered list.** With a `status=succeeded` filter active, optimistically set a row to `refunded` via `setQueryData`. The row stays in a list it no longer belongs to.
5. **Retry without a key.** Enable `retry: 3` on a non-idempotent create and force a 500 after the write succeeds server-side. Count the created records.

### Explain out loud (90 seconds)
1. The three layers of double-submit protection, and why each alone fails.
2. The five steps of an optimistic update, and what breaks without each.
3. When you would *not* be optimistic.
4. `invalidateQueries` vs `setQueryData`, and the list-membership trap.
5. How you'd handle two concurrent edits to the same record.

---

## What's next

Mutations push data up. The last piece of Module 5 is data arriving **unprompted** — SSE and WebSockets integrated with the cache rather than fighting it, which is how the live payment feed works.

Next → **[Lesson 19: Real-time in React](19-realtime.md)**
