# GHL Research — Phase 2 Wave 1 (per-mode)

## CORE / AUTH footguns (confirmed)
- **Token endpoint Content-Type MUST be `application/x-www-form-urlencoded`.** Sending `application/json` → invalid_request/400. This is a top footgun (docs + community confirm).
- Refresh token ROTATES on every use → persist the returned new refresh_token or you get locked out (n8n community reports "not refreshing" = they didn't store rotated token).
- `Version: 2021-07-28` header required on all v2 resource calls.
- 401 = expired access token OR missing scope. 403 = scope/permission. Distinguish before retrying.
- Redirect URI must match app config exactly. Scopes selected in code must be subset of app-registered scopes.

## CONTACTS
- Upsert: `POST /contacts/upsert`. Respects location "Allow Duplicate Contact" setting; dedup priority sequence (email/phone). If two contacts match different fields, updates the one matching FIRST field in sequence, ignores second.
- Create: `POST /contacts/` ; Update: `PUT /contacts/{contactId}`.
- List: `GET /contacts/` with `locationId`, `limit` (max 100), `startAfter` (timestamp), `startAfterId`. NO sort. Undocumented `page` param works but is unofficial — don't rely on it.
- customFields passed as array: `customFields: [{ id: "<fieldId>", field_value: "..." }]` (also accepts `key`). Wrong ID format → 422.
- Contacts are LOCATION-scoped. locationId required.

## OPPORTUNITIES
- Create: `POST /opportunities/`. Required: `locationId`, `pipelineId`, `pipelineStageId`, `name`, `status` (open/won/lost/abandoned). Also `contactId`, `monetaryValue` (number), `assignedTo`, `source`.
- Pipelines/stages: GET only via `/opportunities/pipelines?locationId=`. **No API to CREATE pipelines/stages** (feature request open) → must fetch existing IDs at runtime, never hardcode.
- monetaryValue is a number (currency-unaware) — treat consistently.

## CONVERSATIONS / MESSAGING
- Send: `POST /conversations/messages`. Body: `type` (SMS | Email | WhatsApp | IG | FB | Custom | Live_Chat), `contactId`, `message` (SMS), Email needs `subject`,`html`/`message`,`emailFrom`,`attachments[]`.
- Requires an active **conversation provider** configured for the channel or send fails.
- 50+ events. Inbound arrives via webhook `InboundMessage`; outbound `OutboundMessage`.
- Conversation Providers marketplace module = build custom SMS/Email/Call providers.

## CALENDARS
- Get free slots: `GET /calendars/{calendarId}/free-slots` with `startDate`,`endDate` (epoch ms), optional `timezone`, `userId`. Returns slots grouped by date.
- **Timezone is the #1 bug.** GHL detects visitor tz on booking widget; via API you must pass/interpret tz explicitly. Slots returned in requested tz.
- Create appointment: `POST /calendars/events/appointments` with `calendarId`,`locationId`,`contactId`,`startTime`,`endTime` (ISO 8601 w/ offset), `title`, `appointmentStatus`, `ignoreDateRange`, `toNotify`.
- Booking a taken slot → conflict; always fetch free-slots first.

## WEBHOOKS
- 50+ event types: ContactCreate/Update/Delete/DndUpdate, ConversationCreate/Update/Unread, InboundMessage, OutboundMessage, AppointmentCreate/Update/Delete, OpportunityCreate/Update/Delete/StatusUpdate/StageUpdate/MonetaryValueUpdate, TaskCreate/Complete, etc.
- Signature headers: `x-wh-signature` (RSA-SHA256, DEPRECATED 1 Jul 2026) and `x-ghl-signature` (Ed25519, current). Verify against RAW body bytes.
- Must return 2xx fast; GHL retries on failure → receiver MUST be idempotent (dedup on event id / webhookId).
- SDK: `ghl.webhooks.subscribe()` express middleware sets `req.isSignatureValid`. Env: `WEBHOOK_SIGNATURE_PUBLIC_KEY` (Ed25519), `WEBHOOK_PUBLIC_KEY` (RSA).

## MARKETPLACE APPS
- OAuth install via `marketplace.leadconnectorhq.com/oauth/chooselocation`.
- Custom Pages / Custom Menu Links embed your app in the CRM iframe.
- SSO / user context: `window.exposeSessionDetails(APP_ID)` (or postMessage `REQUEST_USER_DATA`) → encrypted payload → POST to your backend → decrypt with **SSO Key** (AES-256-CBC, OpenSSL salted format). Never trust client-provided user identity without decrypting server-side.
- App templates: github.com/GoHighLevel/ghl-marketplace-app-template
- Two install targets: Sub-Account (Location) apps vs Agency (Company) apps → affects token userType.

## PAYMENTS
- Read APIs available: `GET /payments/orders`, `/payments/orders/{id}`, `/payments/transactions`, `/payments/subscriptions`, `/payments/subscriptions/{id}`. Filter transactions by status, paymentMode, date range, contactId, subscriptionId, entityId.
- **Create/write for orders & subscriptions largely NOT available** (read-heavy) — feature requests open. Don't assume you can POST an order.
- Scopes: `payments/orders.readonly|write`, `payments/subscriptions.readonly`, `payments/transactions.readonly`.
- Custom payment provider integration = marketplace Payments module (build a provider).

## SDK (official @gohighlevel/api-client v3, Node/TS)
- PIT: `new HighLevel({ privateIntegrationToken, logLevel })`
- OAuth: `new HighLevel({ clientId, clientSecret, sessionStorage: new MongoDBSessionStorage({...}) })` — auto-refresh on 401, persistent storage across restarts; in-memory otherwise.
- Custom storage: extend `SessionStorage` (init/disconnect/setSession/getSession/deleteSession).
- Services: `ghl.contacts`, `.conversations`, `.calendars`, `.opportunities`, `.invoices`, `.campaigns`, `.locations`, ...
- Errors: `GHLError` with `statusCode`, `message`.
- `preferredTokenType: 'location'` option to force location token.

## Sources
- https://marketplace.gohighlevel.com/docs/Authorization/OAuth2.0/
- https://marketplace.gohighlevel.com/docs/ghl/contacts/upsert-contact/
- https://marketplace.gohighlevel.com/docs/ghl/calendars/get-slots/
- https://marketplace.gohighlevel.com/docs/webhook/WebhookIntegrationGuide/
- https://marketplace.gohighlevel.com/docs/other/user-context-marketplace-apps/
- https://marketplace.gohighlevel.com/docs/ghl/payments/orders/
- https://github.com/GoHighLevel/highlevel-api-sdk
- https://github.com/GoHighLevel/ghl-marketplace-app-template
