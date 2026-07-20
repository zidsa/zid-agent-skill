# Endpoint Index — Reference

This is a categorized map of what exists in Zid's docs, built from `https://docs.zid.sa/llms.txt`, so you can find the right page fast instead of guessing a URL or a field name.

**How to use this:** find the category below, fetch the listed page(s) for the exact request/response schema, then implement. This index tells you *what exists and roughly where* — it does not replace fetching the live page for parameters, required fields, and response bodies.

**If something isn't listed here:** fetch `https://docs.zid.sa/llms.txt` directly — it's the full, current sitemap and is more likely to be up to date than this file. Don't assume an endpoint doesn't exist just because it's missing from this index.

## Getting started / app basics
- Overview: `docs.zid.sa/zid-apps-overview.md`
- Start here / create your first app: `docs.zid.sa/start-here.md`, `docs.zid.sa/create-first-app.md`
- Authorization (OAuth) — see `oauth-flow.md` in this skill, primary source `docs.zid.sa/authorization.md`
- Responses / error format — see `error-handling.md`, primary source `docs.zid.sa/responses.md`
- Rate limiting — see `error-handling.md`, primary source `docs.zid.sa/rate-limiting-644369m0.md`
- Embedded apps: `docs.zid.sa/embedded-apps-2168216m0.md`
- Zid's own MCP server: `docs.zid.sa/our-custom-mcp-server-1363315m0.md`

## Merchant APIs (require Authorization + X-Manager-Token, base `https://api.zid.sa/v1`)

**Orders & fulfillment**
- Orders (list/view/create/update status/comments/credit notes): `docs.zid.sa/orders-1934403f0.md` and linked pages, e.g. `list-of-orders.md`, `view-order-12479054e0.md`, `create-order-20612055e0.md`, `update-order-by-id.md`
- Reverse orders / returns / refunds: `docs.zid.sa/reverse-orders-1810994f0.md` and linked pages (`create-reverse-orders.md`, `calculate-reverse-totals-27460874e0.md`, `create-refund-for-reverse-order-27460938e0.md`, etc.)
- Abandoned carts: `docs.zid.sa/abandoned-carts-3037527f0.md` (`list-abandoned-carts.md`, `get-abandoned-cart-details.md`)

**Products**
- Product CRUD: `docs.zid.sa/managing-products-1810975f0.md` (`create-a-new-product.md`, `retrieve-a-list-of-products.md`, `get-a-product-by-id.md`, `update-an-existing-product.md`, `bulk-update-product.md`, `delete-a-product.md`)
- Categories: `docs.zid.sa/product-categories-1810983f0.md`
- Attributes & presets: `docs.zid.sa/product-attributes-1810973f0.md`, `product-attribute-presets-1810974f0.md`
- Variants & custom options/inputs: `docs.zid.sa/product-variants-1810980f0.md`
- Images: `docs.zid.sa/product-images-1810976f0.md`
- Stock: `docs.zid.sa/product-stock-1810977f0.md`
- Sorting: `docs.zid.sa/product-sorting-1810978f0.md`
- Import/export (CSV/xlsx): `docs.zid.sa/product-export-1810979f0.md`
- Digital products/vouchers/downloadables: `docs.zid.sa/digital-vouchers-1810993f0.md`, `docs.zid.sa/digital-products-2390702f0.md`
- Reviews & Q&A: `docs.zid.sa/product-reviews-2179631f0.md`, `product-questions-answers-1810995f0.md`
- Availability notifications, badges: `docs.zid.sa/product-availability-notifications-1810996f0.md`, `product-badge-1810992f0.md`

**Inventory & shipping**
- Locations/stock-by-location: `docs.zid.sa/inventories-1810981f0.md`
- Shipping methods: `docs.zid.sa/shipping-1810986f0.md`

**Marketing**
- Coupons: `docs.zid.sa/coupons-1810985f0.md`
- Discount rules: `docs.zid.sa/list-discount-rules-37830329e0.md` and linked pages
- Bundle offers: `docs.zid.sa/bundle-offers-1934530f0.md`
- Loyalty program: `docs.zid.sa/loyalty-program-1810991f0.md`

**Customers**
- Customer CRUD, tags, bulk import: `docs.zid.sa/customers-1810970f0.md` and linked pages

**Store settings & account**
- Manager profile & permissions: `docs.zid.sa/get-manager-profile.md`, `-user-roles-and-permissions-604944m0.md`
- Store profile/branding/localization/social/operations/KYC: `docs.zid.sa/store-33664558e0.md` and sibling pages under "Account > Store"
- VAT, payment methods, operating countries: `docs.zid.sa/store-settings-1810968f0.md`
- Countries & cities: `docs.zid.sa/countries-and-cities-1810988f0.md`

**Blogs** (merchant-managed content)
- Settings/categories/posts/media/tags: `docs.zid.sa/blogs-endpoints.md` and linked pages
- Public storefront blog read endpoints: pages under "Blogs > Storefront"

**App management**
- Subscription details, usage-based billing: `docs.zid.sa/subscription-details-13896876e0.md`, `update-usage-based-charges-13896680e0.md`
- App lifecycle events: `docs.zid.sa/events-878234m0.md`

## Webhooks
- Overview & subscribing: `docs.zid.sa/webhooks.md`, `docs.zid.sa/list-of-webhooks.md`, `create-a-webhook.md`, `delete-a-webhook-by-original-id.md`
- Health tracking / recovering broken webhooks: `docs.zid.sa/webhook-health-tracking-2197281m0.md`, `health-summary-37247358e0.md`, `broken-webhooks-37378513e0.md`, `recover-broken-webhooks-37378520e0.md`
- Event payload references by domain: Order (`webhook-events-order.md`), Product (`webhook-events-product.md`), Abandoned Cart (`webhook-events-abandoned-cart.md`), Customer (`webhook-events-customer.md`), Product Category (`webhook-events-product-category.md`)
- App uninstall event — critical for token cleanup, see `oauth-flow.md`

## Storefront-facing APIs (customer-authenticated, different auth model than Merchant APIs)
- Auth (SMS/WhatsApp/email login, register): pages under "API's > Authentication"
- Products, categories, search: pages under "API's > Products", "API's > Categories"
- Checkout / cart / coupons / gift cards / loyalty redemption: pages under "API's > Checkout"
- Customer account, addresses, orders, wishlist: pages under "API's > Account"
- Storefront pages/blogs/scripts: pages under "API's > Storefront"

These use customer-session auth, not the Authorization/X-Manager-Token pair used by Merchant APIs — don't mix the two auth models. Fetch the specific page before assuming which applies.

## Payments (Zid Pay / payment provider integration)
- Overview, embedded payment, gateway error codes: `docs.zid.sa/overview-1340608m0.md`, `embedded-payment-1030692m0.md`, `gateway-error-codes-1042597m0.md`
- Direct payment, execute payment, payment status: `docs.zid.sa/direct-payment-17957496e0.md`, `execute-payment-request-17112105e0.md`, `get-payment-status-17286396e0.md`
- Refunds: `docs.zid.sa/request-refund-17286541e0.md`
- Apple Pay: `docs.zid.sa/applepay-3801359f0.md` and linked pages
- Payment-related webhooks (merchant link, payment paid, refund): `docs.zid.sa/link-merchant-event-17956108e0.md`, `payment-paid-event-17956148e0.md`, `refund-event-17956149e0.md`

This is its own integration category (you're building a *payment provider* for Zid, not just calling the Merchant API) — if the user's task looks like this, confirm that's actually the goal before assuming it maps onto the standard OAuth app flow.

## Theme / Vitrin development (storefront theming, not the Merchant API)
Distinct subsystem for building/customizing store themes (Jinja templates, theme editor, Vitrin CLI). See pages under "Getting Started", "Key Concepts", "Building with Vitrin", "Vitrin CLI" in `llms.txt`. Only relevant if the task is about storefront theme/UI customization rather than backend integration — confirm which the user means, since "building on Zid" could refer to either.
