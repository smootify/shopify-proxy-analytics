# shopify-proxy-analytics

Self-hosted Shopify attribution proxy (`/_cks`) for headless storefronts. One lightweight call per session makes Shopify issue first-party cookies and tracking tokens (`_y` / `_s`) — without routing all Storefront API traffic through the worker.

Needed since Shopify deprecated the `shopify_y` / `shopify_s` cookies (April 30, 2026): without a same-domain or centralized proxy, visitor/session attribution in Shopify Analytics degrades.

## 1-click deploy

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/smootify/shopify-proxy-analytics)

1. Click the button, log in to Cloudflare, confirm the deploy.
2. You get a URL like `https://shopify-proxy-analytics.<your-account>.workers.dev`.

No DNS changes needed: the endpoint works cross-origin via CORS, even on a `workers.dev` subdomain. It stays within the Cloudflare free plan (100k requests/day included — at 1 call/session that's plenty).

## Configuration (Cloudflare env vars)

Set in `wrangler.jsonc` under `vars`, or in the dashboard
(Workers & Pages → your Worker → Settings → Variables):

| Variable | Default | Description |
|---|---|---|
| `STOREFRONT_API_VERSION` | `2026-10` | Storefront API version used upstream |
| `ALLOWED_ORIGIN` | `*` | Value of `Access-Control-Allow-Origin`. Use a specific domain (e.g. `https://www.myshop.com`) to restrict callers |

## Webflow setup (custom code)

In **Site Settings → Custom Code → Head section**, before the smootify/storefront script:

```html
<script>window.smootifyCksEndpoint = "https://shopify-proxy-analytics.<your-account>.workers.dev/_cks";</script>
```

Smootify calls it automatically (once per session, only with tracking consent). No other changes needed in Webflow.

## Manual deploy

```bash
npm install
npm run deploy   # requires logged-in wrangler: npx wrangler login
```

## How it works

```
browser ──GET /_cks?shop=xxx.myshopify.com──▶ worker ──POST { shop { name } }──▶ Shopify
browser ◀── Set-Cookie + Server-Timing (_y/_s) ── worker ◀──┘
```

- `shop` is validated against `*.myshopify.com` (anti open-proxy).
- The GraphQL query is **fixed and minimal**, not injectable by the client → fixed cost per hit.
- `Shopify-Storefront-Y/S` headers are forwarded as `X-Shopify-UniqueToken/VisitToken` → visitor/session continuity.
- CORS + `Timing-Allow-Origin` headers for cross-origin reads from the browser.

## Verify

```js
fetch("https://shopify-proxy-analytics.<your-account>.workers.dev/_cks?shop=YOUR-SHOP.myshopify.com", {
  headers: { "X-Shopify-Storefront-Access-Token": "PUBLIC_TOKEN" },
}).then(r => r.headers.get("server-timing")); // must show _y and _s
```
