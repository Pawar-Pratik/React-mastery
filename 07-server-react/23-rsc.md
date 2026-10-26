# Lesson 23 — React Server Components

> **Why this lesson exists:** RSC is the biggest architectural change to React since hooks, and *"what is a Server Component?"* is a question most candidates answer with vibes — "it runs on the server" — which doesn't explain why it's different from SSR, what actually crosses the network, or why you can't use `useState` in one. This lesson gives you the mechanism, so you can reason about RSC rather than memorise rules.

**Time:** ~70 minutes · **Prereq:** Modules 1–6

---

## 1. The idea in one sentence

> **A Server Component runs *once*, on the server, and its output is a serialized description of UI — not HTML, and never any JavaScript — which means its code, its dependencies and its data access never reach the browser.**

The two words that matter: **once** (no re-rendering, no interactivity) and **never** (the code isn't in the bundle at all).

---

## 2. RSC vs SSR — the distinction everyone fumbles

They sound similar and are completely different mechanisms.

| | **SSR** | **RSC** |
|---|---|---|
| What runs on the server | The component, to produce HTML | The component, to produce a UI description |
| Does the component's JS ship to the browser? | **Yes** — it must hydrate | **No** — never |
| Runs again on the client? | **Yes**, during hydration | **Never** |
| Can hold state / handlers | Yes (after hydration) | **No** |
| Output | HTML string | **RSC payload** (a serialized tree) |
| Purpose | Faster first paint, SEO | **Less JavaScript**, direct data access |

**The clearest way to say it:** *"SSR renders your components to HTML on the server and then ships the same components to the browser to hydrate them. RSC renders components on the server and ships only their **output** — the component code never goes to the client, so it isn't in the bundle and it never runs again."*

They're **complementary**, not alternatives: a Next.js app uses RSC to decide what's in the bundle and SSR to produce the initial HTML.

---

## 3. What actually crosses the network

Not HTML. A serialized tree — the **RSC payload** (sometimes called the Flight format):

```
1:"$Sreact.suspense"
2:I["./chunks/RefundButton.js","RefundButton"]      ← a reference to a CLIENT component
0:["$","div",null,{"className":"page","children":[
    ["$","h1",null,{"children":"Payment pi_3Nx8"}],
    ["$","p",null,{"children":"$49.99"}],
    ["$","$L2",null,{"paymentId":"pi_3Nx8"}]        ← a placeholder + its serialized props
  ]}]
```

Read what's in there:
- **Host elements** (`div`, `h1`) with their props, already resolved
- **Client component references** (`$L2`) — a module id plus **serialized props**, to be hydrated in the browser
- **No server component code.** The function that produced this is gone.

**Two consequences that explain most RSC rules:**

1. **Props crossing the boundary must be serializable.** You can pass strings, numbers, objects, arrays, Dates, Maps, Sets, promises and JSX. You **cannot** pass functions, class instances, or anything with behaviour — there's nothing to serialize.
2. **The payload is streamed.** Chunks arrive as they're ready, which is what makes Suspense-based streaming work ([Lesson 25](25-ssr-streaming-hydration.md)).

---

## 4. Server and Client Components

```tsx
// app/payments/[id]/page.tsx — a Server Component by DEFAULT in the App Router
import { db } from "@/lib/db";
import { RefundButton } from "./refund-button";

export default async function PaymentPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const payment = await db.payments.findById(id);      // ← direct DB access. No API route.

  return (
    <div>
      <h1>Payment {payment.id}</h1>
      <p>{formatMoney(payment.money)}</p>
      <RefundButton paymentId={payment.id} />           {/* ← a Client Component */}
    </div>
  );
}
```

```tsx
// app/payments/[id]/refund-button.tsx
"use client";                                           // ← the boundary marker

import { useState } from "react";

export function RefundButton({ paymentId }: { paymentId: string }) {
  const [open, setOpen] = useState(false);              // state → must be a Client Component
  return <>
    <button onClick={() => setOpen(true)}>Refund</button>
    {open && <RefundModal paymentId={paymentId} onClose={() => setOpen(false)} />}
  </>;
}
```

### The capability split

| | Server Component | Client Component |
|---|---|---|
| `async`/`await` at the top level | ✅ | ❌ |
| Direct DB / filesystem / secrets access | ✅ | ❌ |
| `useState`, `useReducer`, `useEffect` | ❌ | ✅ |
| Event handlers (`onClick`) | ❌ | ✅ |
| Browser APIs (`window`, `localStorage`) | ❌ | ✅ |
| Context — `useContext` | ❌ | ✅ |
| Ships JavaScript to the browser | **No** | Yes |
| Re-renders | **Never** | Yes |

**`"use client"` marks a boundary, not a file.** Everything imported *from* a client component becomes part of the client bundle too — so one `"use client"` at the top of your tree makes the entire app a client app. Push the directive **down**, as far toward the leaves as possible.

```tsx
// ❌ The whole page becomes client code
"use client";
export default function PaymentPage() {
  const [open, setOpen] = useState(false);
  return <div>{/* everything below is now in the bundle */}</div>;
}

// ✅ Only the interactive leaf ships
export default async function PaymentPage() {     // server
  const payment = await db.payments.findById(id);
  return <div>{/* server-rendered */}<RefundButton /></div>;   // one small client island
}
```

### The pattern that unlocks everything: Server Components as children

```tsx
// ❌ A Server Component imported INTO a Client Component becomes client code
"use client";
import { ServerChart } from "./server-chart";      // ← no longer a Server Component

// ✅ Pass it as children — it's rendered on the SERVER and passed through
export default async function Page() {             // server
  return (
    <ClientLayout>                                  {/* client */}
      <ServerChart data={await getChartData()} />   {/* stays on the server! */}
    </ClientLayout>
  );
}

// client-layout.tsx
"use client";
export function ClientLayout({ children }: { children: React.ReactNode }) {
  const [collapsed, setCollapsed] = useState(false);
  return <div className={collapsed ? "collapsed" : ""}>{children}</div>;
}
```

**This is the single most important RSC pattern.** A Client Component can *render* Server Components passed as `children` or props, because by then they're already-rendered output, not code to import. It's how you get an interactive shell around server-rendered content — and it's the same composition trick as [Lesson 03](../01-mental-model/03-rendering-and-commit.md)'s children-passing, now operating across the network boundary.

---

## 5. What RSC actually buys you

### 1. Less JavaScript
```tsx
// A markdown renderer, a date library, a syntax highlighter — 300KB of dependencies
import { marked } from "marked";
import hljs from "highlight.js";

export default async function Article({ slug }: Props) {
  const md = await fs.readFile(`content/${slug}.md`, "utf8");
  return <div dangerouslySetInnerHTML={{ __html: marked(md) }} />;
}
// Bundle impact: ZERO. The libraries run on the server and never ship.
```
**That's the headline benefit**, and it's structural rather than an optimisation: heavy dependencies that only transform data at render time simply leave the client bundle.

### 2. Data access without an API layer
```tsx
export default async function Page() {
  const payments = await db.payments.list({ merchantId });    // straight to the database
  return <PaymentsTable payments={payments} />;
}
```
No fetch, no API route, no serialization boundary to design, no client-side loading state. **And the caveat that matters:** you still need real authorization here ([API Lesson 15](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/15-authorization-and-multitenancy.md)) — `merchantId` must come from the verified session, never from a param. RSC removes the HTTP layer; it doesn't remove the security requirements.

### 3. No client/server waterfall
```
❌ Client fetching:  HTML → JS → hydrate → fetch → render
✅ RSC:              server fetches (fast, co-located with the DB) → HTML with data
```
The server is milliseconds from the database; the browser is 50–200ms away. Moving the fetch server-side removes a full round trip from the critical path — the same waterfall argument as route loaders ([Lesson 16](../04-composition/16-routing-and-architecture.md)), taken further.

### 4. Secrets stay on the server
```tsx
export default async function Page() {
  const data = await fetch("https://api.internal/x", {
    headers: { Authorization: `Bearer ${process.env.INTERNAL_TOKEN}` },   // never shipped
  }).then(r => r.json());
}
```

---

## 6. Server Actions

The write path — functions that run on the server, callable from the client.

```tsx
// app/payments/actions.ts
"use server";                                            // ← every export becomes a server endpoint

import { revalidatePath } from "next/cache";
import { auth } from "@/lib/auth";

export async function refundPayment(paymentId: string, formData: FormData) {
  // ★ SECURITY: this is a PUBLIC HTTP ENDPOINT. Authenticate and authorize here.
  const session = await auth();
  if (!session) throw new Error("Unauthorized");

  const parsed = RefundSchema.safeParse(Object.fromEntries(formData));   // validate — always
  if (!parsed.success) return { error: "invalid_input", fields: fieldErrors(parsed.error) };

  // Tenant scope from the SESSION, never from an argument (API Lesson 15)
  const payment = await db.payments.findById(paymentId, session.merchantId);
  if (!payment) return { error: "not_found" };

  const refund = await db.refunds.create({ ... });
  revalidatePath(`/payments/${paymentId}`);              // invalidate the server cache
  return { ok: true, refund };
}
```

```tsx
// Used from a form — works WITHOUT JavaScript (progressive enhancement)
<form action={refundPayment.bind(null, payment.id)}>
  <input name="amount" defaultValue={payment.money.amountMinor / 100} />
  <SubmitButton />
</form>

// Or with useActionState for pending state and errors (Lesson 14)
const [state, formAction, isPending] = useActionState(refundPayment, { status: "idle" });
```

> **The security point is the one to lead with in an interview:** *"a Server Action is a public HTTP endpoint with a generated ID. Anyone can call it with any arguments. So every action must authenticate, authorize, and validate its inputs exactly like a REST endpoint — and the tenant scope must come from the session, never from a parameter. The `"use server"` directive is not a security boundary; it's a transport mechanism."*

That's [API Lesson 15](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/15-authorization-and-multitenancy.md)'s BOLA discussion, applied to a new transport. The vulnerability is identical; only the syntax changed.

**Other practical notes:** actions are serial by default (Next.js dispatches them one at a time per client), arguments must be serializable, and they can't return non-serializable values. `updateTag`, `revalidateTag` and `revalidatePath` are how you tell the server cache that data changed — the RSC equivalent of `invalidateQueries` ([Lesson 18](../05-data/18-mutations-and-optimistic.md)). Which one to call is [Lesson 24](24-nextjs-app-router.md)'s job.

---

## 7. The mental model for structuring an RSC app

```
┌─────────────────────────────────────────────────┐
│ SERVER (default)                                 │
│  · data fetching, DB access, secrets              │
│  · heavy dependencies (markdown, syntax, i18n)    │
│  · static content, layout, formatting             │
│                                                   │
│   ┌───────────────────────────────────────┐      │
│   │ CLIENT ISLANDS  ("use client")         │      │
│   │  · anything with state or handlers      │      │
│   │  · browser APIs                          │      │
│   │  · third-party interactive libraries     │      │
│   │                                           │      │
│   │   ┌─────────────────────────────────┐   │      │
│   │   │ SERVER CONTENT via children ✅   │   │      │
│   │   └─────────────────────────────────┘   │      │
│   └───────────────────────────────────────┘      │
└─────────────────────────────────────────────────┘
```

**The rule: server by default, client at the leaves, server content threaded back through as children.**

```tsx
// Ledger Console, if it were RSC-based
app/
├── layout.tsx                    server — shell, nav
├── payments/
│   ├── page.tsx                  server — fetches the first page directly from the DB
│   ├── payments-table.tsx        "use client" — virtualization, selection, sorting
│   ├── filter-bar.tsx            "use client" — inputs
│   ├── refund-button.tsx         "use client" — a modal
│   └── actions.ts                "use server" — refund, capture, cancel
└── reports/
    └── page.tsx                  server — heavy aggregation, zero client JS
```

**Note what stays on the client:** the virtualized table (it needs scroll state and measurement), the filter inputs, the modal. **Note what moves to the server:** the initial data fetch, the reports aggregation, and every formatting/markdown/i18n dependency. That split is the whole design.

---

## 8. Where RSC doesn't help

Be honest about this; interviewers respect it and it's genuinely true.

| Situation | Why RSC doesn't help |
|---|---|
| **A highly interactive dashboard behind a login** | Almost everything needs state — it's client components with extra ceremony |
| **No server to run** | Static hosting, Electron, a Chrome extension |
| **Realtime-heavy UIs** | Live data is a client-cache problem ([Lesson 19](../05-data/19-realtime.md)) |
| **You already have a good API + query layer** | The waterfall is already handled by loaders/prefetching |
| **Small apps** | The mental overhead exceeds the benefit |

**The cost side, stated plainly:** you now have two component models with different rules, a serialization boundary to think about, harder local reasoning ("why is this in the bundle?"), a server you must run and scale, and an ecosystem where many libraries still need `"use client"` wrappers. The tooling is still maturing outside Next.js.

> **For Ledger Console specifically: CSR is the right call.** It's an authenticated dashboard — nothing to index, no anonymous first paint, and nearly every surface is interactive. **Being able to say "RSC wouldn't help here, and here's why" is a stronger answer than adopting it reflexively.**

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Server by default; `"use client"` as far down the tree as possible** | The directive marks a boundary — everything below it ships |
| **Pass Server Components as `children` to Client Components** | The only way to keep server content out of the bundle inside an interactive shell |
| **Props crossing the boundary must be serializable** | No functions, no class instances |
| **Treat every Server Action as a public HTTP endpoint** | Authenticate, authorize and validate — `"use server"` is transport, not security |
| **Tenant scope from the session, never from an argument** | Same BOLA rule as any API |
| **Don't put secrets in files that could become client code** | One `"use client"` upstream changes what ships |
| **After a mutation, `updateTag` (or `revalidatePath`) so the user sees their own write** | The server-cache equivalent of invalidating a query. `revalidateTag(tag, "max")` serves the old value on the next read |
| **Keep heavy render-time dependencies on the server** | Markdown, syntax highlighting, i18n data — zero bundle cost |
| **Don't adopt RSC for a fully-interactive authenticated app** | It's client components with extra ceremony |

---

## 10. Interview traps

**Q1. "What is a Server Component?"**
Lead with the mechanism, not "it runs on the server": *"a component that runs once on the server and ships only its serialized output — a UI description, not HTML and not JavaScript. Its code never reaches the browser, so its dependencies aren't in the bundle, and it never re-renders."*

**Q2. "RSC vs SSR?"**
SSR renders components to HTML **and still ships them** to hydrate. RSC renders on the server and ships only the output — the component code never goes to the client. Different problems: SSR is about first paint and SEO; RSC is about bundle size and data access. **They're complementary and a Next.js app uses both.**

**Q3. "Why can't a Server Component use `useState`?"**
Because it runs once and is then gone. State implies re-rendering in response to interaction, and there's no component on the client to re-render — only its output. Same reason there are no event handlers: a function can't be serialized.

**Q4. "What can you pass across the boundary?"**
Serializable values: primitives, plain objects, arrays, Dates, Maps, Sets, promises, JSX. **Not** functions, class instances, or anything with behaviour — the payload is serialized, so there's nothing to send.

**Q5. "How do you keep a Server Component inside a Client Component?"**
Pass it as `children` or a prop, never import it. An import makes it part of the client module graph; `children` passes already-rendered output through. **This is the most important RSC pattern**, and it's the same composition trick as children-passing for re-render avoidance.

**Q6. "Is `"use server"` a security boundary?"**
**No** — it's the opposite. It creates a public HTTP endpoint with a generated ID that anyone can call with any arguments. Every action must authenticate, authorize and validate exactly like a REST endpoint, and tenant scope must come from the session. **This is the highest-value RSC security point**, and it's the BOLA problem with new syntax.

**Q7. "What does RSC actually buy you?"**
Four things: heavy dependencies leave the bundle entirely, data fetching happens next to the database instead of across the network, the client/server waterfall disappears, and secrets stay server-side. **The bundle one is structural** — not an optimisation you tune, but code that's simply never shipped.

**Q8. "When would you not use RSC?"**
A fully-interactive authenticated dashboard (almost everything needs state), no server to run, realtime-heavy UIs, or when you already have a good API with a query layer and prefetching. **Say what it costs too:** two component models, a serialization boundary, harder local reasoning, and a server to operate.

**Q9. "How do you invalidate data after a Server Action?"**
With the server-cache equivalents of `invalidateQueries`, picked by what the user should see next. In Next 16, **`updateTag(tag)`** is the call for the user's own write: it expires the tag, and the action's response carries the re-rendered page. **`revalidateTag(tag, "max")`** is stale-while-revalidate, so the next read still gets the old value: right for a webhook, wrong straight after a refund. **`revalidatePath`** is the simpler path-based option; tags are finer-grained. And if a client query cache also holds that data, invalidate it there too ([Lesson 24](24-nextjs-app-router.md) has the full model).

**Q10. "What's in the RSC payload?"**
A serialized tree: resolved host elements with props, and references to client components (module id + serialized props) that the browser will hydrate. **Not HTML, and not the server components' code.** It streams, which is what makes Suspense-based progressive rendering work.

---

## 11. Build & break

### Build — see the payload
```bash
npx create-next-app@latest rsc-lab --typescript --app
```
From Next 16.4 this app has **Cache Components** on, so any Server Component in these labs that awaits uncached data (a `fetch`, a database query) needs a `<Suspense>` boundary above it. Without one, the dev overlay flags it and `next build` fails ([Lesson 24 §3](24-nextjs-app-router.md#3-the-four-caches) explains why).

Create a server page with a `<Suspense>` boundary around a Server Component that fetches data and renders a client button, passing it some of that data as props. Then open the Network panel, find the RSC request (`?_rsc=`), and **read the raw response**. Find: the host elements, the client component reference, its serialized props — and confirm the server component's code isn't there.

**That five-minute exercise removes most RSC confusion**, the same way reading a React element object did in [Lesson 01](../01-mental-model/01-how-react-works.md).

### Build — the bundle proof
1. Build a page that imports `marked` + `highlight.js` and renders markdown in a **Server** Component. Check the bundle size.
2. Add `"use client"` to the top of that file. Rebuild. **Compare.**
3. Revert, and instead pass the rendered output as `children` into a client shell. Check again.

The three numbers make the "server by default, client at the leaves" rule concrete.

### Build — a Server Action, attacked
Write a `refundPayment` action. Then **call it directly** from the browser console or `curl` using the action ID from the network tab, with a `paymentId` belonging to another tenant.
- Without the session check: it works. **That's the BOLA vulnerability.**
- With `session.merchantId` scoping: 404.

**Do this once.** It's the most persuasive demonstration that `"use server"` isn't a security boundary.

### Break — four experiments
1. **Boundary pollution.** Put `"use client"` at the top of a layout and watch the entire subtree enter the bundle.
2. **Non-serializable prop.** Pass a function from a Server to a Client Component. Read the error.
3. **Importing a server component into a client one.** Watch it silently become client code (check the bundle).
4. **A secret in a client file.** Reference `process.env.SECRET` in a `"use client"` file and see what Next.js does (it's `undefined` unless prefixed `NEXT_PUBLIC_` — and if you prefix it, **it's in the bundle**).

### Explain out loud (90 seconds)
1. What a Server Component is, mechanically.
2. RSC vs SSR, and why they're complementary.
3. What crosses the boundary, and what can't.
4. The children pattern and why it matters.
5. Why a Server Action needs the same authorization as a REST endpoint.

---

## What's next

RSC is the model. Next.js App Router is where you actually meet it — along with the caching layers that cause the most confusion in modern React, and the routing conventions you'll be asked about.

Next → **[Lesson 24: Next.js App Router](24-nextjs-app-router.md)**
