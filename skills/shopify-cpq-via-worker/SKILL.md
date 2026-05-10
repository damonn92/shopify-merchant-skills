---
name: shopify-cpq-via-worker
description: Build a Configure-Price-Quote system on a non-Plus Shopify store using a Cloudflare Worker that issues 45-minute single-use discount codes — bypassing the Shopify Functions Plus-only requirement.
when_to_use:
  - You need per-spec dynamic pricing (e.g. shower doors by W×H, custom suits by measurement)
  - The shop is on Basic / Shopify / Advanced — no Shopify Plus
  - 3rd-party CPQ apps cost more than the Worker's $5/month or want a revenue share
not_for:
  - Stores already on Plus (use Cart Transform Functions, simpler/cleaner)
  - Subscriptions/recurring (use Shopify's Subscription APIs)
  - Wholesale tier pricing (use B2B catalogs)
---

## Why

Shopify Functions (Cart Transform / Discount / Pricing) are the "right" way to do server-side per-spec pricing. But:

> `Shop must be on a Shopify Plus plan to activate functions from a custom app`

Plus is $2,300/month. CPQ apps are $50-300/month. Neither is acceptable for a small custom-product line.

**Path G — on-demand discount codes from a Cloudflare Worker** is the workaround:

1. Theme renders a Liquid configurator (`<W>×<H>×<glass>×<finish>`).
2. On Add-to-Cart, theme POSTs the spec to a Worker.
3. Worker recomputes the price server-side (ignoring any client-supplied target — never trust the client for money).
4. Worker creates a single-use 45-min discount code via `discountCodeBasicCreate`, scoped to the variant.
5. Theme redirects to `/discount/CODE?redirect=/cart` — Shopify auto-applies on cart visit.
6. Cart shows base $9,999 minus the discount → real price.
7. Weekly cron sweeps expired CPQ-* codes via `discountCodeDelete`.

Works on every Shopify plan. Total cost: $5/month Cloudflare Worker (free tier covers most stores).

## How

### Architecture

```
Browser                Cloudflare Worker         Shopify
─────────              ─────────────────         ───────
Configurator
  ↓ Add to Cart
  POST /api/cpq-discount
  { spec: {W,H,glass,finish,...}, variantId }
                       ↓
                       computeTargetPrice(spec, variantConfig)
                       ↓
                       discountCodeBasicCreate(
                         code: "CPQ-XXXXXXXX",
                         appliesOnce: true, usageLimit: 1, expires: +45min,
                         productVariantsToAdd: [variantId],
                         valueOff: $9,999 - target
                       )
                       ↓
  ←── { code, expiresAt }
  
  // Clear stuck non-combinable codes first!
  POST /cart/update.js { discount: '' }
  
  redirect → /discount/CODE?redirect=/cart
                                                  ↓
                                                  Shopify validates code,
                                                  applies to cart,
                                                  redirects to /cart
                                                  
Cart shows: base $9,999 - $X discount = real price
Cart line title: "KZG Custom Pricing" (rewritten via theme snippet)
```

### Worker code skeleton (TypeScript)

```typescript
// kzg-cpq-worker/src/index.ts
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);

    // OAuth install (one-time, custom app)
    if (url.pathname === '/install') return handleInstall(url, env);
    if (url.pathname === '/auth/callback') return handleCallback(url, env);

    // Main pricing endpoint
    if (url.pathname === '/api/cpq-discount' && request.method === 'POST') {
      return handleCpqDiscount(request, env);
    }

    // Weekly cron-callable cleanup
    if (url.pathname === '/api/cleanup-cpq-codes') return cleanupCodes(env);

    return new Response('not found', { status: 404 });
  },

  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext): Promise<void> {
    ctx.waitUntil(cleanupCodes(env));
  },
};

async function handleCpqDiscount(req: Request, env: Env) {
  // CORS — pin to your domains
  const origin = req.headers.get('Origin') || '';
  const allowed = ['https://thekzg.com', 'https://<your-permanent-domain>.myshopify.com', /\.shopifypreview\.com$/];
  const corsOk = allowed.some(a => typeof a === 'string' ? a === origin : a.test(origin));
  if (!corsOk) return new Response('forbidden', { status: 403 });

  const body = await req.json() as { spec: Spec, variantId: string, productId: string };

  // Fetch the variant's config metafield
  const cfg = await fetchVariantConfig(body.productId, env);

  // SERVER-SIDE pricing — never trust client
  const target = computeTargetPrice(body.spec, cfg);
  const baseCents = 999900; // $9,999 placeholder
  const discountCents = baseCents - Math.round(target * 100);

  if (discountCents <= 0 || discountCents > baseCents) {
    return new Response(JSON.stringify({ error: 'spec out of range' }), { status: 400 });
  }

  const code = 'CPQ-' + crypto.randomUUID().slice(0, 8).toUpperCase();
  const expiresAt = new Date(Date.now() + 45 * 60 * 1000).toISOString();

  await shopifyAdminMutation({
    query: `mutation ($input: DiscountCodeBasicInput!) {
      discountCodeBasicCreate(basicCodeDiscount: $input) {
        codeDiscountNode { id }
        userErrors { field message }
      }
    }`,
    variables: {
      input: {
        title: `CPQ ${body.spec.W}x${body.spec.H}`,
        code,
        startsAt: new Date().toISOString(),
        endsAt: expiresAt,
        appliesOncePerCustomer: true,
        usageLimit: 1,
        combinesWith: { orderDiscounts: false, productDiscounts: false, shippingDiscounts: true },
        customerSelection: { all: true },
        customerGets: {
          value: { discountAmount: { amount: (discountCents / 100).toFixed(2), appliesOnEachItem: false } },
          items: { products: { productVariantsToAdd: [body.variantId] } },
        },
      },
    },
  }, env);

  return Response.json({ code, expiresAt }, {
    headers: { 'Access-Control-Allow-Origin': origin },
  });
}

function computeTargetPrice(spec: Spec, cfg: VariantConfig): number {
  let p = cfg.pricing.base;
  // Width premium per inch above base
  if (spec.W > cfg.pricing.base_width) {
    p += (spec.W - cfg.pricing.base_width) * cfg.pricing.per_inch_width_above_base;
  }
  // Depth (only if config declares it — defends alcove against tampered "depth" spec)
  if (cfg.pricing.base_depth && spec.depthW) {
    p += (spec.depthW - cfg.pricing.base_depth) * cfg.pricing.per_inch_depth_above_base;
  }
  // Glass / finish / handle premiums from option catalog
  p += (cfg.options.glass[spec.glass]?.premium ?? 0);
  p += (cfg.options.finish[spec.finish]?.premium ?? 0);
  p += (cfg.options.handle[spec.handle]?.premium ?? 0);
  // Add-ons
  for (const a of (spec.addons ?? [])) p += (cfg.options.addons[a]?.premium ?? 0);
  // Tolerance surcharge for non-standard fractional sizes
  if (spec.W % 1 !== 0) p += cfg.pricing.tolerance_surcharge ?? 0;
  return Math.round(p * 100) / 100;
}
```

### Theme integration (Liquid)

Configurator renders a 5-step wizard. On Add-to-Cart:

```javascript
// sections/cpq-configurator.liquid (script block)
async function addToCart(spec, variantId, productId) {
  // 1. Clear any existing discount code (CPQ-* codes are non-combinable; stuck codes block new ones)
  await fetch('/cart/update.js', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ discount: '' }),
  });

  // 2. Add the placeholder $9,999 variant to cart
  await fetch('/cart/add.js', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ items: [{ id: variantId, quantity: 1 }] }),
  });

  // 3. Get the discount code from the Worker
  const r = await fetch('https://kzg-cpq-worker.example.workers.dev/api/cpq-discount', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ spec, variantId, productId }),
  });
  const { code } = await r.json();

  // 4. Redirect — Shopify auto-applies the discount
  window.location.href = `/discount/${code}?redirect=/cart`;
}
```

### Cart-line label rewrite

The auto-generated `CPQ-XXXXXXXX` shows up as a line discount in the cart. Rewrite it via theme snippet:

```liquid
{%- comment -%} snippets/cart-products.liquid (~line 259) {%- endcomment -%}
{%- assign discount_label = line.discount_allocations.first.discount_application.title -%}
{%- if discount_label contains "CPQ-" -%}
  {%- assign discount_label = "KZG Custom Pricing" -%}
{%- endif -%}
<span class="discount-label">{{ discount_label }}</span>
```

## Gotchas

- **Non-combinable codes are SESSION-STICKY.** Customer adds CPQ to cart, then later adds a 2nd CPQ — Shopify rejects the 2nd `applicable: false` because the 1st code is still in `cart.discount_codes`. ALWAYS clear via `POST /cart/update.js {discount: ''}` before redirecting to a new `/discount/CODE`.
- **Don't trust client-supplied `_cpq_target_price`.** Recompute every time on the Worker. The whole point of server-side pricing is that the client can't tamper.
- **45-min expiry is a UX trade-off.** Too short and customers checking out slowly fail. Too long and you accumulate stale codes. 45 min works for ~95% of buyers.
- **Weekly cleanup is real.** Without it, you'll hit Shopify's 100k discount-code limit. Cron `0 3 * * 1` deletes any CPQ-* code with `endsAt < now`.
- **Custom app, NOT public.** This Worker authenticates as a custom app (one shop only). For multi-tenant, refactor to per-shop OAuth + KV storage.

## Reference

- **KV namespace:** stores offline access tokens per shop (`TOKENS` namespace)
- **CORS allowlist:** must include `*.shopifypreview.com` for theme editor previews
- **Discount config:** `combinesWith.shippingDiscounts: true` keeps shipping promos working alongside CPQ codes
- **Real product GIDs (KZG, 8 live variants):** `8567134027913` (alcove-sliding) through `8582619168905` (corner-swing-x)
- **Metafield definition:** `kzg_cpq.config` JSON, pinned on Product, holds full pricing schema per variant

## Anti-pattern

Don't try to compute price client-side and send it as the "true" price. The client always wins (Postman, browser console, or just a saved fetch). The Worker recomputes from spec → cfg every single time.

Don't deploy this on Plus when you have it — Shopify Cart Transform Functions are cleaner and don't pollute your discount-code admin with thousands of CPQ-* codes.
