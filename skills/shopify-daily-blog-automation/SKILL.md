---
name: shopify-daily-blog-automation
description: macOS launchd + Anthropic API direct + Shopify Admin GraphQL = unique daily blog post with internal links, no manual touch, ~$0.05/post.
when_to_use:
 - You want a Shopify store blog that updates daily for SEO without writing each post
 - You have collection pages that need internal-link traffic to rank
 - You want zero ongoing API cost from a 3rd-party content app
not_for:
 - Stores where every post must be hand-edited (just don't automate)
 - Multi-tenant SaaS (use a real backend, not launchd on a Mac)
 - Frequencies > 5 posts/day (Mac-must-be-awake constraint becomes painful)
---

## Why

The 3rd-party "AI blog" Shopify apps are $30-99/mo for what is, mechanically, a daily cron + Claude API call + GraphQL mutation. We built it ourselves for ~$0.05/post (Claude Haiku-4-5) and got better internal-linking control because we wrote the prompt.

Architecture: launchd LaunchAgent fires daily at 9 AM local → bash wrapper sources config → Python script fetches existing article titles + recent products → calls Claude for unique article draft → publishes via Shopify Admin GraphQL `articleCreate` mutation.

## How

### File layout (`~/bin/<store>-blog/`)

```
~/bin/<store>-blog/
├── config.env # chmod 600 — shop domain, blog GID, Anthropic key
├── publish.sh # bash wrapper sourcing config + invoking python
├── publish.py # the actual logic
├── logs/
│ └── <recent>.log # one log per day (`YYYY-MM-DD.log`), `.draft.json` saved alongside
└── (optional) weekly_audit.py # runs Mondays, regenerates SEO for products updated in last 7 days
```

### `config.env`

```bash
# chmod 600 — keep secrets out of shell history
export SHOPIFY_STORE_DOMAIN="<permanent>.myshopify.com"
export SHOPIFY_ADMIN_TOKEN="shpat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export SHOPIFY_BLOG_GID="gid://shopify/Blog/<your-blog-id>"
export ANTHROPIC_API_KEY="sk-ant-..."
export ANTHROPIC_MODEL="claude-haiku-4-5" # cheap+fast for blog drafts
export STORE_BRAND_NAME="the store"
export STORE_DOMAIN="yourstore.com"
```

### Custom App for permanent token (one-time setup)

Don't use `shopify store` CLI for crons — its auth is interactive-friendly only. Instead:

1. Shopify Dev Dashboard → Apps → Create custom app
2. Grant scopes: `write_content, read_products, write_products, read_themes, write_themes, write_online_store_pages`
3. Install on the target shop → copy the `shpat_*` permanent access token
4. Stash in `config.env` as `SHOPIFY_ADMIN_TOKEN`

Endpoint: `https://{shop}/admin/api/2026-04/graphql.json` with header `X-Shopify-Access-Token: shpat_*`.

### `publish.py` core logic

```python
#!/usr/bin/env python3
import os, json, urllib.request, urllib.error, datetime, re

STORE = os.environ['SHOPIFY_STORE_DOMAIN']
TOKEN = os.environ['SHOPIFY_ADMIN_TOKEN']
BLOG_GID = os.environ['SHOPIFY_BLOG_GID']
ANTHRO_KEY = os.environ['ANTHROPIC_API_KEY']
MODEL = os.environ.get('ANTHROPIC_MODEL', 'claude-haiku-4-5')
BRAND = os.environ['STORE_BRAND_NAME']

def shopify_gql(query, variables=None):
 body = json.dumps({'query': query, 'variables': variables or {}}).encode()
 req = urllib.request.Request(
 f'https://{STORE}/admin/api/2026-04/graphql.json',
 data=body, method='POST',
 headers={'Content-Type':'application/json', 'X-Shopify-Access-Token': TOKEN}
 )
 with urllib.request.urlopen(req) as r:
 return json.loads(r.read())

def fetch_recent_articles(blog_gid, n=20):
 q = '''query ($id: ID!, $n: Int!) {
 blog(id: $id) { articles(first: $n, sortKey: PUBLISHED_AT, reverse: true) {
 nodes { title handle publishedAt }
 }}
 }'''
 return shopify_gql(q, {'id': blog_gid, 'n': n})['data']['blog']['articles']['nodes']

def fetch_recent_products(n=12):
 q = '''query ($n: Int!) {
 products(first: $n, sortKey: UPDATED_AT, reverse: true) {
 nodes { handle title onlineStoreUrl }
 }
 }'''
 return shopify_gql(q, {'n': n})['data']['products']['nodes']

def claude_draft(existing_titles, recent_products):
 system = f'''You are a content writer for {BRAND}. Write a unique blog post that:
- Has a title NOT in this list of existing titles: {existing_titles}
- 700-900 words, original (don't rephrase existing titles)
- Includes 3-5 internal links to product or collection pages from this list:
 Products: {[(p["handle"], p["title"]) for p in recent_products]}
 Collections: bedroom, kids-furniture, outdoor, living-room, office-furniture, dining-room
- Internal link format: <a href="/products/{{handle}}">descriptive text</a> or <a href="/collections/{{handle}}">...</a>
- HTML with <h2> subheads, <p>, <ul>/<li>; no <html> or <body> tags
- SEO-aware: keyword in h1+first paragraph

Output strict JSON: {{"title": "...", "handle": "...", "body_html": "...", "summary_html": "...", "tags": ["tag1","tag2"]}}'''

 body = json.dumps({
 'model': MODEL,
 'max_tokens': 4000,
 'system': system,
 'messages': [{'role':'user','content':f'Today is {datetime.date.today().isoformat()}. Write today\'s post.'}]
 }).encode()

 req = urllib.request.Request(
 'https://api.anthropic.com/v1/messages', data=body, method='POST',
 headers={
 'Content-Type':'application/json',
 'x-api-key': ANTHRO_KEY,
 'anthropic-version':'2023-06-01',
 }
 )
 with urllib.request.urlopen(req) as r:
 text = json.loads(r.read())['content'][0]['text']
 # Sometimes Claude wraps in markdown code fence — strip it
 text = re.sub(r'^```(?:json)?\s*|\s*```$', '', text.strip(), flags=re.M)
 return json.loads(text)

def publish_article(draft):
 q = '''mutation ($input: ArticleCreateInput!) {
 articleCreate(article: $input) {
 article { id handle onlineStoreUrl publishedAt }
 userErrors { field message }
 }
 }'''
 inp = {
 'blogId': BLOG_GID,
 'title': draft['title'],
 'handle': draft['handle'],
 'body': draft['body_html'],
 'summary': draft.get('summary_html', ''),
 'tags': draft.get('tags', []),
 'author': {'name': BRAND}, # REQUIRED — docs imply optional, it's not
 'isPublished': True,
 }
 res = shopify_gql(q, {'input': inp})
 return res

def main():
 today = datetime.date.today().isoformat()
 log_path = os.path.expanduser(f'~/bin/{BRAND.lower()}-blog/logs/{today}.log')
 os.makedirs(os.path.dirname(log_path), exist_ok=True)

 with open(log_path, 'w') as log:
 try:
 existing = [a['title'] for a in fetch_recent_articles(BLOG_GID, 50)]
 products = fetch_recent_products(12)
 log.write(f'fetched {len(existing)} existing titles + {len(products)} recent products\n')

 draft = claude_draft(existing, products)
 with open(log_path + '.draft.json', 'w') as df: json.dump(draft, df, indent=2)
 log.write(f'drafted: {draft["title"]}\n')

 res = publish_article(draft)
 if res.get('data', {}).get('articleCreate', {}).get('article'):
 a = res['data']['articleCreate']['article']
 log.write(f'PUBLISHED: {a["onlineStoreUrl"]}\n')
 else:
 log.write(f'FAILED: {json.dumps(res, indent=2)}\n')
 except Exception as e:
 log.write(f'ERROR: {e}\n')
 raise

if __name__ == '__main__':
 main()
```

### LaunchAgent (`~/Library/LaunchAgents/com.{brand}.daily-blog.plist`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
 <key>Label</key>
 <string>com.<store>.daily-blog</string>
 <key>ProgramArguments</key>
 <array>
 <string>/bin/bash</string>
 <string>-c</string>
 <string>source /Users/damon/bin/<store>-blog/config.env && /usr/bin/python3 /Users/damon/bin/<store>-blog/publish.py</string>
 </array>
 <key>StartCalendarInterval</key>
 <dict>
 <key>Hour</key><integer>9</integer>
 <key>Minute</key><integer>0</integer>
 </dict>
 <key>StandardOutPath</key>
 <string>/Users/damon/bin/<store>-blog/logs/launchd.out.log</string>
 <key>StandardErrorPath</key>
 <string>/Users/damon/bin/<store>-blog/logs/launchd.err.log</string>
</dict>
</plist>
```

Load it:
```bash
launchctl load ~/Library/LaunchAgents/com.<store>.daily-blog.plist
launchctl list | grep daily-blog # verify
```

### Multi-store trick: stagger times

If you run two stores from the same Mac, stagger the times to avoid CLI auth contention:

- the store: 9:00 AM
- the store: 9:15 AM

## Gotchas

- **`articleCreate` requires `author: AuthorInput!` — not optional despite docs implying otherwise.** Minimal: `{"name": "<brand>"}`.
- **Mac must be awake at 9 AM.** If asleep, launchd fires when you wake the Mac. Set Energy Saver schedule to wake at 8:59 AM if you need exact firing.
- **Daylight Saving handled automatically.** launchd respects `America/Los_Angeles` system timezone — no manual toggle for PDT/PST transitions.
- **Shopify CLI's stdout/stderr mixing breaks subprocess parsing.** That's why we use `urllib.request` directly to the Admin GraphQL endpoint. CLI is for humans, raw GraphQL is for crons.
- **JSON parse on Claude output: strip markdown code fences first.** Sometimes Claude wraps `{...}` in `\`\`\`json ... \`\`\`` even when system prompt says "strict JSON."
- **Internal link prompt drift.** Claude will sometimes invent product handles that don't exist. Verify post-hoc, OR include a strict allow-list in the system prompt.

### Reusing for product SEO weekly audit

Same wrapper pattern — different schedule + different script. We use it for:

- `weekly_audit.py` — Mondays 9 AM, finds products updated in last 7 days with templated descriptions / missing tags, regenerates via Claude.
- `daily_giga.sh` — Same time, runs the GIGA dropshipping import + post-processing pipeline.

## Reference

- Cost: ~$0.05/post (claude-haiku-4-5, ~3000 tokens output × $5/M)
- First auto-published articles after ~3 days of running show up correctly in /sitemap.xml.
-
- Disabled remote-trigger approach (managed sandboxes may block your specific shop domain via network allowlist; we fell back to local launchd)

## Anti-pattern

Don't run this on a server you don't control. The Anthropic API key is in `config.env`. Use a personal Mac, a private VPS, or a managed Cloudflare Worker (rewrite in TS) — never a shared dev box.
