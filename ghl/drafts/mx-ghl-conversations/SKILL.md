---
name: mx-ghl-conversations
description: GoHighLevel / HighLevel API v2 conversations and messaging for AI agents — send SMS/Email/WhatsApp via POST /conversations/messages, required type + contactId, email-specific fields (subject/html/emailFrom/attachments), conversation providers, inbound vs outbound, and adding inbound messages. Load when sending or receiving messages, building SMS/email automation, or a custom conversation provider in GHL/LeadConnector.
---

# GoHighLevel Conversations — Messaging for AI Coding Agents

**Loads when you send messages, read threads, or build a conversation provider in GoHighLevel.**

## When to also load
- `mx-ghl-core` — auth, Version header, rate limits (always)
- `mx-ghl-contacts` — every message targets a contactId
- `mx-ghl-webhooks` — receiving `InboundMessage` / `OutboundMessage` events

---

## Level 1: Patterns That Always Work (Beginner)

### 1. Send with `POST /conversations/messages` — `type` + `contactId` are required
```ts
✅ GOOD — SMS
await ghlFetch("/conversations/messages", {
  method: "POST",
  body: JSON.stringify({ type: "SMS", contactId, message: "Thanks — we got your request." }),
});
```

### 2. `type` is a fixed set of channels
`SMS`, `Email`, `WhatsApp`, `IG`, `FB`, `Live_Chat`, `Custom`. The channel you send on must have a configured provider (see Level 2).

### 3. Email needs more than `message`
```ts
✅ GOOD — Email
await ghlFetch("/conversations/messages", {
  method: "POST",
  body: JSON.stringify({
    type: "Email",
    contactId,
    subject: "Your proposal",
    html: "<p>Hi — the proposal is attached.</p>",   // or emailBody/message per your provider
    emailFrom: "sales@agency.com",
    attachments: ["https://.../proposal.pdf"],
  }),
});
```

### 4. You message a contact, not a raw phone/email
The API resolves the channel from the contact's phone/email. Ensure the contact exists with a valid E.164 phone (SMS) or email (Email) first — see `mx-ghl-contacts`.

### 5. A conversation is created/continued automatically
You don't pre-create a conversation to send. GHL threads the message onto the contact's conversation. Capture the returned `conversationId`/`messageId` for tracking.

---

## Level 2: Providers, Inbound & Threads (Intermediate)

### No provider → the send fails
Sending on a channel with no active provider (SMS phone number, email domain, WhatsApp, or a custom provider) returns an error. Verify the location has the channel configured before automating sends.

### Building a custom provider is a separate concern
The **Conversation Providers** marketplace module lets you back SMS/Email/Call with your own infrastructure. Your provider receives delivery requests from GHL and reports status back. That's an app-build task — pair this with `mx-ghl-marketplace`.

### Add an inbound message (log a message you received elsewhere)
The endpoint *is* the direction — there is no `direction` field. Provide `conversationId` **or** `contactId`. A `conversationProviderId` is required when you're using an additional custom SMS provider (not needed when replacing the default email/SMS provider).

```ts
✅ GOOD
await ghlFetch("/conversations/messages/inbound", {
  method: "POST",
  body: JSON.stringify({
    type: "SMS",
    contactId,                       // or conversationId
    message: "Customer reply text",
    conversationProviderId,          // required for custom SMS providers
  }),
});
```

### Read a thread with pagination
```ts
const { messages } = await ghlFetch(
  `/conversations/${conversationId}/messages?limit=100`
).then(r => r.json());
```

### Respect DND
If a contact is marked Do-Not-Disturb (globally or per channel), sends may be blocked or suppressed. Check/handle DND rather than retrying a suppressed send as if it were a transient error.

---

## Level 3: Reliable Messaging Automation (Advanced)

### De-duplicate outbound sends with your own key — claim BEFORE sending
GHL has no built-in idempotency key for sends. The dangerous case is a **timeout where the send actually delivered**: if you only mark-seen on a clean 2xx, a timeout throws before marking, the retry re-sends, and the customer is texted twice — the exact case dedup exists for. Claim the key first, then reconcile.

```ts
❌ BAD — only marks seen on success; a timeout after delivery double-texts
async function sendOnce(dedupeKey, payload) {
  if (await seen(dedupeKey)) return;
  const res = await ghlFetch("/conversations/messages", { method: "POST", body: JSON.stringify(payload) });
  if (res.ok) await markSeen(dedupeKey);        // never runs on timeout → retry re-sends
}

✅ GOOD — atomically claim first; release ONLY on a definite non-delivery
async function sendOnce(dedupeKey, payload) {
  if (!(await claim(dedupeKey))) return;        // atomic SETNX; already claimed → stop
  try {
    const res = await ghlFetch("/conversations/messages", { method: "POST", body: JSON.stringify(payload) });
    if (!res.ok && res.status < 500) await release(dedupeKey); // 4xx = definitely not sent → allow retry
    return res;                                  // timeout/5xx: keep the claim (ambiguous → don't re-send)
  } catch { /* network timeout: ambiguous — keep the claim, reconcile out-of-band */ }
}
```

### Pair inbound webhooks with outbound sends for two-way flows
Subscribe to `InboundMessage`, process, then reply via `/conversations/messages`. Keep reply logic idempotent per inbound `messageId` so webhook redelivery doesn't double-reply.

---

## Performance: Make It Fast

- **Throttle bulk campaigns** under the 100/10s burst; a blast of sends will 429 and drop messages.
- **Reuse the contact's existing conversation** — don't look up/rebuild thread state per message when you already have the `conversationId`.
- **Prefer templated content assembled locally** over multiple round-trips to fetch snippets per send.

```ts
❌ BAD — unthrottled blast, 429s past #100 in 10s
await Promise.all(list.map(c => send(c)));

✅ GOOD — bounded
const limit = pLimit(8);
await Promise.all(list.map(c => limit(() => send(c))));
```

## Observability: Know It's Working

- Track **send success vs error by channel**; a spike in errors on one channel usually means its provider broke or a number/domain lost verification.
- Log `messageId`/`conversationId` for every send to correlate with delivery/inbound webhooks.
- Monitor **DND-suppressed counts** separately from failures — they're expected, not errors.
- Watch for **duplicate sends** (same dedupeKey twice) as a signal your idempotency guard is bypassed.

---

## Enforcement: Anti-Rationalization Rules

### Rule 1: Set `type` and target a contactId
**You will be tempted to:** send a message with just a body and a phone number.
**Why that fails:** the endpoint requires `type` (channel) and resolves the recipient via `contactId`; a raw phone won't route.
**The right way:** ensure the contact exists, then send `{ type, contactId, message | email fields }`.

### Rule 2: Email is not SMS-with-a-subject
**You will be tempted to:** send Email with only `message` like an SMS.
**Why that fails:** email needs `subject` and body (`html`/`emailBody`) and often `emailFrom`; without them the send fails or arrives blank.
**The right way:** include subject, HTML body, and from-address for `type: "Email"`.

### Rule 3: A channel with no provider cannot send
**You will be tempted to:** assume a location can send SMS/WhatsApp because the API accepted the shape.
**Why that fails:** without a configured provider the send errors at delivery; you'll ship "working" code that silently sends nothing.
**The right way:** verify the location has the channel/provider configured (or build one via the Conversation Providers module) before automating.

### Rule 4: Add your own send idempotency
**You will be tempted to:** retry a failed/timed-out send by just calling the endpoint again.
**Why that fails:** the first send may have succeeded; there's no server idempotency key, so the customer gets texted twice.
**The right way:** atomically *claim* the dedupe key before sending; release it only on a definite 4xx non-delivery. On a timeout/5xx keep the claim (delivery is ambiguous) and reconcile out-of-band rather than blind-retrying.

### Rule 5: Honor DND
**You will be tempted to:** treat a suppressed/DND send as a transient failure and retry.
**Why that fails:** you'll hammer the API and, worse, risk compliance issues messaging opted-out contacts.
**The right way:** detect DND, skip (don't retry), and log it as suppressed rather than failed.
