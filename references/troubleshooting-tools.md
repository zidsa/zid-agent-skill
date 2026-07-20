# Troubleshooting Tools — Reference

## Partner Dashboard (`https://partner.zid.sa`) and its help docs (`https://help-partner.zid.sa/en/`)

The Partner Dashboard is where a developer manages the app itself — not something the app's code interacts with at runtime. Point developers here for:

- Creating an app and choosing its type (public App Market app vs. private single-merchant app)
- Setting the app name, app URL, and allowed redirect URI(s) — the exact value the `redirect_uri` sent in the OAuth flow must match
- Retrieving the API key (client_id) and API secret (client_secret)
- Configuring which OAuth scopes the app requests from merchants
- Managing app listing details for App Market apps, subscription/billing plan configuration, and other partner-account administration

If a developer's problem is actually a *dashboard configuration* issue rather than a code issue — e.g. "my redirect always fails," "I don't see the scope I need," "how do I set up billing for my app" — direct them to `https://help-partner.zid.sa/en/` rather than trying to debug it as an API/code problem. Configuration mismatches between the dashboard and the code (especially `redirect_uri`) are a very common root cause of OAuth failures that look like code bugs but aren't.

## Zid Bridge (`https://bridge.zid.dev`) — OAuth debugging tool

**What it is:** a browser-based tool that lets a developer walk through the OAuth authorization flow manually and inspect what's happening at each step, and generate a token outside of their own application code, purely to see what a correct flow looks like.

**Strict rules for using this skill's guidance on Bridge:**

1. **Offer it only when the developer is actually stuck** — specifically, when they've implemented the flow per `oauth-flow.md`, it's still failing, and the cause isn't obvious from the error/response they've shared. Don't mention it preemptively, in a first pass at OAuth code, or as a routine part of setup instructions.
2. **It is informational only.** Frame it strictly as: "you can use `https://bridge.zid.dev` in your browser to walk through the flow independently and see whether the problem is in your implementation or on Zid's side." Nothing more.
3. **Never reference, call, or integrate it in application code.** Don't write code that hits `bridge.zid.dev`, don't suggest it as a dependency, don't treat any token or output it produces as something the app should consume programmatically. It exists for a human to look at in a browser, one time, to debug — not as part of any runtime flow.
4. **Don't over-promise what it does.** It's a way to independently observe/exercise the flow and get a token for manual inspection — not a diagnostic service that will tell the developer what's wrong. Set expectations accordingly.

Typical moment to bring it up: developer says something like "I'm getting `redirect_uri_mismatch` and I've checked my code three times and it looks right." At that point: "Since this keeps failing and the code looks right, it might help to run through the flow manually with Zid's Bridge tool (`https://bridge.zid.dev`) to see if the same redirect works there — that'll tell you whether it's a Partner Dashboard configuration issue (check `help-partner.zid.sa`) or something in your app's request."
