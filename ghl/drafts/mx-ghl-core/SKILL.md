---
name: mx-ghl-core
description: GoHighLevel / HighLevel API v2 fundamentals for AI agents — auth (OAuth 2.0 marketplace apps vs Private Integration Tokens), the mandatory Version header, agency (Company) vs sub-account (Location) tokens, rotating refresh tokens, rate limits (100/10s, 200k/day), pagination, and error handling against services.leadconnectorhq.com. Load for ANY GHL/LeadConnector/HighLevel API work before writing a request.
---

# GoHighLevel Core — API v2 Fundamentals for AI Coding Agents

**Loads whenever you touch the GoHighLevel / HighLevel / LeadConnector API. Every other mx-ghl-* skill assumes the rules here.**

## When to also load
- `mx-ghl-marketplace` — building an installable app, OAuth install flow, SSO/custom pages
- `mx-ghl-webhooks` — receiving events, signature verification
- `mx-ghl-contacts` / `-opportunities` / `-conversations` / `-calendars` / `-payments` — per-resource work

---

## Level 1: Patterns That Always Work (Beginner)

### 1. Use API v2 only. v1 is dead.
API v1 (`rest.gohighlevel.com/v1`, static API keys) reached **end-of-support 31-Dec-2025**. New v1 keys cannot be generated. All new code targets v2.

```
❌ BAD  — base URL from old tutorials
https://rest.gohighlevel.com/v1/contacts/    // v1, EOL, no new keys, no support

✅ GOOD — v2 base, every time
https://services.leadconnectorhq.com/contacts/
```

### 2. The `Version` header is mandatory on every call.
There is no default. Omit it and requests are rejected.

```ts
const headers = {
  Authorization: `Bearer ${accessToken}`,
  Version: "2021-07-28",           // ← REQUIRED on every v2 request
  "Content-Type": "application/json",
  Accept: "application/json",
};
```

### 3. Pick the right auth for the job.

| Situation | Use | Why |
|---|---|---|
| One sub-account, internal server-to-server | **Private Integration Token (PIT)** | No consent flow, no refresh cycle. `Authorization: Bearer pit-...` |
| Multi-account / installable Marketplace app | **OAuth 2.0** | Per-location consent, agency distribution |
| Anything | ~~v1 API key~~ | Dead. Do not. |

```ts
// PIT — simplest correct path for a single location
const res = await fetch("https://services.leadconnectorhq.com/contacts/?locationId=" + locationId, {
  headers: { Authorization: `Bearer ${PIT_TOKEN}`, Version: "2021-07-28", Accept: "application/json" },
});
```

### 4. `locationId` is required almost everywhere.
Nearly every resource (contacts, opportunities, calendars, conversations) is scoped to a sub-account (Location). Pass `locationId` as a query param or body field per the endpoint. Missing it → 401/422.

### 5. Prefer the official SDK over hand-rolled fetch.
`@gohighlevel/api-client` (Node/TS), plus Python and PHP SDKs. It sets the Version header, refreshes tokens, and exposes typed services.

```ts
import HighLevel, { LogLevel } from "@gohighlevel/api-client";
const ghl = new HighLevel({ privateIntegrationToken: process.env.GHL_PIT, logLevel: LogLevel.INFO });
const contacts = await ghl.contacts.getContacts({ locationId, limit: 20 });
```

---

## Level 2: OAuth Tokens & Scoping (Intermediate)

### The token endpoint takes form-encoding, not JSON
This is the single most common OAuth failure.

```ts
❌ BAD
await fetch("https://services.leadconnectorhq.com/oauth/token", {
  method: "POST",
  headers: { "Content-Type": "application/json" },      // ← rejected: invalid_request
  body: JSON.stringify({ ... }),
});

✅ GOOD
await fetch("https://services.leadconnectorhq.com/oauth/token", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },   // ← required
  body: new URLSearchParams({
    client_id: CLIENT_ID,
    client_secret: CLIENT_SECRET,
    grant_type: "authorization_code",
    code,
    redirect_uri: REDIRECT_URI,
    user_type: "Location",            // "Company" for agency-level, "Location" for sub-account
  }),
});
```

### Agency (Company) token → Location token
An agency/Company token cannot call most Location-scoped resources directly. Exchange it:

```ts
// POST with the agency access token to mint a location-scoped token
await fetch("https://services.leadconnectorhq.com/oauth/locationToken", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${agencyAccessToken}`,
    Version: "2021-07-28",
    "Content-Type": "application/x-www-form-urlencoded",
  },
  body: new URLSearchParams({ companyId, locationId }),
});
```

### Refresh tokens ROTATE — persist the new one
Access tokens live ~24h (`expires_in` ≈ 86399). Refreshing returns a **new** refresh_token; the old one is now invalid.

| Choice | Result |
|---|---|
| Store the new `refresh_token` from every refresh | Keeps working |
| Reuse the original refresh_token after refreshing | Locked out — must reinstall |

```ts
const r = await fetch("https://services.leadconnectorhq.com/oauth/token", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: new URLSearchParams({
    client_id, client_secret, grant_type: "refresh_token",
    refresh_token: stored.refreshToken, user_type: "Location",
  }),
}).then(r => r.json());
await saveTokens({ accessToken: r.access_token, refreshToken: r.refresh_token }); // ← save BOTH
```

### Scopes must be a subset of what the app registered
Request a scope in the URL that the app didn't declare → error. Call an endpoint whose scope wasn't consented → 401/403 (not 404). Add the scope in the Marketplace app config AND re-consent.

---

## Level 3: Resilient Clients (Advanced)

### One adapter, every call goes through it
Centralize base URL, Version header, auth, retry, and rate-limit awareness. Never scatter raw `fetch` to GHL across the codebase.

```ts
async function ghlFetch(path: string, init: RequestInit = {}, tries = 0): Promise<Response> {
  const res = await fetch(`https://services.leadconnectorhq.com${path}`, {
    ...init,
    headers: { Version: "2021-07-28", Accept: "application/json",
               Authorization: `Bearer ${await getFreshToken()}`, ...init.headers },
  });
  if (res.status === 429 && tries < 3) {
    const wait = (2 ** tries) * 500 * (0.8 + Math.random() * 0.4); // 500→1000→2000ms ±20%
    await new Promise(r => setTimeout(r, wait));
    return ghlFetch(path, init, tries + 1);
  }
  if (res.status === 401 && tries === 0) { await refreshAndStore(); return ghlFetch(path, init, tries + 1); }
  return res;
}
```

### Cursor pagination — loop until short page
List endpoints use `limit` (max 100) + `startAfter` (timestamp) + `startAfterId`. There is no reliable `page` or sort param.

```ts
let startAfter, startAfterId, all = [];
do {
  const p = new URLSearchParams({ locationId, limit: "100" });
  if (startAfter) { p.set("startAfter", String(startAfter)); p.set("startAfterId", startAfterId); }
  const { contacts, meta } = await ghlFetch(`/contacts/?${p}`).then(r => r.json());
  all.push(...contacts);
  ({ startAfter, startAfterId } = meta ?? {});
} while (startAfter);   // stop when the page is short / meta empty
```

---

## Performance: Make It Fast

- **Respect the burst budget: 100 requests / 10s per app per Location.** A tight `for` loop over 500 contacts will 429 by request 101. Batch, or throttle to ≤10 req/s with a concurrency limiter (e.g. `p-limit`).
- **Prefer webhooks over polling.** Subscribing to `ContactUpdate`/`OpportunityStatusUpdate` costs zero of your daily 200k budget vs. polling lists every minute.
- **Cache stable lookups** (pipeline IDs, custom-field IDs, calendar IDs, location settings) for the process lifetime — they change rarely and cost a request each.

```ts
❌ BAD  — 500 sequential calls, 429 by #101
for (const c of contacts) await ghlFetch(`/contacts/${c.id}`);

✅ GOOD — bounded concurrency under the burst ceiling
import pLimit from "p-limit";
const limit = pLimit(8);
await Promise.all(contacts.map(c => limit(() => ghlFetch(`/contacts/${c.id}`))));
```

## Observability: Know It's Working

Log the rate-limit headers on every response and alarm before you hit zero:

| Header | Meaning |
|---|---|
| `X-RateLimit-Max` | max per interval (100) |
| `X-RateLimit-Remaining` | remaining in the current 10s window |
| `X-RateLimit-Interval-Milliseconds` | burst window (10000) |
| `X-RateLimit-Limit-Daily` | daily cap (200000) |
| `X-RateLimit-Daily-Remaining` | remaining today |

- Alert when `X-RateLimit-Daily-Remaining` drops below 10% — you're about to lose all API access for the day.
- Log `traceId`/response body on every non-2xx; GHL error bodies carry the real reason (bad scope vs bad field).
- Track token refresh events and failures — a spike in 401s means refresh-token rotation broke.

---

## Enforcement: Anti-Rationalization Rules

### Rule 1: No v1
**You will be tempted to:** copy a `rest.gohighlevel.com/v1` snippet because it's shorter and appears in most blog posts.
**Why that fails:** v1 is end-of-support (31-Dec-2025), new keys are disabled, and it gets no fixes. Code you write today on v1 is dead on arrival.
**The right way:** `https://services.leadconnectorhq.com` + `Version: 2021-07-28` + OAuth or PIT.

### Rule 2: The Version header is not optional
**You will be tempted to:** drop the `Version` header because the request "looks complete" with just Authorization.
**Why that fails:** v2 rejects requests without it. You'll waste time debugging auth when the header is the cause.
**The right way:** put `Version: "2021-07-28"` in your shared client so it's impossible to omit.

### Rule 3: Token endpoint is form-encoded
**You will be tempted to:** send `Content-Type: application/json` to `/oauth/token` because every other GHL endpoint is JSON.
**Why that fails:** the token endpoint only accepts `application/x-www-form-urlencoded`; JSON returns `invalid_request` and you'll blame your client_secret.
**The right way:** `new URLSearchParams({...})` with `Content-Type: application/x-www-form-urlencoded`. JSON everywhere else.

### Rule 4: Persist the rotated refresh token
**You will be tempted to:** save only the access_token on refresh and keep reusing the original refresh_token.
**Why that fails:** refresh tokens are single-use and rotate. The original is invalid after the first refresh → permanent lockout requiring reinstall.
**The right way:** on every token/refresh response, persist BOTH access_token and the new refresh_token atomically.

### Rule 5: Agency token ≠ Location access
**You will be tempted to:** call `/contacts` with the Company (agency) token because it authenticated fine.
**Why that fails:** most resources are Location-scoped; the agency token 401s or returns nothing.
**The right way:** exchange via `/oauth/locationToken` (companyId + locationId) and use the location token for resource calls.
