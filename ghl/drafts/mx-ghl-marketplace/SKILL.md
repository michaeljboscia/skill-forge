---
name: mx-ghl-marketplace
description: GoHighLevel / HighLevel Marketplace app development for AI agents — the OAuth install flow (chooselocation), agency (Company) vs sub-account (Location) app targets, per-location token storage, custom pages / custom menu links in the CRM iframe, and SSO user context (exposeSessionDetails / postMessage, AES-256-CBC decrypt with the SSO key). Load when building an installable GHL/LeadConnector app, embedded page, or handling app user context.
---

# GoHighLevel Marketplace — App Development for AI Coding Agents

**Loads when you build an installable HighLevel Marketplace app: OAuth install, embedded custom pages, SSO, or multi-location token management.**

## When to also load
- `mx-ghl-core` — OAuth token exchange, refresh rotation, Version header (always)
- `mx-ghl-webhooks` — subscribing the app to events and verifying them
- `mx-ghl-payments` — if building a payment provider

---

## Level 1: Patterns That Always Work (Beginner)

### 1. The install flow starts at `chooselocation`
Send the agency admin to the authorization URL; they pick the location(s) to install into.

```
https://marketplace.leadconnectorhq.com/oauth/chooselocation
  ?response_type=code
  &client_id=YOUR_CLIENT_ID
  &redirect_uri=YOUR_REDIRECT_URI
  &scope=contacts.readonly contacts.write conversations.write ...
```
Your `redirect_uri` receives `?code=...`; exchange it at `/oauth/token` (form-encoded — see `mx-ghl-core`).

### 2. Decide the app's target: Sub-Account vs Agency
| Target | `user_type` on token | Use when |
|---|---|---|
| Sub-Account (Location) | `Location` | App operates inside one location's data |
| Agency (Company) | `Company` | App manages the agency / many locations; mint location tokens via `/oauth/locationToken` |

Choosing wrong means your tokens can't reach the resources you need.

### 3. Store tokens per location, not one global token
Each installed location has its own access/refresh token pair. Key your storage by `locationId` (or `companyId` for agency).

```ts
await tokens.set(locationId, { accessToken, refreshToken, expiresAt });
```

### 4. Request only the scopes you use — and declare them in the app config
The scopes in your install URL must be a subset of what the app registered. Missing/extra scope → error or 401 on calls.

### 5. Never trust identity the browser sends you — decrypt SSO server-side
Embedded custom pages get an *encrypted* user context. Decrypt it on your backend with the SSO key; don't accept a plaintext `userId` from the iframe.

---

## Level 2: Custom Pages & SSO (Intermediate)

### Get the encrypted user context in the iframe
Custom Pages run inside the CRM iframe. Ask the parent for the session, then send the encrypted blob to your backend.

```ts
// In the embedded page — CHECK event.origin and clean up the listener
const GHL_ORIGINS = ["https://app.gohighlevel.com", "https://app.leadconnectorhq.com"]; // allowlist
const encrypted: string = await new Promise((resolve) => {
  const onMsg = ({ origin, data }: MessageEvent) => {
    if (!GHL_ORIGINS.includes(origin)) return;                 // ignore spoofed frames
    if (data?.message === "REQUEST_USER_DATA_RESPONSE") {
      window.removeEventListener("message", onMsg);            // don't leak the listener
      resolve(data.payload);
    }
  };
  window.addEventListener("message", onMsg);
  window.parent.postMessage({ message: "REQUEST_USER_DATA" }, "*"); // parent origin varies; server-side decrypt is the real gate
});
// Some app types use window.exposeSessionDetails(APP_ID) instead.
await fetch("/api/decrypt-sso", { method: "POST", body: JSON.stringify({ encrypted }) });
```
Even with an origin check, the payload is still untrusted until your backend decrypts it with the SSO key — the origin check just stops obviously-hostile frames from feeding your decrypt path and prevents the listener leaking.

### Decrypt on the backend with the SSO key (AES-256-CBC, OpenSSL "Salted__" format)
```ts
import crypto from "node:crypto";

function decryptSSO(encryptedBase64: string, ssoKey: string) {
  const data = Buffer.from(encryptedBase64, "base64");
  const salt = data.subarray(8, 16);                 // "Salted__" + 8-byte salt header
  const ciphertext = data.subarray(16);
  // OpenSSL EVP_BytesToKey (MD5) to derive key+iv from passphrase + salt
  let d = Buffer.alloc(0), prev = Buffer.alloc(0);
  while (d.length < 48) { prev = crypto.createHash("md5")
      .update(Buffer.concat([prev, Buffer.from(ssoKey), salt])).digest(); d = Buffer.concat([d, prev]); }
  const key = d.subarray(0, 32), iv = d.subarray(32, 48);
  const dc = crypto.createDecipheriv("aes-256-cbc", key, iv);
  const out = Buffer.concat([dc.update(ciphertext), dc.final()]);
  return JSON.parse(out.toString("utf8"));           // { userId, companyId, role, activeLocation, ... }
}
```
The SSO key comes from your app's **Advanced Settings**. Store it as a secret; it decrypts every user's context.

### Authorize after decrypting
The decrypted payload tells you who the user is and which location/agency is active. Apply your own authorization (is this user/location entitled?) before returning app data.

---

## Level 3: Multi-Location Operations (Advanced)

### Agency app → per-location tokens on demand
```ts
async function locationToken(companyId: string, locationId: string) {
  const cached = await tokens.get(locationId);
  if (cached && cached.expiresAt > Date.now() + 60_000) return cached.accessToken;
  const r = await fetch("https://services.leadconnectorhq.com/oauth/locationToken", {
    method: "POST",
    headers: { Authorization: `Bearer ${await agencyToken(companyId)}`,
               Version: "2021-07-28", "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({ companyId, locationId }),
  }).then(r => r.json());
  await tokens.set(locationId, { accessToken: r.access_token, expiresAt: Date.now() + r.expires_in * 1000 });
  return r.access_token;
}
```

### Handle uninstall
Subscribe to the app uninstall event/webhook and delete that location's stored tokens so you stop calling a revoked install.

---

## Performance: Make It Fast

- **Cache location tokens** until shortly before expiry; don't mint a new one per request.
- **Lazy-mint** location tokens only for locations you actually call, not all installs upfront.
- **Verify SSO once per session** and issue your own short-lived session cookie/JWT rather than decrypting on every page load.

## Observability: Know It's Working

- Track **installs vs uninstalls** and reconcile against your token store; orphaned tokens = wasted refreshes and 401 noise.
- Monitor **per-location 401/403 rates** to catch revoked installs or scope changes early.
- Log **SSO decrypt failures** — a spike means a rotated SSO key or a tampered payload.
- Alert on **refresh-token failures per location**; one broken location shouldn't silently stop syncing.

---

## Enforcement: Anti-Rationalization Rules

### Rule 1: One token per location, keyed by locationId
**You will be tempted to:** keep a single access token for the whole app because the first install worked.
**Why that fails:** each install has its own token pair; a global token can't touch other locations and 401s the moment a second location installs.
**The right way:** store `{access, refresh, expiresAt}` per `locationId`/`companyId` and look up by install.

### Rule 2: Decrypt SSO on the server, never trust the iframe
**You will be tempted to:** read a `userId`/`locationId` the embedded page passes in plaintext and use it for authorization.
**Why that fails:** anyone can forge iframe messages; you'd grant access to arbitrary accounts.
**The right way:** take the encrypted blob, decrypt with the SSO key on your backend, then authorize from the decrypted identity.

### Rule 3: Pick the right app target
**You will be tempted to:** build a Location app and later try to manage many locations with its token.
**Why that fails:** a Location token is scoped to one sub-account; multi-location management needs an Agency (Company) app and `/oauth/locationToken`.
**The right way:** decide Sub-Account vs Agency up front based on whether the app spans locations.

### Rule 4: Scopes must match the app registration
**You will be tempted to:** add a scope to the install URL to "just get access" without updating the app config.
**Why that fails:** requested scopes must be a subset of registered ones; the install errors or calls 401/403 for unconsented scopes.
**The right way:** declare every scope in the app config, request the same set, and re-consent when you add one.

### Rule 5: Clean up on uninstall
**You will be tempted to:** ignore uninstalls and keep refreshing stored tokens.
**Why that fails:** you'll hammer refresh with revoked tokens, generating 401s and false alerts, and may retain data you shouldn't.
**The right way:** handle the uninstall event, delete that location's tokens, and stop calling it.
