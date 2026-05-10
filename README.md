# Shopify Merchant Skills

> 11 battle-tested operational skills for Shopify merchants — distilled from running multi-SKU stores in production.
>
> Each skill captures a real production problem and the playbook that solved it. The official Shopify dev toolkit covers _APIs_; these cover _operations_ — the parts nobody writes down.

## Why these skills exist

Shopify's official docs and SDK examples are great for "how do I write this GraphQL mutation?" They're terrible for the questions you actually have at 11 PM on a Friday:

- "I pushed a theme file 25 minutes ago and the live site STILL shows the old content. WTF?"
- "Why does my checkout fail with 'Items don't meet weight requirements'?"
- "Why does my dropshipper's batch import only land 1 of 50 products?"
- "How do I do per-spec custom pricing if I'm not on Shopify Plus?"

Every skill in this repo answers one of those questions, with the gotcha that cost the original author 30+ minutes of debugging called out explicitly.

## Skills index

| Skill | Solves |
|---|---|
| [`shopify-store-operations`](skills/shopify-store-operations/SKILL.md) | Authing the CLI, expanding scopes mid-project, the permanent-vs-custom-domain trap |
| [`shopify-cache-debugging`](skills/shopify-cache-debugging/SKILL.md) | Page cache serves stale HTML for 30+ minutes after a theme push — verify via curl + GraphQL, not the browser |
| [`shopify-giga-dropshipping`](skills/shopify-giga-dropshipping/SKILL.md) | GigaCloud B2B importing via direct API (10× faster than UI clicks); shop-pinning to avoid cross-store pollution |
| [`shopify-listing-optimization`](skills/shopify-listing-optimization/SKILL.md) | The 9-script pipeline: classify → publish → weights → AI rewrite → barcodes → tags → media reorder → audit → fix |
| [`shopify-cpq-via-worker`](skills/shopify-cpq-via-worker/SKILL.md) | Configure-Price-Quote on a non-Plus store using on-demand discount codes from a Cloudflare Worker |
| [`shopify-design-wizard`](skills/shopify-design-wizard/SKILL.md) | Multi-step product configurator wizard with URL-restorable state |
| [`shopify-cart-attributes-form`](skills/shopify-cart-attributes-form/SKILL.md) | Replace 3rd-party booking/install apps with native `attributes[…]` cart fields |
| [`shopify-daily-blog-automation`](skills/shopify-daily-blog-automation/SKILL.md) | macOS launchd + Anthropic API → unique daily article with internal links, ~$0.05/post |
| [`shopify-seo-baseline`](skills/shopify-seo-baseline/SKILL.md) | Pre-launch SEO pass: collection prose, JSON-LD bundles, GSC Domain-Connect verification, sitemap audit |
| [`shopify-ai-product-images`](skills/shopify-ai-product-images/SKILL.md) | Vertex AI Imagen for lifestyle scenes + brightness-corner classifier to drop white-bg shots |
| [`shopify-theme-deployment`](skills/shopify-theme-deployment/SKILL.md) | Whisper/Horizon-aware deployment: query MAIN before push, fetch live JSON before edit, mega-menu reserved-handle trap |

## How these compare to Shopify's official toolkit

| Need | Use this | Use Shopify's [`shopify-ai-toolkit`](https://github.com/Shopify/shopify-ai-toolkit) |
|---|---|---|
| Generate a valid GraphQL mutation | | ✅ |
| Validate Liquid syntax | | ✅ |
| Build Polaris admin extensions | | ✅ |
| Why my theme push didn't stick | ✅ | |
| 9-script pipeline for dropshipped catalog cleanup | ✅ | |
| CPQ on Basic plan without Functions | ✅ | |
| Replace Zapiet with 30 lines of Liquid | ✅ | |
| Daily blog auto-publisher | ✅ | |

They're complementary. Use both.

## Install

### As a Claude Code plugin

```bash
# Inside Claude Code
/plugin install <your-github>/shopify-merchant-skills
```

### As a manual clone (works for any Claude-compatible client)

```bash
git clone https://github.com/<your-github>/shopify-merchant-skills.git ~/.claude/plugins/local/shopify-merchant-skills
```

The skills don't conflict with `shopify-ai-toolkit` — they live under a different namespace.

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
The list of footguns that cost the author 30+ minutes each.

## Reference
Source files, GraphQL mutations, env vars, real numbers.

## Anti-pattern
What NOT to do (and why people are tempted).
```

## Real numbers (anonymized)

To prove these aren't theoretical:

- **Listing optimization pipeline** ran on a 175-SKU bath fixtures store → **172/175 = 98.3%** pass full Google Shopping + Meta Catalog ad audit
- **Daily blog automation** has been firing every morning at 9 AM for 3+ months on two stores, ~$0.05/post via Claude Haiku
- **Cache debugging skill** caught a 30+ minute page_cache stickiness issue that 4 hours of dashboard clicking couldn't diagnose
- **CPQ via Worker** runs 8 live custom-product variants on a Basic plan; Plus would have been $2,300/month
- **AI product images** generated 254 lifestyle scenes on a furniture store for ~$5 total Vertex AI cost
- **Cart attributes form** replaced a $50/mo 3rd-party booking app with 30 lines of Liquid

## Contributing

PRs welcome. New skills must:

1. Be born from an actual production problem (no "what if" speculation)
2. Include the gotcha that cost you ≥ 30 minutes of debugging
3. Not duplicate Shopify's official docs — link to them and add the operational lessons docs miss

## License

MIT — use freely. Attribution appreciated but not required.
