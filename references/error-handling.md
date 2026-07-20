# Error Handling, Rate Limits, and Escalation — Reference

Source of truth: `https://docs.zid.sa/responses.md` and `https://docs.zid.sa/rate-limiting-644369m0.md`. Re-fetch if behavior doesn't match what's documented here.

## Response shape

Zid responses follow a consistent envelope. Success:

```json
{
  "status": 200,
  "success": true,
  "data": { }
}
```

Error:

```json
{
  "status": 422,
  "success": false,
  "error": {
    "code": "validation_failed",
    "message": "Invalid input data.",
    "fields": {
      "email": ["Email format is invalid."]
    }
  }
}
```

Note: error `code`/`message` can come back localized (Arabic) depending on the `Accept-Language` header sent — don't pattern-match on the English string alone; branch on the HTTP status and, where present, the `code` slug.

## Status codes worth handling explicitly (not just "if not 200")

| Status | Slug | Meaning | What to do |
|---|---|---|---|
| 400 | bad_request | Malformed request | Fix request syntax/params — a code bug, not retryable |
| 401 | unauthorized | Missing/invalid auth | Check both `Authorization` and `X-Manager-Token` are present and not expired — see `oauth-flow.md` |
| 403 | forbidden | Authenticated but not permitted | Scope issue — the merchant didn't grant this scope, or app isn't configured for it |
| 404 | not_found | Resource doesn't exist | Check the ID/URL; don't retry |
| 405 | method_not_allowed | Wrong HTTP verb | Code bug — fix the verb |
| 406 | not_acceptable | `Accept` header mismatch | Adjust `Accept` header |
| 410 | gone | Resource permanently removed | Clean up local references; don't retry |
| 422 | validation_failed | Payload failed validation | Read `error.fields` for the exact field-level problems |
| 429 | too_many_requests | Rate limit exceeded | Back off and retry — see below, don't hammer harder |
| 500 | server_error | Zid-side failure | Retry with backoff; if persistent, escalate |
| 503 | service_unavailable | Zid-side maintenance/overload | Retry later with backoff |

## Rate limiting: 60 req/min per app per store

Zid uses a leaky-bucket algorithm: sustained bursts above the limit get throttled/retried internally first, then rejected with 429 if the bucket stays full. Practical implications for your code:

- Implement exponential backoff (with jitter) specifically for 429 — don't retry immediately in a tight loop, that keeps the bucket full.
- Prefer webhooks to polling wherever a webhook event exists for the data you need (explicitly recommended by Zid — see `multi-tenant-architecture.md`).
- For bulk operations (e.g., syncing thousands of products), batch and pace requests deliberately rather than firing them all concurrently — a burst sync is exactly what trips this limit.
- Remember the limit is per app *per store*: one merchant's bulk job shouldn't be throttled by another merchant's traffic, but it can absolutely throttle itself.

## Escalating an issue to Zid's technical team

When a problem can't be resolved from the docs or the patterns above, prepare a minimal report — Zid's team needs exactly enough to reproduce, nothing more:

```markdown
## Issue summary
[One or two sentences: what you expected vs. what happened.]

## Endpoint
METHOD https://api.zid.sa/v1/<path>

## Request (secrets redacted)
curl -X <METHOD> "https://api.zid.sa/v1/<path>" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <REDACTED>" \
  -H "X-Manager-Token: <REDACTED>" \
  -d '<request body if applicable>'

## Response received
Status: <code>
Body: <exact response body, or the error.code/message>

## Expected behavior
[What the docs at <doc URL> say should happen]

## Context
- Store ID: <if shareable>
- Timestamp: <when this occurred, with timezone>
- Frequency: [one-off / consistently reproducible / intermittent]
```

Rules for building this report:
- **Always redact `client_secret`, `Authorization` token values, `X-Manager-Token`/`access_token` values, and refresh tokens** before sharing — replace with `<REDACTED>`, never truncate-and-hope.
- Keep it to exactly these sections — a long narrative buries the one detail (exact endpoint, exact status, exact error code) that lets the Zid team act quickly.
- Link the specific docs.zid.sa page the behavior contradicts, if there is one — makes the discrepancy concrete instead of a vague "it's not working."
- If the issue is OAuth-specific and reproduction is unclear, this is the one case where pointing the developer at `https://bridge.zid.dev` to independently observe the flow and confirm whether the problem is in their code or on Zid's side is useful — see `troubleshooting-tools.md` for the exact framing. Only surface it once the developer is genuinely stuck, and only as something they run themselves in a browser, never as part of the report or the app's code.
