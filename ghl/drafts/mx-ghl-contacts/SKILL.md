---
name: mx-ghl-contacts
description: GoHighLevel / HighLevel API v2 contacts, custom fields, tags, and notes for AI agents — create vs update vs upsert, duplicate detection by email/phone, the customFields array format ({id, field_value}), per-location field IDs, tags, DND, and safe bulk sync. Load when reading, writing, deduplicating, or syncing GHL/LeadConnector contacts.
---

# GoHighLevel Contacts — CRM Records for AI Coding Agents

**Loads when you create, update, upsert, tag, or sync GoHighLevel contacts and their custom fields.**

## When to also load
- `mx-ghl-core` — auth, Version header, rate limits, pagination (always)
- `mx-ghl-opportunities` — attaching opportunities to a contact
- `mx-ghl-conversations` — messaging a contact

---

## Level 1: Patterns That Always Work (Beginner)

### 1. Upsert instead of blind create when syncing external data
`POST /contacts/` always creates and can duplicate. `POST /contacts/upsert` respects the location's duplicate rules and updates the match.

```ts
❌ BAD — re-running your sync makes duplicates
await ghlFetch("/contacts/", { method: "POST", body: JSON.stringify({ locationId, email }) });

✅ GOOD — idempotent on email/phone
await ghlFetch("/contacts/upsert", {
  method: "POST",
  body: JSON.stringify({ locationId, email, phone, firstName, lastName }),
});
```

### 2. Custom fields are an array of `{id, field_value}`, not top-level keys
```ts
❌ BAD — custom field names are not contact properties
body: JSON.stringify({ locationId, email, "External ID": "ERP-99" })  // ignored / 422

✅ GOOD
body: JSON.stringify({
  locationId, email,
  customFields: [{ id: "aBcD1234fieldId", field_value: "ERP-99" }],
})
```

### 3. Resolve custom-field IDs at runtime — they are per-location
Field IDs differ across sub-accounts. Fetch and map by key/name once, then cache.

```ts
const { customFields } = await ghlFetch(`/locations/${locationId}/customFields`).then(r => r.json());
const byKey = Object.fromEntries(customFields.map(f => [f.fieldKey ?? f.name, f.id]));
// use byKey["contact.external_id"] as the id
```

### 4. Phone numbers need E.164
Store/send `+15551234567`, not `(555) 123-4567`. Non-E.164 phones break dedup matching and SMS.

### 5. Tags are additive strings; there is no "set exact tags"
`POST /contacts/{id}/tags` adds; `DELETE /contacts/{id}/tags` removes. To reach a target set, diff current vs desired and add/remove the delta.

---

## Level 2: Duplicate Detection & Safe Updates (Intermediate)

### Know how upsert resolves duplicates
Upsert obeys the location's **Allow Duplicate Contact** setting and its match priority (email/phone).

| Scenario | Result |
|---|---|
| One existing contact matches email | That contact is updated |
| Email matches contact A, phone matches contact B | Updates the one matching the **first** field in the configured priority; ignores the other to avoid a merge |
| No match, duplicates disallowed | Creates one |
| No match, duplicates allowed | Creates (may add a dup) |

**Implication:** don't assume upsert merges A and B. It picks one. If you need true reconciliation, search first and decide explicitly.

### Search before you assume
```ts
// Find by exact email/phone before creating, when you need control over the match
const { contacts } = await ghlFetch("/contacts/search", {
  method: "POST",
  body: JSON.stringify({ locationId, filters: [{ field: "email", operator: "eq", value: email }] }),
}).then(r => r.json());
```

### Update is a partial merge, but arrays replace
`PUT /contacts/{id}` merges scalar fields. But sending `tags` or `customFields` may replace rather than append depending on the field — prefer the dedicated tag endpoints for tags, and send the full customFields array you intend.

### Match the custom-field value TYPE
```ts
// text/number/date → scalar; multi-select/checkbox → array
customFields: [
  { id: TEXT_ID, field_value: "hello" },
  { id: MULTISELECT_ID, field_value: ["A", "B"] },   // array, not "A,B"
  { id: DATE_ID, field_value: "2026-07-22" },        // match the field's expected format
]
```
A scalar into a multi-select field (or a bad date format) → **422 Unprocessable Entity**.

---

## Level 3: Bulk Sync at Scale (Advanced)

### Idempotent, resumable, rate-aware sync loop
```ts
import pLimit from "p-limit";
const limit = pLimit(8);                 // stay under 100/10s burst
const byKey = await loadFieldMap(locationId);   // cache field IDs once

await Promise.all(records.map(rec => limit(async () => {
  await ghlFetch("/contacts/upsert", {
    method: "POST",
    body: JSON.stringify({
      locationId,
      email: rec.email, phone: toE164(rec.phone),
      firstName: rec.first, lastName: rec.last,
      customFields: [{ id: byKey["contact.external_id"], field_value: rec.externalId }],
    }),
  });
})));
```

### Store the GHL contactId keyed by your external ID
Persist the mapping `externalId → ghlContactId` on first upsert so later syncs update directly and you never re-search.

---

## Performance: Make It Fast

- **Batch reads with `limit=100`** and cursor pagination (see `mx-ghl-core`); never fetch one contact at a time in a loop.
- **Cache the custom-field ID map** per location for the process lifetime — it changes rarely and costs a request each time.
- **Deduplicate your input before sending.** Two upserts for the same email in one run waste requests and can race.

```ts
❌ BAD — a lookup request per record before each upsert
for (const r of records) { await getFieldMap(locationId); await upsert(r); }

✅ GOOD — map once, reuse
const byKey = await getFieldMap(locationId);
await Promise.all(records.map(r => limit(() => upsert(r, byKey))));
```

## Observability: Know It's Working

- Track **create vs update outcomes** from upsert responses to catch a broken dedup config (sudden spike in creates = duplicates forming).
- Count **422s by field id** — a rising 422 rate on one custom field means its ID changed, was deleted, or the value type drifted.
- Log the `contactId` returned so downstream systems can be reconciled.
- Alert if your sync's contact count diverges from the source system by more than a small threshold.

---

## Enforcement: Anti-Rationalization Rules

### Rule 1: Sync = upsert, never create
**You will be tempted to:** use `POST /contacts/` in a sync job because it's the "create contact" endpoint.
**Why that fails:** every re-run creates duplicates; within weeks the location is full of triplicate contacts and dedup reports are wrong.
**The right way:** `POST /contacts/upsert` with email/phone so re-runs update in place.

### Rule 2: Custom fields go in the customFields array by ID
**You will be tempted to:** set a custom field as a top-level property using its display name.
**Why that fails:** GHL ignores unknown top-level keys or 422s; the field silently never updates.
**The right way:** `customFields: [{ id, field_value }]` with the field's per-location ID.

### Rule 3: Never hardcode custom-field IDs
**You will be tempted to:** paste the field ID from one sub-account into config and reuse it everywhere.
**Why that fails:** field IDs are per-location; the same code writes to the wrong field (or 422s) in another sub-account.
**The right way:** fetch `/locations/{id}/customFields`, build a key→id map at runtime, cache per location.

### Rule 4: Don't assume upsert merges two contacts
**You will be tempted to:** send both email and phone expecting upsert to merge the email-match and phone-match into one.
**Why that fails:** it updates only the first-priority match and ignores the other — you still have two contacts, now inconsistent.
**The right way:** if you need reconciliation, search both fields first and merge/decide explicitly.

### Rule 5: E.164 phones or dedup breaks
**You will be tempted to:** pass phone numbers in whatever format the source system stored them.
**Why that fails:** non-E.164 phones fail duplicate matching and SMS delivery, creating dups and undelivered messages.
**The right way:** normalize to `+<country><number>` before every create/upsert.
