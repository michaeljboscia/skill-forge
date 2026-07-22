# mx-ghl-* — GoHighLevel / HighLevel API v2 Skill Package

**8 skills** for AI coding agents building on the GoHighLevel (HighLevel / LeadConnector) API v2, produced via the [Skills Factory Framework](../../SKILLS-FACTORY-FRAMEWORK.md).

| Skill | Mode | Distinct AI failure it prevents |
|-------|------|----------------------------------|
| `mx-ghl-core` | API fundamentals & auth | v1 code, missing `Version` header, JSON to the token endpoint, un-rotated refresh tokens, agency-vs-location token confusion, 429s |
| `mx-ghl-contacts` | Contacts & custom fields | duplicate contacts, wrong customFields format, hardcoded per-location field IDs |
| `mx-ghl-opportunities` | Pipelines & deals | hardcoded pipeline/stage IDs, invalid status enum, status-vs-stage confusion |
| `mx-ghl-conversations` | Messaging | sending without a provider, email-as-SMS, duplicate sends, ignoring DND |
| `mx-ghl-calendars` | Scheduling | timezone bugs, non-epoch slot ranges, double-booking, cancel+create reschedules |
| `mx-ghl-webhooks` | Event receivers | verifying against a re-serialized body, lost events (GHL only retries 429), non-idempotent handlers, RSA-only after Ed25519 cutover |
| `mx-ghl-marketplace` | Installable apps | one global token, trusting un-decrypted SSO context, wrong app target, uninstall leaks |
| `mx-ghl-payments` | Orders/txns/subs | inventing create endpoints, float money math, mixing live/test, entitlement on presence not status |

## Structure

Each skill follows the framework template:
- **Level 1–3** patterns (beginner → advanced) with BAD/GOOD code pairs
- **Performance** and **Observability** sections specific to the mode
- **Enforcement: Anti-Rationalization Rules** (5 per skill, 40 total) — each naming the exact rationalization, why it fails in production, and the right way

## Key platform facts (as of 2026-07)

- Base URL: `https://services.leadconnectorhq.com` — **API v1 is end-of-support (31-Dec-2025)**
- `Version: 2021-07-28` header is mandatory on every call
- Auth: OAuth 2.0 (marketplace/multi-account) or Private Integration Token (single account)
- Rate limits: 100 req / 10s and 200,000 req / day, per app per resource (Location or Company)
- Webhooks: verify `x-ghl-signature` (Ed25519); legacy `x-wh-signature` (RSA) deprecated 1-Jul-2026

## Provenance

Research waves backing these rules live in [`../research/`](../research/). Every non-obvious rule traces to the official docs, SDK, or documented community footguns cited there.
