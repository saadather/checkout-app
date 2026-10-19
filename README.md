# Checkout Session Demo

A small full-stack demo of a **resumable ticket checkout**: one server-owned session can be started on web, continued on another browser or phone, and completed safely even when inventory, price, or payment state changes mid-flow.

Stack:

- `checkout-server` — Express + TypeScript API
- `checkout-ui` — React + Vite client

---

## What you built and how to run it

### What it does

- Signs in with a **demo email** (no password) that scopes all sessions to that account
- Starts a checkout for a sample event listing (`listing-1`)
- Soft-holds inventory for the session TTL (default 10 minutes)
- Freezes an **agreed price** at session creation and requires explicit re-acceptance if the listing price changes
- Completes checkout into an order, with simulated success / failure / pending payment paths
- Resumes the same session from a shareable URL or the account dashboard

### Run locally

Requires Node.js 18+.

```bash
# Terminal 1 — API (http://localhost:3000)
cd checkout-server
npm install
npm run dev

# Terminal 2 — UI (Vite default: http://localhost:5173)
cd checkout-ui
npm install
npm run dev
```

Open the UI, enter a demo email, start checkout, and use the **Resume link** or demo controls on the checkout page to exercise edge cases.

Optional env on the server:

| Variable | Default | Meaning |
| --- | --- | --- |
| `PORT` | `3000` | API port |
| `SESSION_TTL_MS` | `600000` (10 min) | How long an open session stays valid |

---

## The checkout session state model

The server is the source of truth. Sessions live in memory as `CheckoutSession` records:

| Field | Role |
| --- | --- |
| `id` | Stable session id (also used in resume URLs) |
| `listingId` / `quantity` | What is being purchased |
| `agreedPrice` | Price the buyer last accepted (integer US cents) |
| `priceAccepted` | Whether the current listing price is approved |
| `status` | Lifecycle state (below) |
| `userEmail` | Owner; required to read or mutate the session |
| `createdAt` / `expiresAt` | TTL window |
| `orderId` | Set only after a successful completion |

### Statuses

```
ACTIVE ──► PAYMENT_PENDING ──► COMPLETED
   │              │
   │              └──► FAILED
   └──► EXPIRED
```

| Status | Meaning |
| --- | --- |
| `ACTIVE` | Open checkout; inventory soft-held; payment not submitted |
| `PAYMENT_PENDING` | Payment submitted, still unresolved; hold remains |
| `COMPLETED` | Order created; inventory decremented |
| `EXPIRED` | TTL elapsed while `ACTIVE` (or forced via demo endpoint) |
| `FAILED` | Non-retryable failure (e.g. payment failed while pending, or TTL while pending) |

**Soft inventory hold:** open sessions (`ACTIVE` / `PAYMENT_PENDING`) reduce `purchasable` for a listing without changing `available` until `complete` commits. Other buyers see lower purchasable capacity; the hold disappears when the session expires, fails, or completes.

**One open session per user + listing:** creating a second open checkout for the same email and listing returns `409 CHECKOUT_ALREADY_OPEN` with the existing `sessionId`.

Mutations that touch inventory or price run under a **per-listing promise chain** so concurrent completes / accepts / inventory updates for the same listing serialize.

---

## How web and mobile resume the same session

There is one session record on the server. Web and mobile are just different clients of that record.

1. **Identity** — The client stores the demo email in `localStorage` and sends it as `X-Demo-Email` on every API call. The server only returns sessions owned by that email.
2. **Resume URL** — Checkout updates the address bar to `/?session=<sessionId>` and shows a copyable resume link. Opening that URL on another device (or browser) loads the same session after signing in with the **same** demo email.
3. **Dashboard** — `GET /api/checkout-sessions` lists all sessions for the signed-in email so you can resume without the link.
4. **Sync** — While a checkout is open, the UI polls the session every few seconds so a second device sees price, inventory, and status changes without starting a new flow.
5. **No client-side cart as truth** — Quantity, price acceptance, status, and order id all come from the server. Refreshing or switching devices never forks a second cart for the same open session.

If you open a resume link while signed in as a different email, the UI refuses and keeps you on your own account.

---

## Stale inventory, price changes, and duplicate completion

### Stale / contended inventory

- At session create and again at complete, the server computes `purchasable = available - heldByOtherOpenSessions`.
- If `purchasable < quantity`, it returns `409 INVENTORY_UNAVAILABLE` and does **not** create an order.
- Demo controls can force sold-out / restore inventory; open sessions re-sync via poll and block completion when short.

### Price changes

- Creating a session snapshots `agreedPrice` from the listing and sets `priceAccepted: true`.
- Completing requires `priceAccepted` and `agreedPrice === listing.price`. If the listing moved, the server clears acceptance and returns `409 PRICE_CHANGED` with `currentPrice`.
- The buyer must call `POST .../accept-price` with `expectedPrice` matching the current listing price (compare-and-set). A race that changes price again fails with `PRICE_CHANGED` instead of silently accepting a stale number.

### Duplicate completion

- `POST .../complete` is **idempotent** for a finished session: if status is already `COMPLETED` and an `orderId` exists, the same session + order are returned again. A double-submit or retry from a second device does not sell inventory twice.
- Simulated payment pending moves the session to `PAYMENT_PENDING` without creating an order; a later successful complete commits once under the listing lock.

---

## Tradeoffs

| Choice | Why | Cost |
| --- | --- | --- |
| Soft holds (count open sessions) vs hard stock rows | Simple model that still prevents oversell in one process | No durable reservation; no multi-node consistency |
| Simulated payment outcomes | Exercises pending / failure / retry paths without a PSP | No webhooks, captures, or real money movement |
| Polling for multi-device sync | Easy and good enough for a demo | Extra latency and load vs push |

---

## What you'd do differently with more time

- **Real auth** — OAuth / magic links instead of a spoofable demo header; short-lived session tokens on resume links.
- **Real payments** — Stripe (or similar) PaymentIntents, idempotency keys from the client, and webhook-driven transitions out of `PAYMENT_PENDING`.
- **Stronger sync** — WebSockets or SSE so web and mobile update immediately when another device accepts a price or completes.
- **State management** — MobX (or similar) for shared checkout/session state instead of prop-drilling and ad hoc local state.
- **Custom hooks** — Extract API, polling, and session-resume logic into reusable hooks so the UI stays thin and testable.
- **Tests** — Unit tests for pricing/session transitions, component tests for the checkout flow, and a few end-to-end paths (accept offer → pay → complete).
- **Accessibility** — Keyboard-friendly flows, proper labels/roles, focus management on step changes, and checks with axe / screen readers.
