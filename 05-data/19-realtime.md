# Lesson 19 — Real-time in React

> **Why this lesson exists:** real-time is where the effect model, the cache and the network all meet, and it's where hand-rolled code goes wrong in ways that only show up in production — duplicate connections after a fast refresh, memory leaks from missed cleanup, events applied out of order, and a cache that drifts from the server after a reconnect. The React half is mostly about **integrating with the cache instead of maintaining a second copy of the data.**

**Time:** ~60 minutes · **Prereq:** Lessons 08, 17 · [API Lesson 23](../../API/05-beyond-rest/23-realtime-and-webhooks.md)

---

## 1. The idea in one sentence

> **A real-time message is a *signal that something changed*, not a replacement for your data layer — so the right reaction is usually to update or invalidate the cache, not to hold a parallel copy in component state.**

---

## 2. Choosing the mechanism

The decision tree from [API Lesson 23](../../API/05-beyond-rest/23-realtime-and-webhooks.md), applied client-side:

```
Do you need to SEND at low latency too?
├─ Yes (chat, collaborative cursors, games)          → WebSocket
└─ No  (notifications, live feed, progress, tokens)  → SSE

Is a few seconds of delay fine, or is the volume tiny?
└─ Yes → POLLING with refetchInterval. Don't apologise for it.
```

**Polling is genuinely the right answer more often than people expect**, and a query library makes it one line:
```tsx
useQuery({ queryKey: paymentKeys.detail(id), queryFn: ..., refetchInterval: 5_000 });
```
Stateless, works through every proxy, no reconnection logic, and with `ETag`/`304` an empty poll costs ~150 bytes ([API Lesson 17](../../API/04-production/17-caching-and-performance.md)). The argument against it is arithmetic: 10,000 clients at 0.5 Hz is 5,000 req/s. **Measure before adding a connection.**

```tsx
// Smart polling — stop when there's nothing to watch
useQuery({
  queryKey: paymentKeys.detail(id),
  queryFn: ...,
  refetchInterval: q => TERMINAL.has(q.state.data?.status as PaymentStatus) ? false : 3_000,
  refetchIntervalInBackground: false,      // ← don't poll a hidden tab
});
```
That `refetchIntervalInBackground: false` is the default and worth knowing: a background tab stops polling, which is both polite and battery-friendly.

---

## 3. SSE in React — the correct shape

```tsx
export function usePaymentStream() {
  const qc = useQueryClient();
  const [status, setStatus] = useState<"connecting" | "open" | "closed">("connecting");

  // The handler reads the latest cache client without being a dependency (Lesson 07)
  const onEvent = useEventCallback((event: LedgerEvent) => {
    switch (event.type) {
      case "payment.succeeded":
      case "payment.failed":
      case "payment.refunded":
        // Write the authoritative object into the detail cache…
        qc.setQueryData(paymentKeys.detail(event.data.object.id), event.data.object);
        // …and invalidate lists, because ordering/filtering may change (Lesson 18)
        qc.invalidateQueries({ queryKey: paymentKeys.lists() });
        qc.invalidateQueries({ queryKey: ["balance"] });
        break;
      case "webhook.delivery_failed":
        qc.invalidateQueries({ queryKey: webhookKeys.deliveries(event.data.object.endpoint_id) });
        break;
      default:
        break;                                 // ← unknown event types are IGNORED, not errors
    }
  });

  useEffect(() => {
    const es = new EventSource("/v1/events/stream", { withCredentials: true });

    es.onopen = () => setStatus("open");
    es.onmessage = e => onEvent(JSON.parse(e.data));
    es.onerror = () => setStatus("connecting");   // EventSource auto-reconnects

    return () => { es.close(); setStatus("closed"); };
  }, [onEvent]);                                   // onEvent is stable → connects ONCE

  return status;
}
```

**Four things that make this correct:**

1. **`useEventCallback` for the handler** ([Lesson 07](../02-state-effects/07-refs-and-escape-hatches.md)) — a stable identity with fresh behaviour, so the effect's dependency array is honest *and* the connection isn't recreated on every render. Without it, either you reconnect constantly or you silence the lint rule and read stale data.

2. **Cleanup closes the connection.** StrictMode's double-invoke exists precisely to catch a missing `es.close()` ([Lesson 08](../02-state-effects/08-effects.md)). A missed one means two connections per mount, doubling on every fast refresh.

3. **Update the cache, don't hold a second copy.** The stream's job is to keep the cache correct. Components keep reading `usePayment(id)` and get updates automatically — **no component needs to know the stream exists.** That's the architectural point of this lesson.

4. **Unknown event types are ignored.** The server will add event types; a client that throws on an unrecognised one breaks on the next backend deploy ([API Lesson 11](../../API/02-rest-design/11-versioning-and-evolution.md)).

### Resume and gap detection
```tsx
// EventSource sends Last-Event-ID automatically on reconnect, so the server can replay.
// But a long disconnect may exceed the server's replay window — so verify.
es.onopen = () => {
  setStatus("open");
  qc.invalidateQueries({ queryKey: paymentKeys.all });    // ← refetch after ANY reconnect
};
```
**Refetching on reconnect is the safety net.** You can't guarantee the replay covered everything, so treat a reconnect as "my cache may have drifted" and revalidate. Cheap, and it makes the whole system self-healing.

```tsx
// Gap detection using the sequence number the API provides (API Lesson 23)
const lastSeq = useRef(0);
onEvent(event => {
  if (event.sequence > lastSeq.current + 1 && lastSeq.current !== 0) {
    qc.invalidateQueries({ queryKey: paymentKeys.all });   // we missed events — resync
  }
  lastSeq.current = event.sequence;
  applyEvent(event);
});
```

### The SSE gotchas
| Gotcha | Fix |
|---|---|
| **HTTP/1.1's 6-connection limit** — an SSE stream holds one permanently | Serve over **HTTP/2**. Two tabs with three streams each will otherwise deadlock |
| **Proxies buffer the stream** | `X-Accel-Buffering: no` server-side; heartbeats keep it open |
| **No custom headers** — `EventSource` can't set `Authorization` | Cookie auth, or a short-lived ticket in the query string ([API L23](../../API/05-beyond-rest/23-realtime-and-webhooks.md)) |
| **Auto-reconnect is relentless** | If the server is down it retries forever; add your own backoff by closing and reopening |

---

## 4. WebSockets in React

```tsx
export function useWebSocket(url: string, onMessage: (m: ServerMessage) => void) {
  const handler = useEventCallback(onMessage);
  const wsRef = useRef<WebSocket | null>(null);
  const [status, setStatus] = useState<Status>("connecting");
  const attemptRef = useRef(0);

  useEffect(() => {
    let closed = false;
    let reconnectTimer: number;
    let heartbeat: number;

    function connect() {
      const ws = new WebSocket(url);
      wsRef.current = ws;

      ws.onopen = () => {
        setStatus("open");
        attemptRef.current = 0;
        ws.send(JSON.stringify({ type: "auth", token: getAccessToken() }));   // auth-by-first-message
        heartbeat = window.setInterval(() => ws.send(JSON.stringify({ type: "ping" })), 30_000);
      };

      ws.onmessage = e => handler(JSON.parse(e.data));

      ws.onclose = () => {
        clearInterval(heartbeat);
        setStatus("closed");
        if (closed) return;
        // Exponential backoff with FULL jitter — API Lesson 18
        const attempt = attemptRef.current++;
        const delay = Math.random() * Math.min(30_000, 500 * 2 ** attempt);
        reconnectTimer = window.setTimeout(connect, delay);
      };
    }

    connect();
    return () => {
      closed = true;
      clearTimeout(reconnectTimer);
      clearInterval(heartbeat);
      wsRef.current?.close();
    };
  }, [url, handler]);

  const send = useCallback((msg: ClientMessage) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) wsRef.current.send(JSON.stringify(msg));
  }, []);

  return { status, send };
}
```

**The four things hand-rolled WebSocket code usually gets wrong:**

1. **Reconnection without jitter.** When a server restarts, every client reconnects simultaneously and re-kills it. **Full jitter** (`random(0, cap)`) spreads them out — the same fix as [API Lesson 18](../../API/04-production/18-reliability-and-idempotency.md).
2. **No heartbeat.** Intermediaries silently kill idle connections and TCP won't tell you for minutes. Ping every 30s; treat a missed pong as dead.
3. **Auth in the URL.** `wss://api/ws?token=...` puts a credential in a URL that gets logged everywhere. Use a cookie, or authenticate with the first message, or a short-lived single-use ticket.
4. **No cleanup guard.** Without the `closed` flag, the cleanup runs but a queued reconnect timer still fires — reconnecting a component that unmounted.

> **Consider a library.** `partysocket`, `socket.io-client` or your provider's SDK handle reconnection, backoff, heartbeats and buffering. Hand-rolling is fine to *understand*; shipping it is a lot of edge cases.

---

## 5. Integrating with the cache — the patterns

```tsx
// Pattern 1 — Write the object (when the event carries the full entity)
qc.setQueryData(paymentKeys.detail(event.data.object.id), event.data.object);

// Pattern 2 — Invalidate (when the event is thin, or ordering/aggregates change)
qc.invalidateQueries({ queryKey: paymentKeys.lists() });

// Pattern 3 — Prepend to a live list, with a bounded size
qc.setQueryData<Paginated<Payment>>(paymentKeys.list(liveFilters), old => {
  if (!old) return old;
  if (old.data.some(p => p.id === event.data.object.id)) return old;      // dedupe!
  return { ...old, data: [event.data.object, ...old.data].slice(0, 100) }; // bound it
});

// Pattern 4 — A counter/badge without touching the list
qc.setQueryData<number>(["unread-count"], n => (n ?? 0) + 1);
```

**Pattern 3's two guards are the ones people miss:** deduplicate (you may have already fetched this item, or received the event twice — at-least-once delivery means duplicates are guaranteed), and bound the array (a feed running for eight hours will otherwise consume unbounded memory).

### Ordering and idempotency — the client side
```tsx
// Events can arrive out of order (API Lesson 23) — guard with the sequence/version
qc.setQueryData<Payment>(paymentKeys.detail(id), old => {
  if (old && old.version >= incoming.version) return old;   // ← ignore an older event
  return incoming;
});
```
**The server tells you not to assume ordering; this is how you honour that.** Without the version check, a slow `payment.created` event arriving after `payment.succeeded` reverts the UI to "pending".

### Batching bursts
```tsx
// 500 events/second would re-render 500 times. Buffer and flush.
const buffer = useRef<LedgerEvent[]>([]);
const flush = useEventCallback(() => {
  if (buffer.current.length === 0) return;
  const batch = buffer.current;
  buffer.current = [];
  applyBatch(qc, batch);                       // one cache write, one render
});
useEffect(() => {
  const id = setInterval(flush, 250);          // or requestAnimationFrame for smoothness
  return () => clearInterval(id);
}, [flush]);
```
Worth doing for anything high-frequency — a trading feed, a log tail, a progress stream. **React 18's automatic batching helps within a tick but not across 500 separate socket messages.**

---

## 6. Connection lifecycle and UX

```tsx
// One connection for the whole app, shared via context — not one per component
export function RealtimeProvider({ children }: { children: React.ReactNode }) {
  const status = usePaymentStream();                 // ONE EventSource for the app
  return <RealtimeStatusContext value={status}>{children}</RealtimeStatusContext>;
}
```
**One connection per app, not per component.** Ten components each opening their own stream is ten connections, ten reconnect loops, and ten copies of every event. Put it in a provider near the root.

```tsx
// Pause when the tab is hidden — battery and server cost
useEffect(() => {
  function onVisibility() {
    if (document.hidden) es.current?.close();
    else connect();
  }
  document.addEventListener("visibilitychange", onVisibility);
  return () => document.removeEventListener("visibilitychange", onVisibility);
}, []);
// Reconnecting refetches anyway (§3), so no data is lost.
```

```tsx
// Show connection state — users notice silence, and assume the app is broken
function ConnectionBadge() {
  const status = useRealtimeStatus();
  if (status === "open") return null;                       // don't nag when it's fine
  return (
    <div role="status" aria-live="polite" className="banner">
      {status === "connecting" ? "Reconnecting…" : "Offline — showing cached data"}
    </div>
  );
}
```
**`aria-live="polite"` matters here.** A connection banner that appears silently is invisible to screen-reader users, who then have no idea the data has stopped updating.

---

## 7. Ledger Console's real-time design

| Data | Mechanism | Why |
|---|---|---|
| **Live payment feed** | SSE, one app-wide connection | One-way, text, auto-reconnect with resume |
| **A single pending payment's status** | Polling, 3s, stops at a terminal state | Simpler than subscribing per row |
| **Webhook delivery attempts** | Polling, 5s, only while the page is open | Low volume, page-scoped |
| **Balance** | Invalidated by payment events + a 30s poll | Money must be exact ([API L17](../../API/04-production/17-caching-and-performance.md)) |
| **Toasts for events** | Derived from the same SSE stream | No second connection |

```tsx
// The whole integration — note that no page component knows the stream exists
function App() {
  return (
    <QueryClientProvider client={qc}>
      <RealtimeProvider>          {/* one EventSource, updating the cache */}
        <RouterProvider router={router} />
        <ConnectionBadge />
        <ToastViewport />
      </RealtimeProvider>
    </QueryClientProvider>
  );
}

// PaymentsTable just reads the cache. Live updates arrive for free.
function PaymentsTable() {
  const { data } = usePayments(filters);
  return <VirtualTable rows={data?.pages.flatMap(p => p.data) ?? []} />;
}
```

**That's the architectural payoff:** real-time is a *cache-update mechanism*, not a component concern. No page has real-time code in it; the stream keeps the cache correct and every component that reads the cache benefits automatically. If you find real-time logic in a page component, the integration is at the wrong layer.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Poll first; measure before adding a connection** | With `ETag`/304, an empty poll is ~150 bytes |
| **One connection per app, in a provider** | Per-component connections multiply everything |
| **Real-time updates the cache; components never subscribe directly** | Otherwise you maintain two copies of the data |
| **`useEventCallback` for the handler; connect with honest deps** | Or you reconnect every render (or read stale data) |
| **Cleanup closes the connection — StrictMode will catch a miss** | Doubling connections on every fast refresh |
| **Invalidate on reconnect** | You can't prove the replay covered the gap |
| **Guard applies with a version/sequence** | Events arrive out of order; the server says so explicitly |
| **Deduplicate and bound live lists** | At-least-once delivery guarantees duplicates; feeds grow forever |
| **Reconnect with exponential backoff and full jitter** | Otherwise every client reconnects in lockstep and re-kills the server |
| **Heartbeat and treat a missed pong as dead** | Intermediaries kill idle connections silently |
| **Never put a token in a WebSocket URL** | URLs are logged everywhere; use a cookie or first-message auth |
| **Ignore unknown event types** | The server will add some; don't break on the next deploy |
| **Show connection status with `aria-live`** | Silence reads as "broken" — and is invisible without it |
| **Pause when the tab is hidden** | Battery, bandwidth, and server cost |

---

## 9. Interview traps

**Q1. "How do you add real-time updates to a React app?"**
Ask which direction and which client first. Then the architectural answer: **one connection at the app root that updates the query cache** — components keep reading the cache and get updates for free, with no real-time code in any page. *"If I see a `useEffect` opening a socket inside a page component, the integration is at the wrong layer."*

**Q2. "SSE or WebSocket?"**
SSE for one-way server→client: plain HTTP, automatic reconnect with `Last-Event-ID` resume, works with existing auth and proxies, text only, **needs HTTP/2** because it permanently holds one of the six HTTP/1.1 connections. WebSocket when you also need low-latency client→server — and then you hand-write reconnection, backoff, heartbeats and auth.

**Q3. "Why does polling get dismissed too quickly?"**
It's stateless, works through every proxy, needs no reconnection logic, degrades gracefully, and with conditional requests an empty poll is tiny. The case against it is arithmetic — 10k clients at 0.5 Hz is 5k req/s. **Measure before adding a connection.**

**Q4. "Your SSE connection opens twice in development. Why?"**
StrictMode's double-invoke, checking that setup and cleanup are symmetrical. If closing the connection in cleanup fixes it, the effect was correct all along. **The wrong response is disabling StrictMode** — it would also double on fast refresh and real remounts.

**Q5. "Events arrive out of order. What breaks and how do you fix it?"**
A slow `payment.created` landing after `payment.succeeded` reverts the UI to "pending". Fix: compare a version or sequence number before applying, and ignore anything older. The server explicitly doesn't guarantee ordering ([API L23](../../API/05-beyond-rest/23-realtime-and-webhooks.md)) — this is how the client honours that.

**Q6. "What happens after a disconnect?"**
Reconnect with exponential backoff and **full jitter**, resume via `Last-Event-ID` if the server supports replay, and **invalidate the cache on reconnect regardless** — you can't prove the replay covered the whole gap. That last step makes the system self-healing.

**Q7. "You receive 500 events per second. What happens?"**
500 cache writes and up to 500 renders. Buffer events in a ref and flush on an interval or animation frame so it's one write and one render per batch. React's automatic batching helps within a tick, not across separate socket messages.

**Q8. "How do you authenticate a WebSocket?"**
Not with a header — the browser API can't set them. Cookie (sent on the handshake), auth-by-first-message with a close-on-timeout, or a short-lived single-use ticket fetched over REST. **Never a token in the URL** — it lands in logs, and the handshake isn't subject to CORS, so you must also validate `Origin`.

**Q9. "The same payment appears twice in the live feed."**
At-least-once delivery means duplicates are guaranteed, and you may also have fetched the item already. Deduplicate by ID before prepending, and bound the list length so a long-running feed doesn't grow without limit.

**Q10. "How do you avoid one connection per component?"**
A provider at the root owning the single connection, writing to the shared cache. Components subscribe to the *cache*, not the socket. Ten components opening their own streams means ten connections, ten reconnect loops, and every event handled ten times.

---

## 10. Build & break

### Build — the live payment feed
Implement `usePaymentStream` with `useEventCallback`, cache integration (write detail, invalidate lists and balance), reconnect-invalidation, sequence-gap detection, unknown-event tolerance, and a `ConnectionBadge` with `aria-live`.

Then verify by driving a mock SSE server:
- Emit `payment.succeeded` → the detail view updates **with no component knowing about the stream**
- Kill the server → the badge shows "Reconnecting…", backoff visible in the Network panel
- Restart it → reconnect, and a cache invalidation refetches everything
- Emit an unknown event type → ignored, nothing breaks
- Emit events out of order → the older one is discarded

### Build — the burst test
Emit 1,000 events in 2 seconds. Profile it. Then add the 250ms buffer-and-flush and profile again. **Record renders before and after** — it's usually 1,000 vs 8.

### Build — polling vs streaming, measured
Implement the pending-payment status two ways: `refetchInterval: 3000` with a terminal-state stop, and a dedicated subscription. Compare lines of code, failure modes, and requests per minute per client. **Then multiply by 10,000 clients** and decide which you'd actually ship.

### Break — five experiments
1. **Missing cleanup.** Remove `es.close()`. Fast-refresh five times and count open connections in the Network panel.
2. **Unstable handler.** Put the raw `onMessage` prop in the effect deps instead of `useEventCallback`. Watch it reconnect on every render.
3. **No jitter.** Simulate 50 clients reconnecting after a server restart with a fixed delay. Graph the request spike. Add full jitter and re-graph.
4. **No dedupe.** Emit the same event twice (which at-least-once delivery guarantees) and watch the row appear twice in the feed.
5. **Unbounded feed.** Run the feed for ten minutes at 10 events/second without slicing. Watch memory climb in the Performance panel.

### Explain out loud (60 seconds)
1. The decision tree for polling vs SSE vs WebSocket.
2. Why real-time should update the cache rather than component state.
3. Three things hand-rolled WebSocket code gets wrong.
4. Why you invalidate on reconnect.
5. How you handle out-of-order events and duplicates.

---

## Module 5 complete — checkpoint

- [ ] Client vs server state, by ownership
- [ ] Ten things `useEffect` + `fetch` doesn't do
- [ ] `staleTime` vs `gcTime`
- [ ] What goes in a query key, and why a factory
- [ ] How you'd choose `staleTime` per resource (including terminal states)
- [ ] Three layers of double-submit protection
- [ ] The five steps of an optimistic update, and what breaks without each
- [ ] When not to be optimistic
- [ ] `invalidateQueries` vs `setQueryData`, and the list-membership trap
- [ ] How to handle two concurrent edits
- [ ] Polling vs SSE vs WebSocket
- [ ] Why real-time updates the cache, not component state
- [ ] Out-of-order events, duplicates, and reconnect invalidation

---

## What's next

Module 6 is performance — and it starts where every performance conversation should: **measuring**, so you fix what's actually slow rather than what you assume is.

Next → **[Lesson 20: Why did it render? Measuring first](../06-performance/20-measuring.md)**
