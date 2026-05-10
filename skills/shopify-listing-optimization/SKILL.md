---
name: shopify-listing-optimization
description: The 9-script pipeline that takes raw dropshipped products from "imported but broken" to "98% pass Google + Meta ad audit." Markup → publish to 5 channels → set weights (CRITICAL for checkout) → AI title/SEO → barcodes → namespaced tags → media reorder → audit → fix gaps.
when_to_use:
  - You just imported 20-200 products from a dropshipping source (GIGA, Aliexpress, etc.)
  - Products show up in Shopify but don't appear in Google Shopping or sitemap
  - Customers report "Items don't meet weight requirements" at checkout
  - You're prepping a store for paid Google + Meta ads and need full catalog readiness
not_for:
  - Optimizing a single hand-curated product (overkill — just edit it manually)
  - Stores with strict brand-voice requirements that can't tolerate AI-rewritten copy (run optimizer with manual review pass)
---

## Why

Bulk-imported products from any source land with at least 5 silent breakages:

| Breakage | Symptom | Fix |
|---|---|---|
| `weight = 0` on every variant | Checkout fails: "Items don't meet weight/price for shipping" | `inventoryItemUpdate` with size-aware regex estimates |
| `publishedAt = null` | Not in `/sitemap_products_1.xml`, invisible to Google | `publishablePublish` to 5 channels |
| Templated description ("hot sale, fast shipping") | Duplicate-content SEO penalty across 100+ products | Claude Haiku rewrite with idempotency marker |
| No `mm-google-shopping` metafields | Google Merchant Center rejects the feed | `metafieldsSet` with category/condition/age_group/gender |
| Empty/UUID-style tags | Storefront filter sidebar empty | Regex extraction from title + variant titles into namespaces |

This skill is the operational pipeline that took KZG from 70 imported products to **172/175 (98.3%) passing full Google + Meta audit**.

## How

### The 9-script pipeline

Run in this exact order. All idempotent. Stash in `~/bin/<store>-blog/` (NOT `/tmp/` which gets cleaned on reboot):

| # | Script | What it does | Why this order |
|---|---|---|---|
| 1 | `postprocess.py` | Classify by title regex into 5-6 collections, apply 1.5× wholesale → retail markup, set `compareAtPrice = retail × 1.2` | Must classify before publishing so collection membership is right |
| 2 | `publish_all.py` | `publishablePublish` to Online Store + Shop + Google & YouTube + Facebook & Instagram + Pinterest | Without this, optimizer rewrites SEO of products invisible to Google |
| 3 | `set_weights.py` | Size-aware regex weight estimator | **CRITICAL: zero weight breaks checkout entirely** |
| 4 | `optimize.py` | Claude Haiku rewrites title/SEO/description/alt text + trims media to 7 | Heaviest API cost; run after products are confirmed live |
| 5 | `assign_barcodes.py` | Pull GTIN-13 codes from a Google Sheet pool, assign in createdAt order | Required for Google Shopping feed |
| 6 | `set_type_and_tags.py` | Set `productType` + namespaced tags (`size:60in`, `finish:matte-black`, etc.) | Drives storefront filter sidebar |
| 7 | `enrich_collections.py` | One-shot Claude generator for each collection's `descriptionHtml` (~2,200 char structured prose) | Gives collection pages SEO depth |
| 8 | `reorder_media.py` | Detect catalog (white-bg) vs lifestyle shots via 4-corner brightness, put catalog at position 1 | Cleans up collection-grid first impression |
| 9 | `audit_listings.py` + `fix_listing_gaps.py` | Read-only audit across Google + Meta + Shopify storefront fields → automated fixes for everything fixable | Final pass; loop until % passing plateaus |

### Pricing strategy that works

```python
# Wholesale → retail
new_price = wholesale * 1.5
compare_at = new_price * 1.2  # ~17% Sale ribbon

# Idempotency: skip variants that already have compareAtPrice set
if any(v.get('compareAtPrice') for v in variants):
    continue
```

With per-collection retail caps:
- Outdoor furniture: max $1499
- Bedroom/kids: $1199
- Office: $1299
- Dining/living: $999

When 1.5× exceeds cap, retail clamps to `cap - 0.5` (still rounded to `.99`).

### Variant weight estimator (the one that saves checkout)

```python
def estimate_weight(title):
    t = title.lower()
    # Order matters: shower-door checked BEFORE mirror (some have "mirror" in title)
    if re.search(r'shower\s*door|tub\s*door', t):
        return 95 if 'tub' in t else 110 if 'sliding' in t else 95
    if re.search(r'walk-?in.*screen|shower\s*screen', t):
        return 65
    if 'toilet' in t and 'led' not in t:
        return 90
    if 'led' in t and 'mirror' in t:
        # 18-38 lbs by size
        m = re.search(r'(\d+)\s*(?:in|"|inch)', t)
        size = int(m.group(1)) if m else 28
        return 18 + (size - 16) * 0.7
    if 'mirror' in t:
        return 25
    if 'bathtub' in t or re.search(r'(freestanding|soaking|acrylic)\s*tub', t):
        m = re.search(r'(\d+)\s*(?:in|")', t)
        size = int(m.group(1)) if m else 55
        return 110 if size <= 43 else 150 if size <= 55 else 170 if size <= 59 else 200
    if 'vanity' in t:
        m = re.search(r'(\d+)\s*(?:in|")', t)
        size = int(m.group(1)) if m else 36
        # 24in→75, 30in→90, 36in→110, 48in→150, 60in→180, double-60→240
        base = 60 + (size - 24) * 3
        if 'double' in t and size >= 60: base += 60
        return min(base, 240)
    if 'medicine cabinet' in t:
        return 30
    return 50  # safe default
```

Apply via:
```graphql
mutation ($variantId: ID!, $weight: Float!) {
  inventoryItemUpdate(id: $inventoryItemId, input: {
    measurement: { weight: { value: $weight, unit: POUNDS } }
  }) { userErrors { field message } }
}
```

### AI listing optimizer pattern

```python
import anthropic
client = anthropic.Anthropic()

system = """Rewrite this Shopify product for SEO and conversion.
Output strict JSON: { "title", "description_html", "seo_title", "seo_description", "image_alt_base" }.
Rules:
- Title ≤ 70 chars, lead with keyword
- description_html: 2-3 paragraphs + 4-6 bullets
- seo_title ≤ 60 chars, brand suffix at end
- seo_description: 145-200 chars, end with "Free US shipping on orders over $X."
- image_alt_base: descriptive base used as "{alt} — image N"
- Use "in." or "inch" not " (the inch mark breaks JSON parsing)
"""

# Idempotency marker — DON'T re-run on already-optimized products
SIG = '<!-- {brand}-optimized-v1 -->'
if SIG in current_description: skip()

# Batch with claude-haiku-4-5 — cheap, fast, good enough
resp = client.messages.create(
    model='claude-haiku-4-5',
    max_tokens=1500,
    system=system,
    messages=[{ "role": "user", "content": json.dumps({"title": p.title, "description": strip_html(p.descriptionHtml), "vendor": p.vendor})}],
)
parsed = json.loads(resp.content[0].text)
parsed['description_html'] += '\n' + SIG  # mark for next run
```

### Namespaced tag pattern

```python
# Drives storefront filter sidebar via theme grouping by `:` prefix
TAG_NAMESPACES = {
    'size': r'\b(\d+\.?\d*)\s*(?:in|"|inch)\b',
    'finish': ['matte-black', 'brushed-nickel', 'chrome', 'polished-chrome', 'brushed-gold', 'antique-bronze', 'stainless-steel', 'gloss-white'],
    'style': ['frameless', 'semi-frameless', 'framed', 'sliding', 'pivot', 'swing', 'bypass', 'walk-in', 'screen', 'neo-angle', 'single', 'double', 'freestanding', 'wall-mount', 'floating'],
    'feature': ['ada', 'dual-flush', 'tornado-flush', 'siphonic', 'comfort-height', 'elongated', 'round-bowl', 'soft-close', 'anti-fog', 'dimmable', 'backlit', 'led', 'smart', 'heated-seat', 'bidet', 'tankless'],
}

# Strip-and-rebuild ONLY namespaced tags (preserves manual non-namespaced tags)
new_tags = [t for t in current_tags if ':' not in t]  # preserve non-namespaced
for ns, patterns in TAG_NAMESPACES.items():
    if isinstance(patterns, str):
        m = re.search(patterns, title, re.I)
        if m: new_tags.append(f'{ns}:{m.group(1)}in')
    else:
        for p in patterns:
            if re.search(rf'\b{p}\b', title, re.I) or any(re.search(rf'\b{p}\b', vt, re.I) for vt in variant_titles):
                new_tags.append(f'{ns}:{p}')
```

### Sales channel publishing — all 5

```graphql
# Get channel IDs once per shop, hardcode for speed
mutation ($id: ID!, $publicationIds: [ID!]!) {
  publishablePublish(id: $id, input: { publicationIds: $publicationIds })
}

# Channels (KZG-specific IDs — discover via channels query):
# Online Store: gid://shopify/Publication/<store-OS-id>
# Shop:         gid://shopify/Publication/<store-Shop-id>
# Google & YouTube: ...   (required for Google Shopping ads)
# Facebook & Instagram: ... (required for Meta ads)
# Pinterest:    ...        (bonus, low effort high reward)
```

Detect missing channels per-product via `resourcePublicationsV2` and only publish the deltas — idempotent on re-run.

### Google Shopping metafields

```graphql
mutation {
  metafieldsSet(metafields: [
    { ownerId: "gid://shopify/Product/<id>", namespace: "mm-google-shopping",
      key: "google_product_category", type: "single_line_text_field",
      value: "Home & Garden > Plumbing Fixtures > Showers > Shower Doors" },
    { ownerId: "gid://shopify/Product/<id>", namespace: "mm-google-shopping",
      key: "condition", type: "single_line_text_field", value: "new" },
    { ownerId: "gid://shopify/Product/<id>", namespace: "mm-google-shopping",
      key: "age_group", type: "single_line_text_field", value: "adult" },
    { ownerId: "gid://shopify/Product/<id>", namespace: "mm-google-shopping",
      key: "gender", type: "single_line_text_field", value: "unisex" }
  ]) { userErrors { field message } }
}
```

Sets up Google Channel install to be a one-click action when the user is ready.

## Gotchas

- **Order matters.** Skip step 3 (weights) and customers can't check out. Skip step 2 (publish) and the optimizer wastes API calls on invisible products.
- **The inch-mark `"` breaks JSON parsing in Claude responses.** In the optimizer system prompt, explicitly require `in.` or `inch`.
- **`set_type_and_tags.py` should ONLY set `productType` when currently empty** — preserves manual overrides for special products like custom CPQ shower doors.
- **Re-run safety:** every script needs an idempotency check. Title-rewrite optimizer marks `<!-- brand-optimized-v1 -->`; weight-setter checks `weight.value == 0`; barcode-assigner persists a CSV log of assignments.
- **Weight regex must order shower-door BEFORE mirror.** Some products have "mirror" in title but are doors-with-mirror. Order matters.

## Real numbers

KZG ad-readiness pass results (2026-05-05):
- 175 products total
- 172/175 = **98.3% pass full audit**
- 100% have title, description, vendor, productType, status=ACTIVE, publishedAt
- 100% have featuredMedia, mediaCount ≥ 1
- 100% have SEO title + description ≥ 50 chars
- 100% have variant SKU + barcode + price + weight + tracking
- 100% have mm-google-shopping (4 metafields each)
- 98.3% have namespaced filter tags by category requirement
- 3 stragglers: descriptions rewritten by Claude dropped the size — manual fix via metafield

## Reference

- KZG sales-channel publication IDs:
  - Online Store: `gid://shopify/Publication/190948016265`
  - Shop: `gid://shopify/Publication/190948081801`
  - Google & YouTube: `gid://shopify/Publication/190957715593`
  - Facebook & Instagram: `gid://shopify/Publication/190957813897`
  - Pinterest: `gid://shopify/Publication/192331612297`
- Optimizer config: `claude-haiku-4-5` model, ~$0.25/100 products
- All scripts at `~/bin/kzg-blog/` (NOT `/tmp/` which clears on reboot)

## Anti-pattern

Don't run `optimize.py` first. You'll waste Claude API calls on products that haven't been published to channels yet (and you'll have to re-optimize once they are). Order the pipeline: classify → publish → weights → optimize → ...
