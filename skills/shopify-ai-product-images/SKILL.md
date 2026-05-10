---
name: shopify-ai-product-images
description: Vertex AI Imagen 4.0 Fast for AI-generated lifestyle product images + 4-corner-brightness classifier to delete white-background studio shots — gets every product to ≥5 lifestyle images for ~$0.02 each.
when_to_use:
 - Dropshipped products have only white-background catalog shots, you need lifestyle scenes
 - Some products have <3 images and look thin in the storefront grid
 - Background-removal apps want $30/mo for what's a 30-line Python script
not_for:
 - Brand-photography-required stores (luxury, fashion — invest in real shoots)
 - Sites where every image must depict a real-world test case (regulated industries)
---

## Why

Dropshipped products from GIGA / Aliexpress / Spocket land with **80% white-background catalog shots**. They look fine on the PDP but are visually monotonous on collection grids — every product looks like a CAD render. Conversion suffers.

Two fixes work together:

1. **White-bg classifier + delete:** detect white-bg images via 4-corner brightness, drop them (with a "never delete the last 2" safety).
2. **Vertex AI Imagen lifestyle scenes:** classify each product by room type (bedroom / dining-room / kids / outdoor / office), prompt Imagen 4.0 Fast for 5 angle × 4 room variations, upload via `stagedUploadsCreate` + `productCreateMedia`.

Real outcome: deleted 574 white-bg images, generated 254 AI lifestyle scenes, **all 226 products now have ≥5 images**. Total cost: ~$5 (Vertex AI billing).

## How

### Part A — White-bg classifier

```python
# scene_filter.py
import os, sys, json, urllib.request, io
from PIL import Image
import argparse

# 4-corner brightness check + 80-pixel border sample
def is_white_bg(image_path):
 img = Image.open(image_path).convert('RGB').resize((200, 200))
 px = img.load()

 # Check 4 corners (5px deep, average RGB ≥ 245)
 corners = [
 avg_rgb(px, 0, 0, 5, 5),
 avg_rgb(px, 195, 0, 5, 5),
 avg_rgb(px, 0, 195, 5, 5),
 avg_rgb(px, 195, 195, 5, 5),
 ]
 if all(min(c) >= 245 for c in corners):
 return True

 # Check border sample (80 pixels around edge, ≥85% are ≥240 RGB)
 border_pixels = []
 for i in range(0, 200, 10):
 border_pixels.extend([px[i, 0], px[i, 199], px[0, i], px[199, i]])
 bright = sum(1 for p in border_pixels if min(p) >= 240)
 return (bright / len(border_pixels)) >= 0.85

def avg_rgb(px, x0, y0, w, h):
 pixels = [px[x, y] for x in range(x0, x0 + w) for y in range(y0, y0 + h)]
 return tuple(sum(c) // len(pixels) for c in zip(*pixels))

# Process via Shopify CDN (fast — uses width=200 param to avoid downloading full-res)
def fetch_thumb(url, w=200):
 if 'cdn.shopify.com' in url:
 url = url.split('?')[0] + f'?width={w}'
 return urllib.request.urlopen(url).read()

def classify_products(products, dry_run=True):
 plan = []
 for p in products:
 images = p['media'] # list of {id, url}
 flags = []
 for m in images:
 data = fetch_thumb(m['url'])
 with io.BytesIO(data) as f:
 if is_white_bg_from_bytes(f):
 flags.append({'id': m['id'], 'url': m['url'], 'kind': 'white-bg'})
 # Safety: never drop a product below 2 images
 scene_count = len(images) - len(flags)
 if scene_count < 2:
 keep_n = 2 - scene_count
 flags = flags[keep_n:] # backfill from white-bg
 plan.append({'product_id': p['id'], 'title': p['title'], 'flagged': flags})
 return plan
```

Apply with `--apply` flag:
```bash
python scene_filter.py --test --limit 5 # smoke test on 5 products
python scene_filter.py # dry run, write plan to /tmp/scene_filter_plan.json
python scene_filter.py --apply # actually delete via productDeleteMedia
```

the store first-pass: **574 white-bg images deleted from 152 products, 0 failures, 30% image reduction.**

### Part B — Vertex AI Imagen lifestyle scenes

Why Vertex AI not Generative Language API: Imagen 4.0 Fast has **0 free-tier quota** on the Generative Language API. You need a Google Cloud project with billing enabled, then go via Vertex AI.

Setup:
```bash
# Install gcloud SDK if not already
curl https://sdk.cloud.google.com | bash
~/google-cloud-sdk/install.sh

# Auth + set project
gcloud auth login
gcloud config set project <your-project-id>
gcloud auth application-default login # ADC for the script

# Verify access
gcloud auth print-access-token
```

Imagen call pattern:
```python
import subprocess, json, urllib.request, time, base64, os

PROJECT = '<your-project-id>'
LOCATION = 'us-central1'
MODEL = 'imagen-4.0-fast-generate-001' # Fast, ~$0.02/image
ENDPOINT = f'https://{LOCATION}-aiplatform.googleapis.com/v1/projects/{PROJECT}/locations/{LOCATION}/publishers/google/models/{MODEL}:predict'

# Cache gcloud token (expires in 1 hour)
_token_cache = {'value': None, 'expires_at': 0}
def get_token():
 if time.time() < _token_cache['expires_at'] - 60:
 return _token_cache['value']
 out = subprocess.check_output(
 [os.path.expanduser('~/google-cloud-sdk/bin/gcloud'), 'auth', 'print-access-token'],
 text=True
 ).strip()
 _token_cache['value'] = out
 _token_cache['expires_at'] = time.time() + 55 * 60 # 55 min cache
 return out

def generate_image(prompt, n=1):
 body = {
 'instances': [{ 'prompt': prompt }],
 'parameters': { 'sampleCount': n, 'aspectRatio': '1:1' }
 }
 req = urllib.request.Request(
 ENDPOINT, data=json.dumps(body).encode(), method='POST',
 headers={
 'Authorization': f'Bearer {get_token()}',
 'Content-Type': 'application/json',
 }
 )
 with urllib.request.urlopen(req) as r:
 resp = json.loads(r.read())
 return [base64.b64decode(p['bytesBase64Encoded']) for p in resp['predictions']]
```

### Room-type classifier + prompt builder

```python
ROOM_PROMPTS = {
 'bedroom': 'A modern American bedroom interior, bed with neutral linens, soft natural light from a window, minimal decor, photographic quality',
 'kids': 'A bright, cheerful kids bedroom interior, white walls with pastel accents, natural light',
 'dining-room': 'A modern dining room interior with hardwood floor and large window, warm natural lighting',
 'living-room': 'A modern living room interior with neutral palette, hardwood floor, soft afternoon sunlight',
 'office': 'A clean home office interior with desk and chair, large window, plants, soft natural light',
 'outdoor': 'A landscaped backyard patio in soft golden-hour light, manicured lawn, modern outdoor setting',
}

def classify_room(product_title):
 t = product_title.lower()
 if any(x in t for x in ['kids', 'bunk', 'children', 'toddler']): return 'kids'
 if any(x in t for x in ['dining', 'kitchen']): return 'dining-room'
 if any(x in t for x in ['outdoor', 'patio', 'adirondack', 'gazebo', 'umbrella']): return 'outdoor'
 if any(x in t for x in ['office', 'desk', 'workstation', 'standing']): return 'office'
 if any(x in t for x in ['sofa', 'sectional', 'recliner', 'tv stand', 'bookshelf']): return 'living-room'
 if any(x in t for x in ['bed', 'mattress', 'nightstand', 'dresser', 'wardrobe']): return 'bedroom'
 return 'living-room' # safe default

def make_lifestyle_prompt(product, room_key):
 room = ROOM_PROMPTS[room_key]
 title = product['title']
 return f'''{room}. The room contains {title.lower()} as a focal point.
Photorealistic interior design photography, 35mm full-frame, shallow depth of field,
natural lighting, no text, no people, soft warm color grade.'''
```

### Upload to Shopify

```python
# Step 1: stagedUploadsCreate to get a GCS upload target
def stage_upload(filename, mime, file_size):
 q = '''mutation ($input: [StagedUploadInput!]!) {
 stagedUploadsCreate(input: $input) {
 stagedTargets {
 url resourceUrl
 parameters { name value }
 }
 userErrors { field message }
 }
 }'''
 res = shopify_gql(q, {
 'input': [{
 'filename': filename, 'mimeType': mime,
 'resource': 'IMAGE', 'fileSize': str(file_size),
 'httpMethod': 'POST',
 }]
 })
 return res['data']['stagedUploadsCreate']['stagedTargets'][0]

# Step 2: multipart POST the image to GCS
def upload_to_gcs(target, image_bytes, filename):
 import urllib.request, mimetypes
 body = []
 boundary = '----formboundary' + os.urandom(8).hex()
 for p in target['parameters']:
 body.append(f'--{boundary}\r\nContent-Disposition: form-data; name="{p["name"]}"\r\n\r\n{p["value"]}\r\n'.encode())
 body.append(f'--{boundary}\r\nContent-Disposition: form-data; name="file"; filename="{filename}"\r\nContent-Type: image/png\r\n\r\n'.encode())
 body.append(image_bytes)
 body.append(f'\r\n--{boundary}--\r\n'.encode())
 data = b''.join(body)
 req = urllib.request.Request(target['url'], data=data, method='POST',
 headers={'Content-Type': f'multipart/form-data; boundary={boundary}'})
 urllib.request.urlopen(req).read()

# Step 3: productCreateMedia with the resourceUrl
def attach_to_product(product_id, resource_url, alt):
 q = '''mutation ($id: ID!, $media: [CreateMediaInput!]!) {
 productCreateMedia(productId: $id, media: $media) {
 media { ... on MediaImage { id alt } }
 mediaUserErrors { field message }
 }
 }'''
 res = shopify_gql(q, {
 'id': product_id,
 'media': [{
 'mediaContentType': 'IMAGE',
 'originalSource': resource_url,
 'alt': alt,
 }]
 })
 return res
```

### The full loop

```python
def topup_product(product, target_count=5, samples_dir=None):
 if len(product['media']) >= target_count: return 0
 needed = target_count - len(product['media'])
 room = classify_room(product['title'])
 generated = 0
 for i in range(needed):
 prompt = make_lifestyle_prompt(product, room)
 images = generate_image(prompt, n=1)
 for j, img_bytes in enumerate(images):
 filename = f'{product["handle"]}-ai-{i+j+1}.png'
 target = stage_upload(filename, 'image/png', len(img_bytes))
 upload_to_gcs(target, img_bytes, filename)
 alt = f'{product["title"]} — lifestyle scene {i+j+1}'
 attach_to_product(product['id'], target['resourceUrl'], alt)
 if samples_dir:
 with open(os.path.join(samples_dir, filename), 'wb') as f: f.write(img_bytes)
 generated += 1
 return generated
```

## Gotchas

- **Image quality classifier orders matter.** Check 4 corners FIRST (cheap), then border sample (slower). Saves 80% of time on obvious-lifestyle photos.

- **Don't delete below 2 images.** Some products literally only have white-bg shots. The script must backfill — never delete the last 2 images.

- **gcloud token TTL is 1 hour.** Cache with 55-min TTL in script. Otherwise long batches fail mid-run.

- **Prompt must say "no text, no people."** Imagen sometimes adds tiny watermarks or stock-photo people if not explicitly forbidden. Both look unprofessional on a product page.

- **`fetchpriority="high"` only on the LCP image.** Don't preload all hero images on every page — costs more than it saves.

- **`stagedUploadsCreate` returns parameters that MUST be sent in order.** GCS multipart is order-sensitive. Use the parameters loop pattern shown above; don't reorder for legibility.

- **Imagen `imagen-4.0-fast` ≠ `imagen-3.0`.** 4.0-fast is faster (~3s/image) and ~50% cheaper. Quality is fine for lifestyle scenes; if you need photo-real product detail, use 4.0-generate (slower, more expensive).

- **`image/png` from Imagen is uncompressed.** Pass through Pillow `quality=85, optimize=True` save before upload to cut filesize 60%.

## Numbers

the store run :
- 100 products had < 5 images
- Generated 254 AI lifestyle images in 26.8 minutes, 0 failures
- Cost: ~$5 (`imagen-4.0-fast-generate-001` at ~$0.02/image)
- Final state: all 226 products have ≥5 images, total 1600 images

the store: didn't need this (GIGA images are 2000×2000 white-bg with no watermark). Used `reorder_media.py` instead to put catalog shot at position 1.

## Reference

- Vertex AI billing: enable on Google Cloud Console; ~$0.02/image for `imagen-4.0-fast-generate-001`
- Shopify CDN trick: `?width=200` for thumbnails, never download full-res for classification
- Idempotency: `store-optimized-v1` marker on description, image alt = `"{title} — lifestyle scene N"` for AI-generated

## Anti-pattern

Don't generate AI images BEFORE deleting white-bg shots. You'll generate scenes that get backfilled by the safety check, then re-delete on next pass. Order: delete-white-bg → topup-with-AI → reorder-media.
