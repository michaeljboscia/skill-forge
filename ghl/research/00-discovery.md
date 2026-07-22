# GHL Research — Phase 1 Discovery

## Platform facts (authoritative, verified 2026-07)

- **API v2 base URL:** `https://services.leadconnectorhq.com`
- **API v1:** END OF SUPPORT 31-Dec-2025. New v1 key generation disabled in agency + sub-account settings. Existing v1 integrations still run but get no patches/support. Base was `https://rest.gohighlevel.com/v1`. **Do not write new v1 code.**
- **Version header is MANDATORY on every v2 call:** `Version: 2021-07-28`. Missing it → request rejected. This is the single most common footgun.
- **Docs:** https://marketplace.gohighlevel.com/docs/ ; OpenAPI repo https://github.com/GoHighLevel/highlevel-api-docs
- **Official SDKs:** Node `@gohighlevel/api-client` (v3.x), Python, PHP. Repo: https://github.com/GoHighLevel/highlevel-api-sdk . SDKs handle auth, auto token refresh, PIT support, per-location token storage, webhook helpers, typed services.

## Auth model (two paths)

### 1. OAuth 2.0 (marketplace apps, multi-account)
- Authorization page: `https://marketplace.leadconnectorhq.com/oauth/chooselocation` (agency admin selects location(s) to install into)
- Token endpoint: `https://services.leadconnectorhq.com/oauth/token`
- Token exchange params: `client_id`, `client_secret`, `grant_type=authorization_code`, `code`, `redirect_uri`, `user_type` (`Company` for agency-level, `Location` for sub-account-level)
- **Content-Type for token endpoint: `application/x-www-form-urlencoded`** (NOT json — verify in phase 2; common footgun)
- Response: `access_token` (JWT), `refresh_token`, `expires_in` (~86399s ≈ 24h), `userType`, `companyId`, `locationId`, `scope`
- **Access token expires every ~24h → 401 if stale. Must refresh.**
- **Refresh token: valid 1 year OR until used → it ROTATES. Each refresh returns a NEW refresh_token; you must persist the new one or you lock yourself out.**
- Company→Location token: POST `https://services.leadconnectorhq.com/oauth/locationToken` with agency Bearer token + `Version: 2021-07-28`, body `companyId`, `locationId`. Agency token cannot call location-scoped endpoints directly for most resources.

### 2. Private Integration Token (PIT) — single account, server-to-server
- Create: sub-account/agency Settings → Private Integrations → name + scopes → get `pit-...` token
- No OAuth consent flow, no refresh cycle. Use `Authorization: Bearer pit-xxxx`.
- Best for single sub-account internal integrations. OAuth is for marketplace/multi-account.
- Works only with API v2. Replaces legacy v1 API keys (which are EOL).

## Rate limits (v2, per Marketplace app, per resource = per Location or Company)
- **Burst: 100 requests / 10 seconds**
- **Daily: 200,000 requests / day**
- Rate-limit headers returned on responses (X-RateLimit-*). 429 on exceed.
- Best practice: exponential backoff w/ jitter (500→1000→2000ms ±20%), stagger multi-client syncs, cache, prefer webhooks over polling.

## Webhooks — signature verification
- Two headers during transition:
  - `x-wh-signature` — RSA-SHA256, **deprecated 1 July 2026**. Verified with GHL RSA public key.
  - `x-ghl-signature` — Ed25519 (new standard). Verified with GHL Ed25519 public key (PEM/SPKI, raw 32 bytes, hex, or base64).
- **MUST verify against RAW request body bytes.** Any JSON re-serialization/parse-then-stringify breaks the signature.
- Webhooks can be spoofed → always verify.

## Scopes
- Granular per resource: e.g. `contacts.readonly`, `contacts.write`, `conversations.write`, `opportunities.write`, `calendars.write`, `payments/orders.readonly`, etc.
- App requests scopes at creation; token only grants what was consented. Missing scope → 401/403.

## Sources
- https://www.gohighlevel.com/post/deprecating-the-highlevel-api-v1-and-migrating-to-v2
- https://marketplace.gohighlevel.com/docs/Authorization/OAuth2.0/
- https://marketplace.gohighlevel.com/docs/Authorization/PrivateIntegrationsToken/
- https://github.com/GoHighLevel/highlevel-api-sdk
- https://help.gohighlevel.com/support/solutions/articles/48001060529-highlevel-api-documentation
