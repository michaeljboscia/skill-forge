---
name: mx-ghl-payments
description: GoHighLevel / HighLevel API v2 payments for AI agents — reading orders, transactions, and subscriptions (mostly read-only), invoices, filtering transactions, payment scopes (payments/orders, payments/transactions, payments/subscriptions), reconciling amounts, and building a custom payment provider. Load when reporting on revenue, syncing orders/subscriptions, or integrating payments in GHL/LeadConnector.
---

# GoHighLevel Payments — Orders, Transactions & Invoices for AI Coding Agents

**Loads when you read orders/transactions/subscriptions, work with invoices, or build a payment provider in GoHighLevel.**

## When to also load
- `mx-ghl-core` — auth, Version header, rate limits, pagination (always)
- `mx-ghl-contacts` — payments link to a contactId
- `mx-ghl-marketplace` — building a custom payment provider app
- `mx-ghl-webhooks` — order/subscription/transaction events

---

## Level 1: Patterns That Always Work (Beginner)

### 1. Payments APIs are read-heavy — don't assume you can create orders
Available reads:

```
GET /payments/orders/            list orders
GET /payments/orders/{orderId}   one order
GET /payments/transactions/      list transactions
GET /payments/transactions/{id}  one transaction
GET /payments/subscriptions/     list subscriptions
GET /payments/subscriptions/{id} one subscription
```
Creating orders/subscriptions via API is largely **not exposed**. Invoices have more write capability than orders. Design around reading, not writing, orders.

### 2. Use the right scope or you get 401/403
| Resource | Scope |
|---|---|
| Orders | `payments/orders.readonly` (`.write` where supported) |
| Transactions | `payments/transactions.readonly` |
| Subscriptions | `payments/subscriptions.readonly` |

### 3. Everything is location-scoped and paginated
Pass `locationId` (or `altId`/`altType` where required) and paginate with `limit` + cursor (see `mx-ghl-core`). Don't fetch "all revenue" in one call.

### 4. Filter transactions server-side
```ts
const p = new URLSearchParams({
  locationId, limit: "100",
  startAt: "2026-07-01", endAt: "2026-07-31",
  paymentMode: "live",              // live vs test
  // status, contactId, subscriptionId, entityId also supported
});
const { data } = await ghlFetch(`/payments/transactions/?${p}`).then(r => r.json());
```

### 5. Money is money — never do float math on amounts
Treat amounts as integer minor units (cents) or a decimal library. `0.1 + 0.2 !== 0.3`.

---

## Level 2: Reconciliation & Correctness (Intermediate)

### Separate live from test payments
`paymentMode` distinguishes `live` and `test`. Mixing them corrupts revenue reports. Filter explicitly; never sum both.

### Amount + currency travel together
An amount without its currency is meaningless in a multi-currency agency. Carry `currency` alongside every amount and never sum across currencies.

### Subscription state is a lifecycle, not a boolean
A subscription can be `active`, `trialing`, `past_due`, `canceled`, etc. "Has a subscription" ≠ "is paying." Read the status field for entitlement decisions.

### Transactions belong to orders/subscriptions
Reconcile by linking transactions to their `orderId`/`subscriptionId`/`entityId` rather than treating each transaction as standalone; refunds and retries appear as separate transactions on the same entity.

---

## Level 3: Custom Payment Provider (Advanced)

### The Payments marketplace module lets you be the processor
Building a custom payment provider is an app-build task (pair with `mx-ghl-marketplace`): you register a provider, GHL sends you charge/verify requests, and you report status back. Handle:
- **Idempotent charge handling** — the same charge request can arrive more than once.
- **Signed callbacks** — verify inbound requests like webhooks (`mx-ghl-webhooks`).
- **Accurate status reporting** — success/failure/refund must reflect the real processor state.

### Reconcile nightly against your processor
Pull GHL transactions and your processor's ledger for the same window and diff. Silent divergence is where payment bugs hide.

---

## Performance: Make It Fast

- **Query date-bounded windows**, not the entire history, for reports; paginate and aggregate incrementally.
- **Cache subscription status** with a short TTL for entitlement checks instead of hitting the API per request.
- **Prefer webhooks** for new orders/transactions over polling the list endpoints, saving your daily 200k budget.

```ts
❌ BAD — poll every minute for new transactions
setInterval(() => ghlFetch(`/payments/transactions/?locationId=${loc}`), 60_000);

✅ GOOD — react to transaction/order webhooks; reconcile on a schedule
```

## Observability: Know It's Working

- Reconcile **GHL transaction totals vs your processor** per day, per currency; alert on any mismatch.
- Track **subscription state transitions** (active→past_due) to catch failed renewals early.
- Separate **live vs test** in every dashboard so test charges never inflate revenue.
- Log **refunds as first-class events**, not just negative transactions, so net revenue is correct.

---

## Enforcement: Anti-Rationalization Rules

### Rule 1: Don't invent create endpoints for orders/subscriptions
**You will be tempted to:** `POST /payments/orders` to sync a sale from another system.
**Why that fails:** create/write for orders and subscriptions is largely unavailable; you'll hit 404/405 and build on a fiction.
**The right way:** treat orders/subscriptions/transactions as read APIs; use invoices for billing you originate, and confirm any write endpoint exists before depending on it.

### Rule 2: No float math on money
**You will be tempted to:** sum amounts as JS numbers because it's convenient.
**Why that fails:** floating-point rounding makes totals off by cents; reconciliation fails and customers notice.
**The right way:** integer minor units or a decimal library; carry currency with every amount.

### Rule 3: Never sum live and test together
**You will be tempted to:** aggregate all transactions and report revenue.
**Why that fails:** test-mode charges inflate the number; leadership sees fake revenue.
**The right way:** filter `paymentMode: "live"` for revenue; report test separately.

### Rule 4: Subscription "exists" ≠ "paying"
**You will be tempted to:** grant access if a subscription record is present.
**Why that fails:** the sub may be `past_due`, `canceled`, or `trialing`; you'll entitle non-paying users or revoke paying ones.
**The right way:** read the status field and gate entitlement on `active`/`trialing` explicitly.

### Rule 5: Reconcile; don't trust one side
**You will be tempted to:** assume GHL's transaction list is the whole truth.
**Why that fails:** processor retries, refunds, and webhook gaps cause drift; unreconciled ledgers hide real money bugs.
**The right way:** diff GHL transactions against the processor ledger per window and currency, and alert on mismatch.
