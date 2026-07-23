# mx-ghl-* Session Log

**Date:** 2026-07-22
**Package:** mx-ghl-* (GoHighLevel / HighLevel API v2)
**Skills:** 8 (core, contacts, opportunities, conversations, calendars, webhooks, marketplace, payments)

## Phase 0 — Scope
8 modes chosen, each with a distinct AI failure mode (see drafts/README.md table). Closest precedent: mx-hubspot-* (CRM/marketing platform).

## Phases 1–3 — Research (full saturation)
Web-search saturation + doc/SDK fetches, persisted to `research/`:
- `research/00-discovery.md` — platform facts, auth model, rate limits, webhook signatures, SDKs
- `research/01-modes-wave1.md` — per-mode endpoints & footguns
- `research/02-modes-wave2.md` — rate-limit headers, webhook retry quirk, custom objects, deep synthesis

Key verified footguns driving the anti-rationalization rules:
1. v1 API EOL 31-Dec-2025 — no new keys
2. `Version: 2021-07-28` header mandatory
3. Token endpoint requires `application/x-www-form-urlencoded` (not JSON)
4. Refresh tokens rotate — must persist the new one
5. Per-location IDs (pipelines, stages, custom fields, calendars) — never hardcode
6. Webhook signatures verify against RAW body; Ed25519 (`x-ghl-signature`) replacing RSA (`x-wh-signature`) on 1-Jul-2026
7. GHL webhooks retry ONLY on 429 — 5xx/timeout = lost event
8. 100 req/10s burst, 200k/day — needs backoff + bounded concurrency

## Phase 4 — Writing
8 SKILL.md files in `drafts/`, all < 500 lines, 5 anti-rationalization rules each (40 total), L1–L3 + Performance + Observability.

## Phase 5 — Promotion
Copied to `~/.claude/skills/mx-ghl-*/` with `reference/` dirs. Committed to repo branch `claude/gohighlevel-skills-61upku`. ROADMAP updated (#16).

## Phase 6 — Pressure test (DONE 2026-07-22)

Ran 4 fresh subagents: 3 realistic multi-mode build tasks + 1 adversarial reviewer.

### Build tests — 3/3 PASS
- **App + cross-location webhook sync** (core+marketplace+webhooks+contacts+opportunities): typechecked; agency OAuth + locationToken, form-encoded token exchange, atomic rotated-refresh persistence, raw-body Ed25519 verify (+RSA fallback), ack-then-async, runtime pipeline/stage resolution, upsert + dup-deal guard.
- **Booking + SMS** (core+calendars+conversations): epoch-ms slot range + explicit IANA tz, booked an offered slot, 409/422 fall-through, DND-aware idempotent SMS.
- **Revenue sync** (core+payments): refused to invent create-order API, pivoted to invoices+record-payment, decimal-safe money, live/test separation, idempotent.

Skills held under a 5-skill simultaneous load — the framework's binary pass criterion met.

### Adversarial review — 15 findings, all folded back in
The sharp meta-finding: several Level-3 "resilient" code snippets reintroduced the exact bug their own rule forbids. Fixes applied:
- **[CRIT] core**: refresh had no single-flight guard → concurrent 401s = rotated-token lockout. Added single-flight refresh + Rule 6.
- **[CRIT] core**: blanket "cursor pagination, no page param" was wrong → made pagination per-endpoint (contacts=startAfter/startAfterId, opps-search=page, payments=offset/limit).
- **[HIGH] payments**: examples used `locationId` → payments use `altId`+`altType=location`; pagination offset/limit; added concrete invoice→record-payment right-way.
- **[HIGH] conversations**: `sendOnce` double-texted on timeout → claim-before-send, release only on definite 4xx.
- **[HIGH] webhooks**: Ed25519 snippet had `crypto` unimported (ReferenceError) → fixed import.
- **[HIGH] webhooks**: idempotency marked-seen before enqueue + non-atomic → enqueue-first, atomic insertIfAbsent.
- **[MED] webhooks**: softened "return 429 to shed load" → persist-first, don't depend on GHL redelivery (cited issue #257).
- **[MED] opportunities**: stage updates now send `pipelineId` with `pipelineStageId`; softened `status`-required (defaults open).
- **[MED] conversations**: inbound example fixed — dropped bogus `direction`, added `conversationProviderId` (custom SMS), conversationId|contactId.
- **[MED] contacts**: added read/write asymmetry (GET `value` vs write `field_value`); recommended `/contacts/search/duplicate`; added DND field shape.
- **[LOW-MED] marketplace**: SSO postMessage listener now checks `event.origin` + removes listener.
- **[LOW-MED] core**: rate limit reworded "per app per resource (Location OR Company)"; agency budget shared.
- **[LOW] calendars**: showed date-keyed free-slots parsing + assignedUserId for team calendars.

Net: core grew to 6 rules; all 8 skills still <500 lines. Verified factual claims (custom-field value/field_value, inbound provider, payments altId) via web search before editing.
