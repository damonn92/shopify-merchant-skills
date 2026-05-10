---
name: shopify-cache-debugging
description: When Shopify's page_cache serves stale rendered HTML for 30+ minutes after a theme push — diagnose via curl + GraphQL (not the browser), and emergency-bust via inline CSS or template version bumps.
when_to_use:
  - "You pushed a theme file but the live site still shows old content"
  - "Browser shows old version, incognito shows new — and you've already cleared cache"
  - "Section settings update via GraphQL but the rendered section still has old IDs"
  - Asset CDN returns the right `?v=hash` but pages reference an OLDER bundle generation (`/cdn/shop/t/N-2/...`)
not_for:
  - Real Shopify CDN purge (you can't trigger it; only Shopify can)
  - Storefront API caching (different layer entirely)
---

## Why

Shopify caches the **rendered HTML** of a page _per-template-version_. Push a `themeFilesUpsert`, and:

- A new template version is created on the backend.
- The CDN gets the new asset bundle path (`/cdn/shop/t/N+1/...`).
- BUT old template versions stay cached — and visitors whose `_shopify_essential` cookie pins them to an older session may continue seeing the old render for **5 to 30+ minutes**.

The smoking gun: the response header `etag: page_cache:STORE:Controller:HASH` changes per request (suggesting Shopify regenerated the cache), but the rendered output still shows an old section ID.

This skill is the playbook for diagnosing and force-busting.

## How

### Step 1 — Verify the push actually landed

Don't trust the browser. Two independent verification paths:

**(a) GraphQL `theme.files`:**
```graphql
query {
  theme(id: "gid://shopify/OnlineStoreTheme/<id>") {
    files(filenames: ["sections/your-section.liquid"], first: 1) {
      nodes {
        body {
          ... on OnlineStoreThemeFileBodyText { content }
        }
        contentType
        size
      }
    }
  }
}
```

Compare `content` to your local file. If they match → push landed.

**(b) Asset CDN with explicit version:**

Read the rendered HTML once:
```bash
curl -s -c /tmp/jar.txt https://yourstore.com/ | grep -oE 'cdn/shop/t/[0-9]+/assets/[a-zA-Z0-9_-]+\.css\?v=[a-f0-9]+' | head -3
# Output e.g.: cdn/shop/t/5/assets/theme.css?v=abc123
```

Then fetch that exact URL:
```bash
curl -s "https://yourstore.com/cdn/shop/t/5/assets/theme.css?v=abc123" | grep -c "your-new-rule"
```

If GraphQL says new content + the asset URL serves new content but the page still renders old → **page_cache stale**.

### Step 2 — Force-bust the page cache

Three escalating tactics, easiest first:

**(2a) Bump a JSON template setting.** Section content cache is keyed off `templates/index.json` version, so:

```bash
shopify store execute --store <permanent>.myshopify.com --allow-mutations --query "
mutation {
  themeFilesUpsert(themeId: \"gid://shopify/OnlineStoreTheme/<id>\", files: [{
    filename: \"templates/index.json\",
    body: { type: TEXT, value: \$json }
  }]) { upsertedThemeFiles { filename } userErrors { field message } }
}" --variables "json:$(jq '.sections.foo.settings.alt = \"v$(date +%s)\"' templates/index.json)"
```

This bumps the JSON version → new page_cache key → fresh render.

**(2b) Rename the section block key.** `kzg_design_center_promo` → `kzg_dc_promo_v2` (then `_v3` if needed). Sometimes works. Sometimes doesn't — caches can pin the OLD section ID.

**(2c) Inline `<style>` as emergency override.** If you must push a visual fix NOW and asset CDN cache is jammed:

```liquid
{% comment %} sections/your-section.liquid {% endcomment %}
<style>
  .your-class { color: #1E3A5F; }  /* bypasses asset compilation */
</style>
```

Inline `<style>` blocks bypass the asset pipeline entirely. Ugly, but it ships.

### Step 3 — Verify with a fresh cookie jar

The `_shopify_essential` cookie pins your session to a cache version. Any retry must use a fresh jar:

```bash
rm /tmp/jar.txt
curl -s -c /tmp/jar.txt -b /tmp/jar.txt https://yourstore.com/?cb=$(date +%s) | grep "your-new-content"
```

If still stale → wait 5-15 min, retry. If still stale after 30 min → escalate Shopify support.

## Gotchas

- **Browser dev tools "Disable cache" doesn't help.** The cache is on Shopify's edge, not your browser.
- **Incognito sometimes shows new content.** That's because incognito has no `_shopify_essential` cookie → fresh session → fresh cache key. Useful as a smoke test, but don't celebrate too early — your existing visitors still have their cookies.
- **Asset CDN bundle generations are NOT in lockstep with template versions.** `t/5` might be the bundle path while pages reference `t/7`. Always check the `cdn/shop/t/N` value in the rendered HTML, not the latest theme version.
- **`themeFilesUpsert` returns success even when the file content didn't change.** If you pass identical body, the mutation still returns `upsertedThemeFiles` but skips backend invalidation. Add a trailing comment with `<!-- v$(date +%s) -->` to force change.
- **Cache TTL is undocumented.** Empirically: 5 min minimum, 30+ min observed. Plan customer-facing announcements with at least 1 hour buffer after final push.

## Reference

- KZG live theme during this issue: `Whisper Optimized — Mega Menu v2`, `gid://shopify/OnlineStoreTheme/153266028681`
- Old bundle path observed: `cdn/shop/t/7/...` while the new bundle was `cdn/shop/t/5/...`
- `etag` header pattern: `page_cache:<STORE_ID>:<Controller>:<HASH>` — the hash changes but content doesn't, that's the giveaway

## Anti-pattern

Don't keep re-pushing the same theme file hoping it'll "stick." Each `themeFilesUpsert` of identical content is a no-op. Bump a JSON template setting OR change a comment in the file.
