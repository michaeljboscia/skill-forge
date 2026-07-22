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

## Phase 6 — Pressure test (next session, TODO)
Suggested test prompt: "Build a GHL marketplace app that installs into a location via OAuth, upserts a contact with a custom field, creates an opportunity in the correct pipeline stage, books an appointment, and receives InboundMessage webhooks — production-ready." Exercises core + marketplace + contacts + opportunities + calendars + webhooks simultaneously. Check: correct v2 base, Version header, form-encoded token exchange, runtime ID resolution, raw-body signature verify, 2xx-then-async.
