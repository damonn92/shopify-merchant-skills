---
name: shopify-store-operations
description: Authing the Shopify CLI to operate a store, expanding OAuth scopes mid-project, and avoiding the permanent-vs-custom-domain trap.
when_to_use:
  - Setting up a new client store for direct GraphQL operation
  - Adding a scope you forgot to grant at first auth (e.g. read_publications)
  - "shopify store" CLI throws "invalid_request" on the custom domain
not_for:
  - Building public Shopify Apps (use `shopify app` CLI + the official toolkit)
  - One-off GraphQL queries via curl (use Admin Token directly, see `shopify-daily-blog-automation`)
---

## Why

Every new client store eventually hits two gotchas:

1. **The CLI rejects the custom domain.** `shopify store auth --store calivell.myshopify.com` will fail because Shopify's OAuth callback only accepts the **permanent** `xkemzr-me.myshopify.com` form. The custom domain is for shoppers, not admins.

2. **Scopes are sticky.** If you authed with `read_products,write_products` and three weeks later need `write_files`, you must re-auth and explicitly include the new scope. The CLI will not "auto-upgrade".

This skill is the muscle memory of "what do I run, in what order, when."

## How

### First-time auth

```bash
# Always use the permanent domain. Find it in Shopify admin → Settings → Domains → Subdomains.
shopify store auth \
  --store <your-permanent-domain>.myshopify.com \
  --scopes read_products,write_products,read_inventory,write_inventory,read_orders,read_customers,read_content,write_content
```

### Common scope bundles by use case

| Use case | Scopes |
|---|---|
| Catalog + content + theme work (recommended starter) | `read_products,write_products,read_orders,write_orders,read_customers,write_customers,read_inventory,write_inventory,read_locations,read_content,write_content,read_themes,write_themes,read_price_rules,write_price_rules,read_discounts,write_discounts` |
| Add channel publishing + Files API | append `read_publications,write_publications,read_files,write_files` |
| Add navigation menus | append `read_online_store_navigation,write_online_store_navigation` |
| Add metafield definitions (for CPQ, custom data) | append `read_metaobjects,write_metaobjects,read_metaobject_definitions,write_metaobject_definitions` |

When in doubt, **request more not fewer** — Shopify won't care, and you avoid the back-and-forth.

### Mid-project scope expansion

Same `auth` command, with the **full** scope list (existing + new):

```bash
shopify store auth --store <your-permanent-domain>.myshopify.com \
  --scopes <existing,scopes,plus,new,one>
```

The CLI overwrites the previous grant; missing scopes from the previous list will be **revoked**.

### Running queries

```bash
# Read-only
shopify store execute --store <your-permanent-domain>.myshopify.com --query 'query { shop { name primaryDomain { url } } }'

# Mutation — must add --allow-mutations
shopify store execute --store <your-permanent-domain>.myshopify.com --allow-mutations --query 'mutation { ... }'
```

### Validate before mutating

If you're using the official `shopify-admin` skill alongside, run its `validate.mjs` first:

```bash
node ~/.claude/plugins/marketplaces/shopify-ai-toolkit/skills/shopify-admin/scripts/validate.mjs \
  --code "$(cat my-mutation.graphql)" \
  --model claude-sonnet-4-6
```

Don't push un-validated mutations against a live store. A bad `productUpdate` with a typo can wipe descriptions across hundreds of products.

## Gotchas

- **CLI mixes stdout/stderr in non-TTY mode.** Spinner blocks + ANSI cursor codes + box-drawing characters (`│╭╮╰╯─`) land on stderr; JSON sometimes lands on stdout, sometimes gets eaten. Robust parser:
  ```python
  import re
  combined = stdout + stderr
  cleaned = re.sub(r'\x1b\[[0-9;]*[A-Za-z]', '', combined)  # strip ANSI
  cleaned = re.sub(r'[│╭╮╰╯─⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏]', '', cleaned)  # strip box/spinner
  json_start = next(i for i, line in enumerate(cleaned.splitlines()) if line.lstrip().startswith('{'))
  ```

- **The CLI does not lock state across processes.** If two scripts run `shopify store execute` simultaneously, one may get an auth error. Stagger any cron-driven scripts on different minutes (we run KZG blog at 9:15 to avoid Calivell's 9:00 fight).

- **The `--store` flag is order-sensitive.** Always put it BEFORE `execute`/`auth`. After breaks parsing.

- **Custom apps from Shopify Dev Dashboard bypass the CLI** entirely. If you need a permanent token (not OAuth-tied to your machine), install a custom app with the scopes you need and use the resulting `shpat_*` token via curl. See `shopify-daily-blog-automation` for the recipe.

## Reference

- Official CLI install: `npm i -g @shopify/cli`
- Permanent domain lookup: `Settings → Domains → "your-store.myshopify.com"` (the one you can't change)
- `shopify-admin-execution` skill (in shopify-ai-toolkit): the wrapper that calls `shopify store execute` from inside Claude

## Anti-pattern

Don't store the `shpat_` token in shell history or `.bashrc`. Put it in a `chmod 600 config.env` file and `source` it from scripts.
