# KNA Shopify Skills

> 11 battle-tested skills distilled from running **Calivell**, **KZG (thekzg.com)**, and **AISight** in production for cross-border e-commerce.
>
> Maintained by [KNA](https://wearekna.com) — independent Claude API gateway for mainland China developers.

These aren't tutorials regurgitated from Shopify docs. Every skill here came out of an actual production breakage, performance issue, or "how the hell do we do this without paying $X/mo for a 3rd-party app" decision. The official Shopify dev toolkit covers _APIs_; KNA Shopify Skills cover _operations_ — the parts nobody writes down.

## Skills index

| Skill | What it solves | Real client |
|---|---|---|
| [`shopify-store-operations`](skills/shopify-store-operations/SKILL.md) | Authing the CLI, OAuth scope expansion, the permanent-vs-custom-domain trap | Calivell, KZG |
| [`shopify-cache-debugging`](skills/shopify-cache-debugging/SKILL.md) | Page cache serves stale HTML for 30+ min after theme push — verify via curl + GraphQL, not browser | KZG (Whisper theme) |
| [`shopify-giga-dropshipping`](skills/shopify-giga-dropshipping/SKILL.md) | GigaCloud B2B importing via direct API (10× faster than Element Plus button clicks); shop_id pinning to avoid cross-store pollution | Calivell, KZG |
| [`shopify-listing-optimization`](skills/shopify-listing-optimization/SKILL.md) | The 9-script pipeline: postprocess → publish → weights → optimize → barcodes → tags → media reorder → audit → fix | KZG (175 SKUs) |
| [`shopify-cpq-via-worker`](skills/shopify-cpq-via-worker/SKILL.md) | Configure-Price-Quote on a non-Plus store using on-demand discount codes from a Cloudflare Worker — Shopify Functions not required | KZG (8 custom variants live) |
| [`shopify-design-wizard`](skills/shopify-design-wizard/SKILL.md) | Multi-step product wizard with URL-restorable state + cart-line label rewrite — DreamLine-style | KZG Design Center |
| [`shopify-cart-attributes-form`](skills/shopify-cart-attributes-form/SKILL.md) | Replace Zapiet-style booking/install apps with native `attributes[…]` cart fields, no $$/month subscription | KZG SoCal local delivery |
| [`shopify-daily-blog-automation`](skills/shopify-daily-blog-automation/SKILL.md) | macOS launchd + Anthropic API direct → unique daily article with internal links, 0 manual touch | Calivell, KZG |
| [`shopify-seo-baseline`](skills/shopify-seo-baseline/SKILL.md) | Full pre-launch SEO pass: collection prose, JSON-LD bundles, GSC verification via Domain Connect, sitemap audit | Calivell (206 SKUs) |
| [`shopify-ai-product-images`](skills/shopify-ai-product-images/SKILL.md) | Vertex AI Imagen for lifestyle scenes + brightness-corner classifier to drop white-bg studio shots | Calivell (574 cleaned, 254 generated) |
| [`shopify-theme-deployment`](skills/shopify-theme-deployment/SKILL.md) | Whisper/Horizon-aware deployment: query MAIN role before push, fetch live JSON before edit, mega-menu reserved-handle gotcha | KZG (theme switched 3× in 2 weeks) |

## Why these skills exist

| Problem | Where the docs fail you | What this skill gives you |
|---|---|---|
| You push theme JSON, the site shows old content for 30+ minutes | Docs don't mention page_cache stickiness | curl + cookie-jar verification recipe |
| GIGA's batch import button silently drops 49 of 50 products | GIGA's docs say batch is supported | Direct API loop, one-product-at-a-time |
| Your $9,999 placeholder products need real per-spec pricing on a Basic plan | Shopify Functions require Plus | Worker issues 45-min single-use discount code |
| Daily blog posts must be unique and SEO-linked | No first-party automation | launchd + Anthropic API recipe with internal-link prompt |
| Local-delivery scheduling needs date/time/address fields | Zapiet etc. cost $50–200/mo | 12 lines of Liquid using `attributes[…]` |
| You have 2,000 products and 1,900 white-bg shots crowd out lifestyle | Background-removal apps cost $$ | corner-brightness classifier in 30 lines of Python |

## Install

```bash
# Clone alongside your other Claude skills
git clone https://github.com/damonn92/kna-shopify-skills.git ~/.claude/plugins/local/kna-shopify-skills
```

Or as a Claude Code plugin (works alongside Shopify's official `shopify-ai-toolkit`):

```bash
# Inside Claude Code
/plugin install damonn92/kna-shopify-skills
```

The skills don't conflict with `shopify-ai-toolkit` — they're complementary. Use the official ones for **API code generation**, these for **operational playbooks**.

## Skill anatomy

Every `SKILL.md` follows this shape:

```yaml
---
name: skill-name
description: One-sentence summary
when_to_use: Bullet-list triggers
not_for: Anti-cases
---

## Why
The 3-paragraph backstory of why this pattern exists.

## How
Numbered procedure + code recipes.

## Gotchas
The list of footguns.

## Reference
Source files, GraphQL mutations, env vars.
```

## Contributing

PRs welcome. New skills must:
1. Be born from an actual production problem (no "what if" speculation)
2. Include the gotcha that cost you ≥ 30 minutes of debugging
3. Not duplicate Shopify's official docs

## License

MIT — use freely, attribution appreciated. The `references/` directory contains stripped-down memory snapshots that informed each skill (with client-specific IDs replaced by `<placeholders>`).

---

🥃 Built by [@damonn92](https://github.com/damonn92) at KNA. If these saved you a Sunday afternoon, tell a friend in `r/shopify` (or `WeChat`).
