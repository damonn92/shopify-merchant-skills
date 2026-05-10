---
name: shopify-seo-baseline
description: The pre-launch SEO baseline checklist that took Calivell from "indexed by accident" to ranking — collection prose, JSON-LD bundle, sitemap audit, GSC verification via Domain Connect, theme snippet pattern.
when_to_use:
  - You're launching a new Shopify store and need to do SEO right ONCE
  - The site has 100+ SKUs and risk of duplicate-content penalty
  - You're prepping for paid Google ads + need verified Search Console
not_for:
  - Single-SKU brand sites (overkill — manual SEO is fine)
  - Multi-language storefronts (need separate SEO pass per locale)
---

## Why

A new Shopify store has **factory-default SEO**: empty `seo.title`, generic meta descriptions, broken `robots.txt` redirects from 3rd-party app drops, and 0 structured data. Google indexes the homepage and 1-2 collections, then loses interest.

This skill is the **one-time pre-launch pass** that gets the site to the level where ongoing daily content (see [`shopify-daily-blog-automation`](../shopify-daily-blog-automation/SKILL.md)) and ad campaigns can actually compound. Built from doing it for Calivell (206 SKUs, 6 collections, 3 pages) end-to-end.

## How

The 6-wave baseline:

### Wave 1 — Collection-level SEO

Every meaningful collection (anything > 5 products) needs:

- Hand-tuned `seo.title` (≤60 chars, includes primary keyword)
- Hand-tuned `seo.description` (~145-200 chars, ends with shipping promise like "Free US shipping on orders $120+")
- 200-word `descriptionHtml` with `<h2>` subheads, spec bullets, 2-3 internal links to related collections
- Tile image (1200×1200, branded type-only or lifestyle)

Generate via Claude (cheap batch):

```python
system = '''Write Shopify collection SEO + description for {brand}.
Output JSON: {"seo_title", "seo_description", "description_html"}.
- seo_title ≤ 60 chars, lead with category keyword
- seo_description 145-200 chars, end with "Free US shipping on orders over $X."
- description_html: 2 paragraphs (~150 words), 1 <h2> subhead, 4-5 spec bullets, 2-3 <a href="/collections/X"> internal links to RELATED collections
- Forbid: invented certifications, specs, awards. Only use what's verifiable.
- Use "in." or "inch" not " (inch mark breaks JSON)
'''
```

Apply via `collectionUpdate` mutation with `seo: {title, description}` and `descriptionHtml`.

### Wave 2 — Product-level SEO

For every product:

- Unique SEO title ≤ 60 chars
- Unique meta description ~145-200 chars (no template repeats)
- 4-7 lowercase-hyphenated tags (drives related-products + search)
- Clean image alt texts (strip `- Main View` / `- Angle View` / `- Enhanced` cruft)

If you're starting from a templated import (dropshipping), see [`shopify-listing-optimization`](../shopify-listing-optimization/SKILL.md) for the full Claude rewrite pipeline.

Cleanup helpers:
```graphql
# Strip "- Main View" suffixes
mutation ($id: ID!, $alt: String!) {
  productUpdateMedia(productId: $id, media: [{ id: "...", alt: $alt }]) { mediaUserErrors { field message } }
}
```

### Wave 3 — Page-level SEO

Pages (`/pages/about`, `/pages/contact`, `/pages/faq`):

- Add `title_tag` and `description_tag` metafields (namespace=`global`)
- Rewrite content to be > 200 words with internal links
- For FAQ: wrap Q&A pairs in `<details>` + `<summary>` for native expand/collapse
- Include valid `FAQPage` JSON-LD `<script>` block at end of body for Google rich results

```graphql
mutation {
  metafieldsSet(metafields: [
    { ownerId: "gid://shopify/Page/<id>", namespace: "global",
      key: "title_tag", type: "single_line_text_field",
      value: "FAQ — Shipping, Returns, Assembly | <Brand>" },
    { ownerId: "gid://shopify/Page/<id>", namespace: "global",
      key: "description_tag", type: "single_line_text_field",
      value: "30 questions answered: shipping times, return policy, mattress sizing..." }
  ]) { userErrors { field message } }
}
```

### Wave 4 — Theme-level structured data snippet

Create `snippets/<brand>-jsonld.liquid`, render from `layout/theme.liquid` just before `{{ content_for_header }}`:

```liquid
{%- comment -%} snippets/calivell-jsonld.liquid {%- endcomment -%}

{%- comment -%} Organization (all pages) {%- endcomment -%}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "{{ shop.name }}",
  "url": "{{ shop.url }}",
  "logo": "{{ shop.url }}{{ 'logo.png' | asset_url }}",
  "sameAs": ["https://www.instagram.com/...", "https://www.facebook.com/..."]
}
</script>

{%- comment -%} Product + BreadcrumbList (PDP only) {%- endcomment -%}
{%- if request.page_type == 'product' and product -%}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "{{ product.title | escape }}",
  "image": [{% for img in product.images limit: 5 %}"{{ img | image_url: width: 1200 }}"{% unless forloop.last %},{% endunless %}{% endfor %}],
  "description": "{{ product.description | strip_html | truncate: 500 | escape }}",
  "sku": "{{ product.selected_or_first_available_variant.sku }}",
  "brand": { "@type": "Brand", "name": "{{ product.vendor | escape }}" },
  "offers": {
    "@type": "Offer",
    "url": "{{ shop.url }}{{ product.url }}",
    "priceCurrency": "{{ cart.currency.iso_code }}",
    "price": "{{ product.selected_or_first_available_variant.price | money_without_currency | strip_html }}",
    "availability": "{% if product.available %}https://schema.org/InStock{% else %}https://schema.org/OutOfStock{% endif %}"
  }
}
</script>
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Home", "item": "{{ shop.url }}" },
    {%- if collection -%}{ "@type": "ListItem", "position": 2, "name": "{{ collection.title }}", "item": "{{ shop.url }}{{ collection.url }}" },{%- endif -%}
    { "@type": "ListItem", "position": 3, "name": "{{ product.title }}", "item": "{{ shop.url }}{{ product.url }}" }
  ]
}
</script>
{%- endif -%}

{%- comment -%} CollectionPage + BreadcrumbList (collection pages) {%- endcomment -%}
{%- if request.page_type == 'collection' and collection -%}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "CollectionPage",
  "name": "{{ collection.title | escape }}",
  "description": "{{ collection.description | strip_html | truncate: 500 | escape }}",
  "url": "{{ shop.url }}{{ collection.url }}"
}
</script>
{%- endif -%}

{%- comment -%} WebSite + SearchAction (homepage — enables Google sitelinks search box) {%- endcomment -%}
{%- if request.page_type == 'index' -%}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "url": "{{ shop.url }}",
  "potentialAction": {
    "@type": "SearchAction",
    "target": "{{ shop.url }}/search?q={search_term_string}",
    "query-input": "required name=search_term_string"
  }
}
</script>
{%- endif -%}

{%- comment -%} Brand visual + crawler hints {%- endcomment -%}
<meta name="theme-color" content="#1B2D5B">
<link rel="apple-touch-icon" href="{{ 'apple-touch-icon.png' | asset_url }}">
<meta name="msapplication-TileColor" content="#1B2D5B">
{%- if request.page_type == 'cart' or request.page_type == 'customers/account' -%}
<meta name="robots" content="noindex, nofollow">
{%- else -%}
<meta name="robots" content="index, follow, max-image-preview:large, max-snippet:-1">
<meta name="googlebot" content="index, follow">
{%- endif -%}

{%- comment -%} Verification tags via shop metafields (set when GSC issues codes) {%- endcomment -%}
{%- if shop.metafields.seo.google_site_verification -%}
<meta name="google-site-verification" content="{{ shop.metafields.seo.google_site_verification }}">
{%- endif -%}
{%- if shop.metafields.seo.bing_verification -%}
<meta name="msvalidate.01" content="{{ shop.metafields.seo.bing_verification }}">
{%- endif -%}
```

### Wave 5 — Performance hints

Same snippet, top section:

```liquid
{%- comment -%} Preconnect to Shopify CDN + monorail (analytics) {%- endcomment -%}
<link rel="preconnect" href="https://cdn.shopify.com">
<link rel="preconnect" href="https://fonts.shopifycdn.com">
<link rel="preconnect" href="https://monorail-edge.shopifysvc.com">

{%- comment -%} Homepage LCP preload {%- endcomment -%}
{%- if request.page_type == 'index' and section.settings.hero_image -%}
<link rel="preload" as="image" href="{{ section.settings.hero_image | image_url: width: 1920 }}" fetchpriority="high">
{%- endif -%}
```

### Wave 6 — GSC + sitemap submission

**Best path: Domain property via Domain Connect.**

If your domain is on GoDaddy / Squarespace / Cloudflare with auto-DNS-integration support:

1. Go to [Search Console](https://search.google.com/search-console).
2. Add **Domain property** (not URL property).
3. When asked to verify, click "**Verify by DNS provider**" — Google routes through GoDaddy's Domain Connect API.
4. Approve the prompt → DNS TXT record auto-added → property verified in seconds.

If your registrar doesn't support Domain Connect:

- HTML meta-tag method: set `shop.metafields.seo.google_site_verification` to the code from GSC; the snippet above renders the meta tag.
- Or: DNS TXT record manually.

Then submit the sitemap (Shopify maintains it for free):

```
https://yourstore.com/sitemap.xml
```

Verify post-submit:
```bash
curl -s https://yourstore.com/sitemap.xml | grep -c '<loc>'
# expect = total products + collections + pages + articles
```

## Gotchas

- **Homepage SEO won't update via Liquid alone.** Whisper/Horizon themes render homepage with `page_title = shop.name` directly. Setting `shop.metafields.global.title_tag` doesn't override the rendered title. **Solution:** user must set homepage title + meta description manually via Shopify admin → Online Store → Preferences → "Title and meta description". Stop chasing the Liquid approach.

- **Don't auto-generate `seo.description` with a templated suffix.** Google penalizes 40+ products having identical "Free US shipping over $X" tail. **Vary the suffix** between SKUs — sometimes "Free shipping over $X," sometimes "Ships in 2-3 days," etc.

- **3rd-party SEO apps leave junk meta tags in `content_for_header`.** Symptoms: legacy verification tags, OG tags from a removed app. Don't try to remove them via Liquid (you can't always reach into `content_for_header`); just ignore. Domain Connect verification supersedes any HTML meta tag.

- **`?width=` Shopify CDN parameter only works on `image_url`, not raw URLs.** Use Liquid `{{ image | image_url: width: 1200 }}`, not `<img src="{{ image }}?width=1200">`.

- **`shop.metafields.global.title_tag` exists ONLY if you wrote it.** Default response is empty, not an error. Snippet handles both.

- **DNS-verified domain property ≠ URL-prefix property.** They're separate in GSC. Submit the sitemap to whichever you actually verified. Domain property covers all subdomains too (handy if you add `blog.calivell.com` later).

## Reference

- **Calivell baseline result (2026-04-17):**
  - 206 products: 100% have unique SEO title (≤60 chars), unique meta description (~145-200 chars), 4-7 tags
  - 43 templated descriptions rewritten via Claude
  - 140 awkward image alt texts cleaned
  - 6 collections: 200-word descriptionHtml + 2-3 internal links each
  - 3 pages (FAQ, About, Contact) rewritten + JSON-LD FAQPage schema
  - GSC: domain property `calivell.com` verified via GoDaddy Domain Connect
  - Sitemap: `https://calivell.com/sitemap.xml` submitted, 0 indexing errors

- **KZG follow-up:** snippet `kzg-head-addons.liquid` with same pattern, including FAQ JSON-LD for the 7 FAQ Q&A on `/pages/faq` (produces Google rich FAQ snippet).

## Anti-pattern

Don't install a Shopify SEO app for this. Yoast/SEO Manager/Avada all eat 2-3 KB of bloat per page and inject conflicting meta tags. The 200-line snippet above does 95% of what they do, with no monthly fee and no blocking JS. Reserve apps for things you genuinely can't do in Liquid (e.g. dynamic structured data based on customer segment).
