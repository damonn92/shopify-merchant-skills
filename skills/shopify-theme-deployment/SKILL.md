---
name: shopify-theme-deployment
description: Deploying theme files via `themeFilesUpsert` without overwriting concurrent edits, with Whisper/Horizon-specific gotchas (`collection_list` schema, mega-menu reserved handle, mobile vertical_on_mobile).
when_to_use:
  - Pushing section/snippet changes to a live theme programmatically (CI, scripts, agents)
  - Two people are editing the theme: you in code, them in Theme Editor
  - You're targeting Whisper / Horizon and the docs are sparse on schema specifics
not_for:
  - Initial theme installation (use Shopify CLI `shopify theme push` once)
  - Apps that ship as theme app extensions (different mechanism)
---

## Why

`themeFilesUpsert` looks like a simple "save file" mutation. It's not — it's a **full-file replacement**, idempotent on identical content, and silently overwrites whatever's currently in `templates/index.json` if your local copy is stale.

Three specific traps we hit on KZG (theme switched 3 times in 2 weeks, Refresh → Whisper → Whisper Optimized → Whisper Optimized Mega Menu v2):

1. **Always query MAIN role before push.** Don't trust cached theme IDs.
2. **Always fetch the live JSON before edit.** Each `themeFilesUpsert` of `templates/index.json` is a complete replacement; building on a stale local copy silently reverts intermediate changes.
3. **Whisper's `collection_list` setting requires raw collection handles, not gid strings.** `["shower-doors", "toilets"]` works. `["gid://shopify/Collection/123"]` silently renders blank cards. `"shopify://collections/..."` URI silently renders blank cards.

## How

### Pattern 1 — Get the live MAIN theme

```graphql
query {
  themes(first: 5, roles: [MAIN]) {
    nodes { id name role }
  }
}
```

There's exactly one `MAIN` role at any time. Use that ID for pushes. Don't memoize — themes get switched.

### Pattern 2 — Fetch live JSON, edit, push back

```python
def edit_template_json(theme_id, filename, edit_fn):
    # Step 1: Fetch
    q = '''query ($id: ID!, $name: String!) {
      theme(id: $id) {
        files(filenames: [$name], first: 1) {
          nodes {
            body { ... on OnlineStoreThemeFileBodyText { content } }
          }
        }
      }
    }'''
    res = shopify_gql(q, {'id': theme_id, 'name': filename})
    content = res['data']['theme']['files']['nodes'][0]['body']['content']

    # Step 2: Parse + edit
    data = json.loads(content)
    edit_fn(data)
    new_content = json.dumps(data, indent=2)

    # Step 3: Push if changed (avoid no-op upserts)
    if new_content == content:
        print(f'  no change to {filename}, skipping')
        return

    m = '''mutation ($id: ID!, $files: [OnlineStoreThemeFilesUpsertFileInput!]!) {
      themeFilesUpsert(themeId: $id, files: $files) {
        upsertedThemeFiles { filename }
        userErrors { field message }
      }
    }'''
    res = shopify_gql(m, {
        'id': theme_id,
        'files': [{'filename': filename, 'body': {'type': 'TEXT', 'value': new_content}}],
    })
    print(f'  pushed {filename}: {res}')

# Usage
edit_template_json(theme_id, 'templates/index.json', lambda data: data['sections']['hero'].update({'settings': {'image': '...'}}))
```

### Pattern 3 — Whisper `collection_list` setting

```json
// templates/index.json — BLOCK type "collection_list"
{
  "sections": {
    "homepage_collections": {
      "type": "collection-list",
      "settings": {
        // ✅ Works
        "collection_list": ["shower-doors", "toilets", "mirrors", "vanities"]

        // ❌ Silently renders blank
        // "collection_list": ["gid://shopify/Collection/319533285513"]
        // "collection_list": ["shopify://collections/shower-doors"]
        // "collection_list": "shower-doors,toilets"
      },
      "blocks": {},
      "block_order": []
    }
  }
}
```

### Pattern 4 — Static blocks in collection-list / product-list presets

Whisper has presets with `static` blocks (e.g. `_collection-card-image`). You CAN'T declare these as regular blocks. To use them, **copy the full preset block from the section's schema's `presets` array**:

```liquid
{% comment %} sections/collection-list.liquid — already has presets defined in schema {% endcomment %}
{% schema %}
{
  "name": "collection-list",
  "presets": [{
    "name": "Collection list",
    "blocks": {
      "_collection-card-image": { "type": "_collection-card-image", "static": true },
      "_collection-card-info": { "type": "_collection-card-info", "static": true }
    }
  }]
}
{% endschema %}
```

Don't try to wedge `_collection-card-image` into your `templates/index.json` directly — it'll error.

### Pattern 5 — Mobile responsiveness (Horizon `vertical_on_mobile`)

Each `group`/section in Horizon has a `vertical_on_mobile` toggle. **Default is `false`** (side-by-side stays on mobile, often broken).

```json
{
  "type": "trust-bar",
  "settings": { "vertical_on_mobile": true },
  "blocks": { ... }
}
```

For 3-column trust bars and 3-column editorial sections on mobile, this is a must.

For collection-list, set `mobile_columns: "2"` for a 4-card grid.

## Gotchas

### Reserved handles
- `main-menu` is reserved for the system-default menu. Creating a menu with `handle: "main-menu"` auto-renames to `main-menu-1`. Either edit the default in admin or point your theme header section's `menu` setting to `main-menu-1`.

### Section name 25-char limit
- Schema's `name` field is capped at 25 characters. Don't write `"name": "Custom Hero Banner with Lifestyle Image"` — silently truncated. Use `Hero · Lifestyle` form.

### Asset CDN cache layers
- The CSS file at `?v=NEW_HASH` may serve old content for a few minutes after push. Page rendering may reference an OLDER bundle path (`/cdn/shop/t/N-2/...`) for some visitors. See [`shopify-cache-debugging`](../shopify-cache-debugging/SKILL.md) for verification recipe.

### `themeFilesUpsert` returns success on no-op
- If you push identical content (byte-for-byte), the mutation returns `upsertedThemeFiles` but the underlying timestamp doesn't change. To force-bump (e.g. for cache invalidation), append a comment with `<!-- v$(date +%s) -->` to the file body.

### Liquid `request.query_string` returns empty for cached templates
- Don't try to read URL params server-side on cached pages. Render all states + JS toggle. See [`shopify-design-wizard`](../shopify-design-wizard/SKILL.md).

### Theme version pinning
- After a Shopify theme update, schema can change. Run `shopify theme info --store <permanent>` to see the current Whisper version. Watch the changelog at `https://shopify.dev/changelog?filter=themes`.

### Don't `shopify theme push` after API edits
- The CLI `shopify theme push` syncs your LOCAL files to the live theme — overwriting any edits made via API or Theme Editor since you last pulled. Use `shopify theme pull` first if you've been editing remotely. Better yet: don't mix `shopify theme push` with API workflows.

## Reference

- KZG theme switches:
  - 2026-04-17: Refresh `gid://shopify/OnlineStoreTheme/152879497353` (initial)
  - 2026-04-18: Whisper `gid://shopify/OnlineStoreTheme/152879104137`
  - 2026-04-25: Whisper Optimized `gid://shopify/OnlineStoreTheme/153023119497`
  - 2026-05-02: Whisper Optimized — Mega Menu v2 `gid://shopify/OnlineStoreTheme/153266028681`
- Whisper available `icon` types in `blocks/icon.liquid` (59 total): apple, arrow, bag, box, chat_bubble, check_box, heart, lock, return, snapchat, star, truck — for trust-bar use `truck / lock / return` as the canonical trio
- Section schemas live at `config/settings_schema.json` and individual `sections/*.liquid` `{% schema %}` blocks

## Anti-pattern

Don't keep a "snapshot" of the theme files in your local repo and push from there. Each push is a full-file replacement; the snapshot will go stale the moment ANYONE edits the theme via Theme Editor. Always:

1. Query live MAIN theme ID
2. Fetch live file content
3. Edit in-memory
4. Push back

For longer projects, sync via `shopify theme pull` before each session AND warn collaborators not to use Theme Editor mid-session.
