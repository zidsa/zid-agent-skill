---
name: zid-api-integration
description: Build production-grade apps and integrations on the Zid e-commerce platform (Saudi/GCC merchant platform) — OAuth 2.0 authorization, multi-tenant merchant apps, Merchant APIs (orders, products, customers, coupons, webhooks, etc.), and Partner Dashboard apps. Use this skill whenever the user mentions Zid, zid.sa, docs.zid.sa, "Zid app", "Zid API", "Zid OAuth", "Zid partner app", "Zid App Market", connecting/integrating with a Zid store, or building something a Zid merchant will install. Also trigger for troubleshooting Zid auth errors (401/403, token refresh failures, X-Manager-Token issues) or preparing a bug report for the Zid technical team. Always consult this skill before writing any Zid API code — it prevents hallucinated endpoints and broken multi-tenant token handling.
---

# Zid API Integration

Building a Zid app means one thing above all: **many merchants will install this app, each with their own store, tokens, and data.** Every architectural decision flows from that fact. A demo that works for one store and breaks under two is not done.

## Ground rule: docs.zid.sa is the only source of truth

Zid's API surface is large and evolves. Anthropic's training data is not a reliable source for exact Zid endpoint paths, request/response shapes, or field names.

- Before implementing **any** endpoint, fetch its page from `https://docs.zid.sa/` (the endpoint index in `references/endpoint-index.md` maps feature areas to doc URLs) or check `https://docs.zid.sa/llms.txt` for the full sitemap if the endpoint isn't in the index yet.
- Never invent a field name, endpoint path, header, or status code. If you can't find it in the fetched docs, say so explicitly and ask the user, rather than guessing plausibly.
- If a fetched doc page is ambiguous, contradicts what's in this skill, or seems to have changed, trust the freshly-fetched page — it's newer than this skill — and flag the discrepancy to the user.
- Base URLs (confirmed from docs, but re-verify if a call fails unexpectedly):
  - Merchant API: `https://api.zid.sa/v1`
  - OAuth server: `https://oauth.zid.sa`
  - Only `v1` is current; anything else is deprecated.

**When uncertain, ask.** Guessing at scopes, an app's public/private type, which token to use, or a data-modeling decision produces code that looks right and fails in production. A short clarifying question is cheaper than a bad default here — see "When to ask vs. proceed" below.

## Core principles this skill optimizes for

1. **Multi-tenant by default** — assume 2 to 50,000 merchants from day one; never design storage or state around a single store.
2. **OAuth 2.0 Authorization Code flow** — Zid uses the standard confidential-client grant (RFC 6749 §4.1). This is server-side only; the Client Secret must never reach a browser or mobile client.
3. **Secure token storage** — tokens are long-lived (1 year) high-value credentials, not session cookies. Treat them like passwords.
4. **Automatic token refresh** — refresh proactively, not reactively on a 401.
5. **Production-ready architecture over demo code** — error handling, retries, and observability are not "later" work.
6. **Scalability** — the same code path serves merchant #1 and merchant #10,000.
7. **Official Zid documentation only, never hallucinated APIs** — see ground rule above.
8. **Explain *why*** — when you recommend a pattern, say why it exists (e.g., "queue this webhook handler because Zid enforces 60 req/min per app per store, so synchronous fan-out will trip rate limits").

## When to ask vs. proceed

Ask a short clarifying question before writing code when:
- The app type (public App Market app vs. private single-merchant app) isn't stated — it changes the OAuth redirect setup and where credentials live.
- Required scopes aren't specified — don't guess which Merchant API areas (orders, products, webhooks, etc.) the app needs.
- The token storage backend isn't specified (Postgres, Redis, managed secrets store, etc.) — don't silently pick one for a production integration.
- The user reports an error and the error code/response body hasn't been shared yet — ask for it rather than guessing the cause.

Proceed with a stated, reasonable assumption (and say what you assumed) for smaller implementation details — e.g., naming a database column, choosing a retry backoff curve, structuring a webhook queue.

## Workflow

1. **Clarify the shape of the app** — public (App Market, multi-merchant, needs Partner Dashboard app registration) or private (single merchant)? This is set up in the [Partner Dashboard](https://partner.zid.sa). See `references/troubleshooting-tools.md` for what's configurable there.
2. **Implement OAuth** — read `references/oauth-flow.md` before writing any authorization code. It covers the exact redirect, token exchange, refresh, and the Authorization-token vs. X-Manager-Token distinction that is the single most common source of bugs in Zid integrations.
3. **Design multi-tenant storage** — read `references/multi-tenant-architecture.md` for the token/store data model, encryption-at-rest expectations, and the uninstall-webhook cleanup pattern.
4. **Call Merchant APIs** — look up the exact endpoint in `references/endpoint-index.md`, fetch the live doc page for the request/response schema, then implement. Wrap every call through the error-handling pattern in `references/error-handling.md` so failures produce actionable messages instead of raw stack traces.
5. **Handle errors and rate limits gracefully** — see `references/error-handling.md`. Zid enforces 60 requests/minute per app per store (leaky-bucket algorithm) and prefers webhooks over polling for status changes.
6. **If the user gets stuck on OAuth specifically**, and only then, mention the Zid Bridge testing tool (`https://bridge.zid.dev`) as a way to *observe* the OAuth flow and generate a token for manual inspection — see the strict rules on this in `references/troubleshooting-tools.md`. It is a debugging aid the developer uses in a browser; it is never referenced or called from application code.
7. **If an issue needs escalation to Zid's technical team**, produce the minimal, complete report described in `references/error-handling.md` — endpoint, method, redacted cURL repro, issue summary, expected vs. actual — instead of a long narrative.

## Applying the "AI-assisted development" discipline to Zid apps

Zid integrations are exactly the kind of project where non-deterministic AI coding help needs deterministic guardrails, for the same reasons that apply to any complex, long-lived codebase:

- **Convert soft guidance into hard checks.** Don't rely on "remember to refresh tokens" as a comment — write a scheduled job or middleware that refreshes any token within N days of `expires_in`, and a linter/test that fails if a new API call site is added without going through the shared authenticated-request wrapper.
- **Enforce tenant isolation deterministically**, not by convention. Every query or token lookup should be forced through a function that requires a store/merchant identifier — make it structurally impossible to accidentally use merchant A's token for merchant B's request, rather than trusting each call site to remember.
- **Keep a scoped reference doc, not one giant file.** For a real project, mirror this skill's structure: a short root doc plus focused files per concern (`oauth.md`, `webhooks.md`, `rate-limits.md`) so both the AI and the next developer load only what's relevant.
- **Isolate and test the token/auth layer specifically.** It's the highest-blast-radius part of the system (a bug here can leak one merchant's data to another) — cover it with tests before building features on top of it.
- **Keep files focused and commits small**, especially around the OAuth/token module, so changes there are easy to review carefully rather than skimmed.

## Reference files

- `references/oauth-flow.md` — full Authorization Code flow: redirect, token exchange, refresh, the Authorization vs. X-Manager-Token distinction, token lifetimes, uninstall handling.
- `references/multi-tenant-architecture.md` — data model for storing per-merchant credentials, encryption expectations, scaling considerations.
- `references/error-handling.md` — HTTP status/error shapes, rate limiting behavior, retry patterns, and the minimal-report template for escalating to Zid's technical team.
- `references/endpoint-index.md` — categorized index of Merchant API and Storefront API areas mapped to their docs.zid.sa pages, plus how to use `llms.txt` to find anything not listed.
- `references/troubleshooting-tools.md` — Partner Dashboard (help-partner.zid.sa) capabilities and the Zid Bridge OAuth debugging tool, with the rules on when and how (not) to mention it.
