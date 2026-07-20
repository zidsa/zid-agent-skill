# OAuth 2.0 Flow — Reference

Source of truth: `https://docs.zid.sa/authorization.md`. Re-fetch that page if anything here looks stale before implementing — this file is a working summary, not a replacement for the live doc.

## Why Authorization Code grant, and why server-side only

Zid uses the **Authorization Code grant** (RFC 6749 §4.1), the flow designed for confidential clients that can keep a secret. The Client Secret is exchanged for tokens in a server-to-server call, so it must never be embedded in a browser bundle, mobile app binary, or any client-side code. If a task description implies a purely client-side app, flag this — the app needs a backend component for the token exchange even if the rest of the UI is a SPA.

## The three tokens involved (this is the #1 source of bugs)

Zid's token model is genuinely confusing on first read — get this right before writing any request code:

| Name | Where it comes from | What it's for | Header |
|---|---|---|---|
| **Authorization token** | `authorization` field in the token-exchange response | Grants access to the Zid API generally | `Authorization: Bearer <value>` |
| **X-Manager-Token** | `access_token` field in the *same* token-exchange response | Scopes the request to one specific store | `X-Manager-Token: <value>` |
| **Access-Token** | Same value as X-Manager-Token | Used interchangeably with X-Manager-Token on Product-component endpoints, for historical/technical reasons — not a third distinct token | `Access-Token` (on the endpoints that expect it) |

**Every authenticated Merchant API request needs both `Authorization` and `X-Manager-Token` headers set**, not just one. A request with only `Authorization` will typically fail with 401/403 even though the token itself is valid — because the API still doesn't know which store to scope the call to.

```bash
curl -X GET "https://api.zid.sa/v1/managers/account/profile" \
  -H "Accept: application/json" \
  -H "Accept-Language: en" \
  -H "Authorization: Bearer <authorization-token>" \
  -H "X-Manager-Token: <access_token-value>"
```

When storing tokens per merchant, store both values under clearly distinct field names (e.g. `authorization_token` and `manager_token`) — don't collapse them into one field, and don't assume the names in Zid's JSON response ("authorization", "access_token") map cleanly onto REST convention; write a comment at the storage boundary explaining the mapping so the next person (human or AI) doesn't "simplify" it incorrectly.

## Step 1 — Generate Client ID and Client Secret

Done once per app in the Partner Dashboard (`https://partner.zid.sa`): create the app, choose its type, set the app URL and allowed redirect URL(s), then read the API key / API secret from the app's page. If the user hasn't done this yet, direct them there rather than inventing placeholder credentials that look real.

## Step 2 — Redirect the merchant to authorize

```
GET https://oauth.zid.sa/oauth/authorize
    ?client_id=<your_client_id>
    &redirect_uri=<your_registered_callback_url>
    &response_type=code
```

- `redirect_uri` must exactly match a URL registered for the app in the Partner Dashboard, or the authorize step will fail.
- The merchant authenticates (if needed), sees the scopes the app is requesting (configured per-app in the Partner Dashboard), and approves or declines.
- On approval, Zid redirects back to `redirect_uri` with a one-time `code` query parameter.

## Step 3 — Exchange the code for tokens

```bash
curl -X POST https://oauth.zid.sa/oauth/token \
  -d "grant_type=authorization_code" \
  -d "client_id=<your_client_id>" \
  -d "client_secret=<your_client_secret>" \
  -d "redirect_uri=<same_redirect_uri_used_above>" \
  -d "code=<code_from_callback>"
```

Send these as a form-encoded request body, not query parameters. The response payload contains:

```json
{
  "access_token": "...",     // → store as X-Manager-Token / Access-Token
  "authorization": "...",    // → store as Authorization: Bearer <value>
  "refresh_token": "...",    // → used to get new tokens later
  "expires_in": "..."        // token lifetime — long-lived (~1 year) per current docs
}
```

Persist all of these immediately, associated with the merchant/store ID from the callback context, before doing anything else with the request (see `multi-tenant-architecture.md`).

## Step 4 — Refresh before expiry, not after

```bash
curl -X POST https://oauth.zid.sa/oauth/token \
  -d "grant_type=refresh_token" \
  -d "refresh_token=<stored_refresh_token>" \
  -d "client_id=<your_client_id>" \
  -d "client_secret=<your_client_secret>" \
  -d "redirect_uri=<your_redirect_uri>"
```

Per the docs, refresh tokens are themselves long-lived (~1 year) and expire, so the refresh call needs to run **before** that window closes — the docs' own guidance is to refresh around the 10-month mark rather than waiting for expiry. Build this as a scheduled/background job keyed off each stored token's issued-at date, not as a reactive "retry on 401" hack — reactive-only refresh means every merchant's first request after expiry fails visibly, and if the refresh token itself has also lapsed, that merchant is silently locked out until they reinstall.

Practical pattern:
- Store `issued_at` (or compute `expires_at` from `expires_in`) alongside the tokens.
- Run a daily job that finds tokens expiring within the next ~30–60 days and refreshes them proactively.
- Also handle the reactive case (a 401 on a live request) as a fallback — attempt one refresh-and-retry, and if that also fails, surface a clear "merchant needs to reauthorize" state rather than a generic error.

## Uninstall handling

When a merchant uninstalls the app, Zid sends a webhook and **all tokens for that merchant become invalid immediately**. Subscribe to that event and, on receipt:
- Mark the merchant's tokens as revoked in storage (don't just delete silently — you may want the install history).
- Stop any scheduled jobs (like the refresh job above) for that merchant.
- Do not attempt to call the API for that merchant again until they reinstall and go through the flow again.

## Common failure modes and what they actually mean

| Symptom | Likely cause |
|---|---|
| 401 on a request that includes `Authorization` but not `X-Manager-Token` | Missing the store-scoping header — see the token table above. |
| `redirect_uri` mismatch error on the authorize or token step | The `redirect_uri` sent doesn't exactly match (including trailing slash, http vs https) what's registered in the Partner Dashboard for the app. |
| Token exchange succeeds but subsequent API calls 403 | Merchant approved fewer scopes than the endpoint requires, or the app is requesting a scope it wasn't configured for in the Partner Dashboard. |
| Everything worked for months, then suddenly all requests for one merchant fail | Refresh token likely expired (~1 year) without being proactively refreshed, or the merchant uninstalled the app. |
| Works for the first merchant, breaks for the second install | Near-certain sign that tokens are being stored/read without a store ID key — see `multi-tenant-architecture.md`. |

If a genuinely new or unclear auth error shows up that isn't one of these, don't guess — fetch `https://docs.zid.sa/authorization.md` fresh (the "Handling Errors" section there covers more), and if still unresolved, help the user prepare the escalation report in `error-handling.md`.
