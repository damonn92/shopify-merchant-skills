---
name: shopify-design-wizard
description: Multi-step product configurator wizard (DreamLine-style) on a Shopify page template, with URL-restorable state, JSON-driven option catalogs in shop metafields, and PDP handoff for purchase.
when_to_use:
  - You have a custom-product line with many configurable dimensions (shower doors, suits, kitchens)
  - You want a "design center" landing page that funnels visitors into the buy flow
  - You need URL-shareable wizard state (link sales reps + customer can resume)
not_for:
  - Simple variant selectors (use Shopify's native variant picker)
  - One-step quote forms (use a simple `<form action="/cart/add">`)
---

## Why

Selling a custom shower door isn't "pick a SKU." Customers need 5 decisions:

1. Series (style/budget tier)
2. Glass + finish + handle (option matrix)
3. Walls (alcove vs corner)
4. Door type (sliding/pivot/swing/screen)
5. Dimensions (W × H × depth, fractional)

DreamLine has 6 steps; we collapsed to 5 + PDP handoff. The trick: **all 5 steps render server-side simultaneously, JS toggles visibility**, because Shopify Liquid's `request.query_string` returns **empty for cached page templates** — meaning you can't read `?step=options` server-side reliably.

## How

### Architecture

```
templates/page.design-center.json
  → sections/<brand>-design-center.liquid (wrapper)
    → snippets/<brand>-dc-step1..5.liquid (all 5 rendered, gated by data-dc-step)
    → snippets/<brand>-dc-progress.liquid
    → snippets/<brand>-dc-series-card.liquid
    → snippets/<brand>-dc-option-group.liquid
    → snippets/<brand>-dc-running-total.liquid
    → snippets/<brand>-dc-iso-svg.liquid (inline isometric drawings per variant)
  → assets/<brand>-dc.css (brand styling)
  → assets/<brand>-dc-state.js (URL state machine + sessionStorage + step router)
  → assets/<brand>-dc-step2.js (Step 2 live price calculator)
  → assets/<brand>-dc-step5.js (Step 5 dim form + PDP redirect)

Shop metafields drive the JSON catalog:
  <brand>_cpq.series_catalog
  <brand>_cpq.options_catalog
  <brand>_cpq.variant_configs
```

### Three shop-level metafields = single source of truth

```json
// <brand>_cpq.series_catalog
[
  { "key": "origin",  "title": "Origin",  "starts_at": 217, "hero_image": "https://cdn.shopify.com/s/files/.../series-origin-hero.png",
    "variant_types": ["alcove-swing"] },
  { "key": "vista",   "title": "Vista",   "starts_at": 434, "hero_image": "...",
    "variant_types": ["alcove-swing"] },
  { "key": "forma",   "title": "Forma",   "starts_at": 420, "hero_image": "...",
    "variant_types": ["alcove-swing", "alcove-swing-buttress", "alcove-swing-inline"] },
  { "key": "forma-x", "title": "Forma X", "starts_at": 553, "hero_image": "...",
    "variant_types": ["corner-swing-x"] }
]

// <brand>_cpq.options_catalog
{
  "glass": [
    { "key": "clear",     "title": "Clear",     "premium": 0 },
    { "key": "frosted",   "title": "Frosted",   "premium": 50 },
    { "key": "fluted",    "title": "Fluted",    "premium": 80 },
    { "key": "rain",      "title": "Rain",      "premium": 80 },
    { "key": "smoked",    "title": "Smoked",    "premium": 60 },
    { "key": "etched",    "title": "Etched",    "premium": 100 }
  ],
  "finish": [
    { "key": "chrome",          "premium": 0 },
    { "key": "matte-black",     "premium": 50 },
    { "key": "brushed-nickel",  "premium": 30 },
    { "key": "polished-chrome", "premium": 30 },
    { "key": "antique-bronze",  "premium": 60 },
    { "key": "brushed-gold",    "premium": 80 }
  ],
  "handle": [...],
  "addons": [
    { "key": "buttress", "title": "Buttress panel",  "premium": 250 },
    { "key": "inline",   "title": "Inline panel",    "premium": 200 },
    { "key": "shelves",  "title": "Glass shelves",   "premium": 80 }
  ]
}

// <brand>_cpq.variant_configs (per-variant pricing schema)
{
  "alcove-swing": {
    "label": "Alcove Swing",
    "dimensions": { "W": { "min": 23, "max": 36 }, "H": { "min": 70, "max": 78 } },
    "pricing": {
      "base": 217, "base_width": 28,
      "per_inch_width_above_base": 8,
      "tolerance_surcharge": 25
    }
  },
  "alcove-swing-buttress": { /* ... */ },
  "corner-swing-x": { /* ... */ }
}
```

### URL state machine (`assets/<brand>-dc-state.js`)

```javascript
const STATE = {
  step: 'series',
  series: null,        // origin / vista / forma / forma-x
  glass: null,
  finish: null,
  handle: null,
  walls: null,         // alcove / corner
  variant_hint: null,  // alcove-swing / corner-swing-x / etc.
  dw: null, df: null, dh: null, // dimensions
  addons: [],
};

function loadFromURL() {
  const p = new URLSearchParams(location.search);
  for (const k of Object.keys(STATE)) {
    if (p.has(k)) STATE[k] = k === 'addons' ? p.get(k).split(',') : p.get(k);
  }
}

function saveToURL() {
  const p = new URLSearchParams();
  for (const k of Object.keys(STATE)) {
    if (STATE[k] != null && STATE[k] !== '' && STATE[k].length !== 0) {
      p.set(k, Array.isArray(STATE[k]) ? STATE[k].join(',') : STATE[k]);
    }
  }
  history.replaceState({}, '', location.pathname + '?' + p.toString());
  sessionStorage.setItem('dcState', JSON.stringify(STATE));
}

function showStep(step) {
  document.querySelectorAll('[data-dc-step]').forEach(el => {
    el.classList.toggle('kzg-dc-step--active', el.dataset.dcStep === step);
  });
  STATE.step = step;
  saveToURL();
}
```

### URL pattern for sharing

```
/pages/design-center                                                     Step 1 (series)
?step=options&series=forma                                               Step 2 (glass/finish/handle)
?step=walls&series=forma&glass=fluted&finish=matte-black                 Step 3 (alcove vs corner)
?step=door-type&series=forma&...&walls=alcove&variant_hint=alcove-swing  Step 4 (sliding/pivot/swing)
?step=dimensions&series=forma&...&dw=58&df=0.625&dh=76                   Step 5 (W × H × depth)
→ /products/kzg-alcove-swing-shower-door?from=design-center&...          Step 6 (PDP, CPQ configurator)
```

### Step transitions

```javascript
// Step 1 → Step 2 (series chosen)
function chooseSeries(seriesKey) {
  STATE.series = seriesKey;
  STATE.variant_hint = SERIES_CATALOG[seriesKey].variant_types[0]; // pre-pick first compatible
  showStep('options');
}

// Step 5 → PDP (dimensions confirmed)
function continueToPDP() {
  const productHandle = VARIANT_TO_HANDLE[STATE.variant_hint];
  const params = new URLSearchParams();
  params.set('from', 'design-center');
  params.set('series', STATE.series);
  params.set('glass', STATE.glass);
  // ... etc
  location.href = `/products/${productHandle}?${params.toString()}`;
}
```

### Live price preview (Step 2)

```javascript
function recomputePrice() {
  const series = SERIES_CATALOG[STATE.series];
  const cfg = VARIANT_CONFIGS[STATE.variant_hint];
  let p = series.starts_at;
  p += (OPTIONS_CATALOG.glass.find(g => g.key === STATE.glass)?.premium ?? 0);
  p += (OPTIONS_CATALOG.finish.find(f => f.key === STATE.finish)?.premium ?? 0);
  p += (OPTIONS_CATALOG.handle.find(h => h.key === STATE.handle)?.premium ?? 0);
  for (const a of STATE.addons) {
    p += (OPTIONS_CATALOG.addons.find(x => x.key === a)?.premium ?? 0);
  }
  document.querySelector('.kzg-dc-running-total').textContent = `$${p}`;
}
```

## Gotchas

- **Section name 25-char limit.** Schema's `name` field caps at 25 chars. Use `DC Step N · <Name>` form.
- **Liquid `request.query_string` returns empty for cached templates.** Don't try to read URL params server-side. Render all steps + JS toggles.
- **Each step snippet needs an SSR fallback.** Default `series_key='forma'` and `variant_hint='alcove-swing'` so the page is meaningful even before JS runs.
- **`{% render %}` scope blocks query parsing.** A `parse-query` snippet that tries to set parent vars from `request.query_string` won't work. Documentation-only.
- **PDP handoff is "fresh wizard," not pre-filled.** The PDP CPQ configurator doesn't auto-restore from `?from=design-center` URL params (yet). Either build that, or accept a single re-pick.
- **Hero images via stagedUploadsCreate, not productCreateMedia.** Series hero images go to Shopify Files (shop-wide), not product media.

## Reference

- KZG Design Center live: `https://thekzg.com/pages/design-center`
- 4 series in Phase 1: Origin / Vista / Forma / Forma X
- Hero image dimensions: 2000×1500 PNG, transparent or scene-aware backgrounds (we generated via ChatGPT desktop app)
- Worker integration: see [`shopify-cpq-via-worker`](../shopify-cpq-via-worker/SKILL.md) for the pricing endpoint that backs the PDP handoff

## Anti-pattern

Don't render only the current step. The cached-template empty `request.query_string` will leave you with a broken initial render. Render all 5, gate via JS — the extra DOM is < 30KB.
