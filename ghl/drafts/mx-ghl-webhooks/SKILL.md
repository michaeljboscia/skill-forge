---
name: mx-ghl-webhooks
description: GoHighLevel / HighLevel API v2 webhooks for AI agents — verifying x-ghl-signature (Ed25519) and legacy x-wh-signature (RSA-SHA256) against the RAW request body, event types (ContactCreate, InboundMessage, OpportunityStatusUpdate, AppointmentCreate…), idempotent receivers, GHL's retry-only-on-429 quirk, and fast 2xx-then-async processing. Load when receiving or verifying GHL/LeadConnector webhook events.
---

# GoHighLevel Webhooks — Event Receivers for AI Coding Agents

**Loads when you receive, verify, or process GoHighLevel webhook events.**

## When to also load
- `mx-ghl-core` — auth, rate limits (your handler may call back into the API)
- `mx-ghl-marketplace` — subscribing an app to events, app config
- `mx-ghl-conversations` / `-contacts` / `-opportunities` — acting on specific events

---

## Level 1: Patterns That Always Work (Beginner)

### 1. Verify the signature against the RAW body — capture bytes before parsing
Any re-serialization (parse → stringify) changes bytes and breaks the signature.

```ts
import express from "express";
const app = express();

// Capture the raw body for the webhook route BEFORE JSON parsing
app.use("/webhooks/ghl", express.raw({ type: "application/json" }));

app.post("/webhooks/ghl", (req, res) => {
  const raw = req.body;                       // Buffer of exact bytes
  if (!verifyGhl(raw, req.header("x-ghl-signature"))) return res.sendStatus(401);
  const event = JSON.parse(raw.toString("utf8"));
  // ... enqueue, then 2xx
  res.sendStatus(200);
});
```

### 2. Prefer the new `x-ghl-signature` (Ed25519); `x-wh-signature` (RSA) is deprecated
The legacy RSA header `x-wh-signature` is **deprecated 1 July 2026**. New receivers verify `x-ghl-signature` with GHL's Ed25519 public key.

```ts
import { verify } from "node:crypto";   // Ed25519 via KeyObject

function verifyGhl(raw: Buffer, sigB64: string | undefined): boolean {
  if (!sigB64) return false;
  const pub = crypto.createPublicKey(process.env.GHL_ED25519_PUBLIC_KEY!); // PEM/SPKI
  return crypto.verify(null, raw, pub, Buffer.from(sigB64, "base64"));      // null algo = Ed25519
}
```

### 3. Return 2xx fast; do the work asynchronously
Acknowledge in milliseconds, then process off the request path.

```ts
res.sendStatus(200);              // ack first
queue.add(event);                // process later (worker)
```

### 4. Make the receiver idempotent
Deduplicate on the event's unique id (e.g. `webhookId`/`id`) so redelivery doesn't double-process.

```ts
if (await seen(event.webhookId)) return res.sendStatus(200);
await markSeen(event.webhookId);
```

### 5. The event `type` tells you what happened
Common types: `ContactCreate/Update/Delete/DndUpdate`, `InboundMessage`, `OutboundMessage`, `ConversationUnread`, `OpportunityCreate/Update/StatusUpdate/StageUpdate`, `AppointmentCreate/Update/Delete`, `TaskCreate/Complete`. Switch on `event.type`.

---

## Level 2: GHL's Retry Model — The Trap (Intermediate)

### GHL currently retries ONLY on HTTP 429 from your endpoint
Unlike Stripe, GHL does **not** retry on your 5xx or timeouts. If your handler throws a 500 or times out, **the event is lost forever.**

| Your response | GHL behavior | Consequence |
|---|---|---|
| 2xx | delivered, done | ✅ |
| 429 | retried later | ✅ (backpressure works) |
| 5xx / timeout | **no retry** | ❌ event lost |

**Design rule:** never let real work throw inside the request. Verify → enqueue → 2xx. Do fallible processing in a worker where *you* control retries and a dead-letter queue.

```ts
❌ BAD — DB write inline; if it throws, GHL won't resend
app.post("/webhooks/ghl", async (req, res) => {
  await db.save(parse(req));         // throws → 500 → event gone
  res.sendStatus(200);
});

✅ GOOD — ack, then durable queue with your own retry
app.post("/webhooks/ghl", (req, res) => {
  if (!verifyGhl(req.body, req.header("x-ghl-signature"))) return res.sendStatus(401);
  queue.add(JSON.parse(req.body.toString()));   // durable
  res.sendStatus(200);
});
```

### If you're overwhelmed, return 429 on purpose
Because 429 is the one retryable code, you can shed load safely: when your queue is saturated, respond 429 and GHL will redeliver.

### Support both headers during the transition
Until 1 July 2026, verify `x-ghl-signature` if present, else fall back to `x-wh-signature` (RSA-SHA256) so older subscriptions keep working.

---

## Level 3: Robust Event Pipelines (Advanced)

### RSA fallback (legacy `x-wh-signature`)
```ts
function verifyLegacy(raw: Buffer, sigB64?: string): boolean {
  if (!sigB64) return false;
  const v = crypto.createVerify("RSA-SHA256");
  v.update(raw);                                   // raw bytes
  v.end();
  return v.verify(process.env.GHL_RSA_PUBLIC_KEY!, Buffer.from(sigB64, "base64"));
}

function verifyAny(raw: Buffer, req): boolean {
  const ed = req.header("x-ghl-signature");
  if (ed) return verifyGhl(raw, ed);
  return verifyLegacy(raw, req.header("x-wh-signature"));
}
```

### Use the SDK's helper if you're already on it
```ts
app.use("/webhooks/ghl", ghl.webhooks.subscribe());   // sets req.isSignatureValid
app.post("/webhooks/ghl", (req, res) => {
  if (!req.isSignatureValid) return res.sendStatus(401);
  queue.add(req.body); res.json({ success: true });
});
// env: WEBHOOK_SIGNATURE_PUBLIC_KEY (Ed25519), WEBHOOK_PUBLIC_KEY (RSA)
```

### Order is not guaranteed
Events can arrive out of order (e.g. `Update` before `Create` under retries). Reconcile against current API state when order matters rather than assuming sequence.

---

## Performance: Make It Fast

- **Handler latency = milliseconds.** Signature verify + enqueue only. Slow handlers cause timeouts → lost events (no retry).
- **Verify before parsing**; reject bad signatures without doing any work.
- **Scale workers, not the handler.** The HTTP endpoint should be trivial; concurrency lives in the queue consumers.

## Observability: Know It's Working

- Count **signature failures** — a spike means a wrong/rotated public key or a body-parsing regression (you're re-stringifying).
- Track **handler p99 latency**; anything approaching GHL's delivery timeout risks silent event loss.
- Monitor **dead-letter queue depth** — that's where your worker retries land; it's your real failure signal since GHL won't resend.
- Log **duplicate event ids caught** to confirm idempotency is doing its job.

---

## Enforcement: Anti-Rationalization Rules

### Rule 1: Verify against the RAW body
**You will be tempted to:** run `express.json()` globally and verify against `JSON.stringify(req.body)`.
**Why that fails:** key order and whitespace change, so the recomputed bytes never match the signature — every event 401s (or you disable verification and accept spoofed events).
**The right way:** capture the raw Buffer for the webhook route (`express.raw`), verify, then parse.

### Rule 2: Ack fast, process later
**You will be tempted to:** do the database write / API callout inline and then return 200.
**Why that fails:** if that work throws or is slow, GHL sees a 5xx/timeout and — because it only retries 429 — the event is gone permanently.
**The right way:** verify → enqueue to durable storage → return 2xx immediately; process in a worker with your own retry + DLQ.

### Rule 3: Don't rely on GHL to retry your failures
**You will be tempted to:** assume "webhooks retry" like Stripe and let transient errors bubble as 500s.
**Why that fails:** GHL currently retries only on 429; every 5xx is a dropped event with no second chance.
**The right way:** return 429 to shed load (retryable), never 5xx for recoverable work; own reliability in your queue.

### Rule 4: Idempotent receivers, always
**You will be tempted to:** process each delivery as unique.
**Why that fails:** 429-driven redelivery and at-least-once semantics mean the same event can arrive twice → double charges, double replies.
**The right way:** dedupe on the event id before acting; make handlers safe to run twice.

### Rule 5: Migrate to Ed25519 before July 2026
**You will be tempted to:** implement only `x-wh-signature` (RSA) because tutorials show it.
**Why that fails:** RSA `x-wh-signature` is deprecated 1 July 2026; after that events are signed only with `x-ghl-signature` and RSA-only receivers reject everything.
**The right way:** verify `x-ghl-signature` (Ed25519) first, keep RSA as a temporary fallback.
