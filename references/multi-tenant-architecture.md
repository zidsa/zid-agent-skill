# Multi-Tenant Architecture — Reference

A Zid app is, structurally, a single codebase serving N independent merchants who never interact with each other. Nearly every production bug in these apps traces back to one of: tokens not keyed by store, a shared rate-limit budget treated as per-app instead of per-store, or webhook events processed without verifying which store they belong to. Design against all three from the start.

## Data model: one credential row per merchant, never global state

Minimum fields to persist per installed store, however the actual schema is expressed (SQL table, document, KV namespace):

```
store_credentials
------------------
store_id            (Zid store/merchant identifier — the tenant key, index this)
app_client_id        (which of your app's client IDs this install belongs to, if you run more than one app)
authorization_token   (encrypted at rest)
manager_token          (a.k.a. access_token / X-Manager-Token, encrypted at rest)
refresh_token         (encrypted at rest)
token_issued_at
token_expires_at        (derived from expires_in — drives the proactive refresh job)
scopes_granted
status                 (active | revoked | needs_reauth)
installed_at
uninstalled_at         (nullable — set by the uninstall webhook handler)
```

Why encrypted at rest, specifically: these tokens are long-lived (~1 year) and each one grants access to a real merchant's store data — orders, customers, payment-adjacent data. A leaked database backup with plaintext tokens is a full breach across every merchant, not just one. Use your platform's standard secret-encryption mechanism (KMS-backed column encryption, a secrets manager, etc.) rather than storing them as plain text columns, even in early-stage projects — retrofitting encryption later means a painful migration across every existing tenant.

## The one rule that prevents cross-tenant leakage

**Every function that makes a Zid API call must take a `store_id` (or equivalent tenant identifier) as a required argument, and must look up that store's tokens itself** — never accept tokens as a loose parameter that a caller could mix up, and never rely on "whichever tokens are currently in scope/session" in a shared worker process. Concretely:

- Good: `callZidApi(storeId, endpoint, options)` — internally fetches `storeId`'s tokens.
- Risky: `callZidApi(authToken, managerToken, endpoint, options)` — nothing stops a bug from passing merchant A's tokens while processing merchant B's webhook.

This is the deterministic-gate version of "be careful with tenant isolation": make the correct behavior the only behavior the function's signature allows, rather than a convention developers (or an AI assistant) have to remember every time.

If background jobs, queues, or webhook processors exist, the tenant ID should travel with the job payload from the moment it's created, and every downstream step should re-derive tokens from that ID rather than passing tokens themselves through the queue.

## Rate limits are per app *per store*

Zid enforces 60 requests/minute per application per store (leaky-bucket algorithm — see `error-handling.md`). This means:
- The limit is naturally per-tenant already — one very active merchant can't exhaust another merchant's budget.
- But your own code can still trip it accidentally, e.g. a bulk sync job hammering one store's orders endpoint. Rate-limit awareness (queuing, backoff, batching) needs to be per-store in your job scheduler, not a single global limiter shared across all merchants.

## Scaling considerations

- **Webhooks over polling.** Zid explicitly recommends webhooks for tracking order/payment status changes instead of repeated polling, both for your own rate-limit budget and to avoid unnecessary load on Zid's side. Default to webhook-driven sync; use polling only for one-off backfills or where no webhook event exists.
- **Idempotent webhook handling.** Store an event/delivery ID and dedupe, since webhook delivery systems generally retry on non-2xx responses — a handler that isn't idempotent will double-process on retry.
- **Background the token refresh job**, not inline with a user-facing request — refreshing 10,000 merchants' tokens should be a scheduled sweep, not something that happens synchronously during an API call.
- **Isolate one merchant's failure from another's.** A single store with malformed data, a revoked token, or a slow endpoint should not block or slow down processing for other merchants — this usually means per-store job queues or at least per-store error boundaries, not one big synchronous loop over all stores.
