# zid-api-integration

An AI agent skill for building production-grade apps on [Zid](https://zid.sa) — the Saudi/GCC e-commerce platform. Drop it into Claude, Claude Code, opencode, or any other skill-aware coding agent, and it grounds the agent in Zid's real OAuth flow, multi-tenant token handling, and Merchant API docs instead of letting it guess.

## Why this exists

Zid apps are installed by many independent merchants, each with their own store and tokens. Getting this wrong is easy: mixing up Zid's two auth headers, storing tokens without a store ID, or letting an AI assistant hallucinate an endpoint that doesn't exist. This skill encodes the patterns that prevent those failure modes, sourced directly from [docs.zid.sa](https://docs.zid.sa).

## What's inside

```
zid-api-integration/
├── SKILL.md                              # Core principles, workflow, when to ask vs. proceed
└── references/
    ├── oauth-flow.md                     # Authorization Code flow, token exchange & refresh
    ├── multi-tenant-architecture.md      # Per-merchant token storage, tenant isolation
    ├── error-handling.md                 # Status codes, rate limits, escalation report template
    ├── endpoint-index.md                 # Categorized map of Zid's Merchant/Storefront APIs
    └── troubleshooting-tools.md          # Partner Dashboard & Zid Bridge (OAuth debugging)
```

`SKILL.md` is the entry point an agent reads first; the `references/` files are loaded on demand so the agent only pulls in what's relevant to the task at hand.

## What it covers

- **OAuth 2.0 Authorization Code flow** — the exact redirect, token exchange, and refresh calls, plus the single most common bug in Zid integrations: `Authorization` and `X-Manager-Token` are two different headers from two different fields in the same token response, and both are required on every Merchant API call.
- **Multi-tenant token storage** — a data model keyed by `store_id`, encryption-at-rest expectations, and a rule that makes cross-merchant token leakage structurally hard to introduce by accident.
- **Automatic, proactive token refresh** — refresh tokens are long-lived (~1 year) and expire; the skill pushes for a scheduled refresh job rather than a reactive "retry on 401."
- **Error handling & rate limits** — Zid's response envelope, relevant HTTP status codes, the 60 req/min per-app-per-store leaky-bucket limit, and backoff guidance.
- **A minimal escalation-report template** — for when an issue needs to go to Zid's technical team: endpoint, redacted cURL repro, expected vs. actual, nothing more.
- **An endpoint index** mapping Zid's Merchant APIs, Storefront APIs, Webhooks, and Payments docs to their pages on docs.zid.sa, so the agent looks things up instead of inventing them.
- **Zid Bridge** (`bridge.zid.dev`) — surfaced only as a manual, browser-based OAuth debugging aid when a developer is genuinely stuck, and never wired into application code.

## Source of truth

Everything here is grounded in [docs.zid.sa](https://docs.zid.sa) and its [llms.txt](https://docs.zid.sa/llms.txt) sitemap. The skill instructs the agent to fetch the live doc page before implementing any endpoint, and to flag — rather than paper over — any place where a live page disagrees with what's written here. Docs evolve faster than this repo; if something looks stale, the live docs win.

## Installation

**Claude (claude.ai / Claude Desktop / Cowork)**
Download the [`.skill` release](../../releases) (or zip the folder yourself) and use the "Save skill" option when it's shared in a conversation, or upload it wherever your org manages custom skills.

**Claude Code**
Copy or clone this folder into your project's skills directory (or your global `~/.claude/skills/`), e.g.:
```bash
git clone https://github.com/<your-org>/zid-api-integration.git .claude/skills/zid-api-integration
```

**opencode / other skill-aware agents**
Copy the `zid-api-integration/` folder into whatever directory your agent scans for skills. The skill has no external dependencies — it's markdown only.

## Usage

Once installed, just build normally — mention Zid, an app you're building for a Zid merchant, or a Zid OAuth/API error, and the agent should pull this skill in on its own. A few example prompts:

- "Build an Express backend that handles the Zid OAuth install flow and stores tokens per merchant."
- "Why am I getting a 401 from the Zid orders endpoint even though my token is valid?"
- "Set up a scheduled job to refresh Zid tokens before they expire."
- "I need to report a bug to Zid's technical team — help me put together a repro."

## Scope & limitations

- This skill covers **Merchant API integrations and Partner Dashboard apps** (the OAuth-based, server-side integration model). It touches on Zid Pay/payment-provider integration and Vitrin theme development only enough to point you to the right docs — those are distinct subsystems with their own conventions.
- It does not include ready-made, language-specific code templates by design — patterns are given as pseudocode/cURL so they translate cleanly to whatever stack the agent is generating (Node, Python, PHP, etc.).
- App-type specifics (public App Market app vs. private single-merchant app) are intentionally left for the agent to confirm with you rather than assumed — see `SKILL.md`'s "When to ask vs. proceed."

## Contributing

If you find a place where this skill is stale relative to docs.zid.sa, or a failure mode it doesn't cover yet, open an issue or PR. Please keep additions grounded in an actual docs.zid.sa page (link it) rather than assumed behavior.

## License

Add your license of choice here before publishing.
