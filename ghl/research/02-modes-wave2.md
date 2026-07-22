# GHL Research — Phase 2 Wave 2 + Phase 3 deep notes

## Rate-limit headers (EXACT names — use for observability)
- `X-RateLimit-Limit-Daily` — daily cap (200000)
- `X-RateLimit-Daily-Remaining` — remaining today
- `X-RateLimit-Interval-Milliseconds` — burst window (10000)
- `X-RateLimit-Max` — max per interval (100)
- `X-RateLimit-Remaining` — remaining in current 10s window
- Rule of thumb: if `X-RateLimit-Remaining` < 10% of Max, start spacing requests BEFORE hitting 0.

## Webhook retry model (CRITICAL quirk)
- **GHL Marketplace app webhooks currently retry ONLY on HTTP 429 from your endpoint.** They do NOT retry on 5xx/timeouts like Stripe does (feature request #257 open).
- Implication: if your receiver 500s or times out, the event is GONE. So: return 2xx immediately, enqueue, process async. Never do slow work inline in the webhook handler.
- Still implement idempotency (dedup on event id) because 429-driven retries + at-least-once semantics can duplicate.

## Custom Objects (v2)
- REST CRUD on objects + records IS available (create/update/get/delete records).
- **Associations API NOT yet available** (UI only; "future updates"). Don't write code that queries associations via API.
- Custom objects now in all plans, higher limits. Relevant to `core` skill (mention) — not its own skill for v1 of package.

## Custom field value format
- On contact create/update/upsert: `customFields: [{ id: "<fieldId>", field_value: <value> }]`.
  - Text/number/date: scalar value.
  - Multi-select / checkbox: array value.
  - Date: ISO/epoch depending on field; mismatch → 422.
- Get field ids from `GET /locations/{locationId}/customFields` (or Custom Fields V2 API). Field ids are per-location → NEVER hardcode across locations.
- 422 on custom field write usually = wrong field id, wrong value type, or field not existing in that location.

## Pagination
- Cursor-based: `limit` (max 100), `startAfter` (timestamp), `startAfterId`. Loop until fewer than `limit` returned. No stable sort param. Undocumented `page` exists but unofficial — avoid.

## Payments (confirm)
- Read-heavy: orders/transactions/subscriptions GET + filter. Create/write for orders/subscriptions mostly NOT exposed. Invoices API has more write capability.

## Deep synthesis — the 8 highest-leverage anti-rationalizations
1. Writing v1 code (rest.gohighlevel.com) because tutorials show it → v1 is EOL, keys can't be created.
2. Omitting `Version: 2021-07-28` header → silent-ish failures.
3. `application/json` on the token endpoint → invalid_request.
4. Not persisting the ROTATED refresh_token → permanent lockout after first refresh.
5. Hardcoding pipeline/stage/customField/calendar IDs → they are per-location, break across sub-accounts.
6. Verifying webhook signature against parsed/re-stringified body → signature mismatch.
7. Doing real work synchronously in webhook handler → lost events (no 5xx retry).
8. Firing requests in tight loops → 429 (100/10s burst); no backoff.

## Sources
- https://marketplace.gohighlevel.com/docs/oauth/Faqs/
- https://github.com/GoHighLevel/highlevel-api-docs/issues/257
- https://help.gohighlevel.com/support/solutions/articles/155000004023-creating-and-updating-custom-object-records
- https://marketplace.gohighlevel.com/docs/ghl/custom-fields/custom-fields-v-2-api/
- https://marketplace.gohighlevel.com/docs/ghl/contacts/update-contact/
