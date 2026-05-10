---
name: shopify-giga-dropshipping
description: Importing US-warehouse big-and-bulky products from GigaCloud B2B (shop.gigab2b.com) into Shopify — via direct API loop (10× faster than UI clicks), with the shop_id pinning trap that breaks multi-store accounts.
when_to_use:
  - Bulk-importing GIGA products (shower doors, furniture, appliances) into a Shopify store
  - GIGA's "Batch Import" silently lands only 1 of 50 selected products
  - Single GIGA account is bound to multiple Shopify stores and items get cross-imported
not_for:
  - Dropshipping from non-US suppliers (DSers, Spocket use different APIs)
  - Print-on-demand integrations (Printful, Printify)
---

## Why

GigaCloud's **Shopify embedded app** at `shop.gigab2b.com` lets you browse a US-warehouse catalog and push products into Shopify. The intended workflow is "click + Add to Import List → Select All → Batch Import." It is broken in two ways:

1. **Element Plus buttons reject `.click()`.** The DOM has `event.isTrusted` checks. Any Playwright/Puppeteer/JS automation hits a wall.
2. **Multi-product batch import silently drops all but the first.** The internal API endpoint accepts `productInfoList: [...]` array but only inserts the first item. **No error returned.**

The fix: hit the internal API directly from the page console, **one product at a time**, paced. 10× faster than UI even with the loop overhead, and 100% reliable.

The OTHER trap: a single GIGA account can be bound to multiple Shopify stores. The push API accepts `shopIdList: [...]`. If you forget to pin to ONE shop, products land in the wrong store.

## How

### Setup

1. Install GIGA app on each Shopify store you want to source for. Each install gets a unique `shopId` (e.g. KZG = `1650`, Calivell = different).
2. **Critical:** discover your `shopId`s via:
   ```js
   // In console at shop.gigab2b.com
   await fetch('/shopify-app/shop/list', { credentials: 'include' }).then(r=>r.json())
   // → { code:0, data:[{id:1650, shop:"<your-permanent-domain>.myshopify.com"}, ...] }
   ```
   Pin the right one in your script — wrong shopId silently imports to wrong store.

### Endpoint reference (all `credentials: 'include'`, same-origin)

| Action | Endpoint | Body | Notes |
|---|---|---|---|
| List import-list | `POST /shopify-app/import/query/list` | `{current:1, size:300, keyWord:""}` | Returns `data.records[]` with `importId` (DELETE/PUSH key) and `productId` (catalog key) |
| Add 1 product | `POST /shopify-app/import/add/product` | `{productInfoList:[{productId}], sync:1}` | **Loop one at a time** with ~150ms delay |
| Delete | `DELETE /shopify-app/import/delete` | `[importId, importId, ...]` | Array of importIds, NOT productIds |
| Push to Shopify | `POST /shopify-app/importPush/execute` | `{importIdList:[...], shopIdList:[<YOUR_SHOP_ID>], productPushStatus:1, isDeleteImport:true}` | Up to ~50 importIds per push |
| Batch status | `POST /shopify-app/notification/list` | `{current:1, size:5}` | Look for `title === "Batch import completed"`, `content === "X succeeded and Y failed"` |

### Working pattern (paste into shop.gigab2b.com console)

```js
const SHOP_ID = 1650; // YOUR pinned shop ID — verify before running
const PRODUCT_IDS = [/* list of catalog product IDs to import */];

const sleep = ms => new Promise(r => setTimeout(r, ms));

// Step 1 — add to Import List, ONE AT A TIME
async function addAll() {
  for (const pid of PRODUCT_IDS) {
    const r = await fetch('/shopify-app/import/add/product', {
      method: 'POST',
      credentials: 'include',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({ productInfoList: [{ productId: pid }], sync: 1 }),
    }).then(r => r.json());
    if (r.code === 12002) console.log(`  ${pid}: already in list (skip)`);
    else if (r.code === 12022) console.log(`  ${pid}: gated SKU (skip)`);
    else if (r.code === 0) console.log(`  ${pid}: added`);
    else console.log(`  ${pid}: code ${r.code} — ${r.message}`);
    await sleep(150);
  }
}

// Step 2 — get importIds we just created
async function getImportIds() {
  const r = await fetch('/shopify-app/import/query/list', {
    method: 'POST', credentials: 'include',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({ current: 1, size: 300, keyWord: '' }),
  }).then(r => r.json());
  return r.data.records.map(x => x.importId);
}

// Step 3 — push to Shopify in batches of ~50
async function pushBatched(importIds) {
  for (let i = 0; i < importIds.length; i += 50) {
    const batch = importIds.slice(i, i + 50);
    const r = await fetch('/shopify-app/importPush/execute', {
      method: 'POST', credentials: 'include',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({
        importIdList: batch,
        shopIdList: [SHOP_ID],   // PIN to single shop
        productPushStatus: 1,
        isDeleteImport: true,
      }),
    }).then(r => r.json());
    if (r.code === 500 && /batch import task in progress/i.test(r.message)) {
      console.log('  another batch running, waiting 60s...');
      await sleep(60000);
      i -= 50; continue; // retry this batch
    }
    console.log(`  pushed batch ${i / 50 + 1} (${batch.length} items): code ${r.code}`);
    await sleep(2000);
  }
}

// Run
addAll().then(getImportIds).then(pushBatched);
```

### Pacing reality

GIGA's batch processor handles ~10 products/min. So:
- 50-item batch ≈ 5 min
- 100-item batch ≈ 10 min
- Push batches **sequentially** — concurrent pushes return `code: 500 "currently a batch import task in progress"`.

### Verifying success — DON'T trust GIGA's count

GIGA reports "X succeeded and Y failed" but **failures often still land in Shopify** (their count is pessimistic). Check the actual Shopify state:

```graphql
query {
  productsCount(query: "vendor:KZG") { count }
}
```

If the count went up by ~your batch size, you're good.

## Post-processing — required after every batch

**GIGA imports land with broken state.** Five things break checkout / SEO / publish status. Run all five scripts after every batch (idempotent):

1. **Classify + markup pricing** — assign products to collections by title regex, apply 1.5× wholesale → retail markup, set `compareAtPrice = retail × 1.2`.
2. **Publish to channels** — `publishablePublish` to Online Store + Shop + Google & YouTube + Facebook & Instagram. **Without this, products are invisible to Google and storefront filters.** GIGA imports with `status: ACTIVE` but `publishedAt: null`.
3. **Set variant weights** — GIGA imports with `weight=0`. Shopify checkout breaks: "Items in the cart do not meet price or weight requirements." Fix via `inventoryItemUpdate` with size-aware estimates (60in vanity → 180 lbs, etc.).
4. **AI listing optimization** — Claude Haiku rewrites title/SEO/description/alt text. Idempotency marker `<!-- kzg-optimized-v1 -->` on `descriptionHtml` prevents re-running.
5. **Trim media + set namespaced tags** — GIGA images already 2000×2000 pure-white, no need for background removal. Just keep first 7. Add `size:60in`, `finish:matte-black`, `style:frameless`, etc. for storefront filtering.

See [`shopify-listing-optimization`](../shopify-listing-optimization/SKILL.md) for the full 9-script pipeline.

## Gotchas

- **Card-extracted productId:** read attribute `data-gmd-attr-product_id` on `.card-outter-container` — saves a search API call.
- **Search terms that work:** `shower+door`, `freestanding+bathtub`, `acrylic+bathtub`, `led+bathroom+mirror`, `bathroom+vanity`. **Don't use** `smart+toilet` / `dual+flush` (returns over-toilet cabinets), `bathroom+faucet` (returns vanities — GIGA doesn't carry standalone faucets).
- **Cross-store pollution:** if your GIGA account is shared between two Shopify stores, the Import List is shared too. Inspect for off-brand items before pushing.
- **Optimizer wastes API calls if you delete products mid-pipeline.** Run delete BEFORE optimize, not after.
- **Variant weight estimates** — be conservative: standard parcel rates require non-zero weight. Better to overestimate (heavier shipping rate) than underestimate (free shipping = company eats the difference).

## Reference

- GIGA admin: https://shop.gigab2b.com (Shopify embedded app, separate login from Shopify itself)
- Pricing strategy that works: 1.5× wholesale → retail markup, 1.2× retail → compareAtPrice (~17% Sale)
- Image source: GIGA images are 2000×2000 white-bg, no watermarks, no need for background removal
- Real outcome on KZG: 188 vendor:KZG products, 175 ad-ready, 98.3% pass full Google/Meta audit

## Anti-pattern

Don't try to use the official Element Plus button click via `document.querySelector('.add-to-import-btn').click()`. The library checks `event.isTrusted` and rejects synthetic clicks. **Use the API directly.**
