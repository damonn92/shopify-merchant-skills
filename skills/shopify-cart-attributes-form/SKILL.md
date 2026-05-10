---
name: shopify-cart-attributes-form
description: Replace 3rd-party cart-stage apps (Zapiet, Bookify, Local Delivery Pro) with native `attributes[…]` cart fields — date, time, address, service tier all attach to the order, free.
when_to_use:
  - You need delivery date/time pickers, address form, or service-tier picker on the cart page
  - The 3rd-party app you're using charges $50-200/month for what's basically a form with 6 fields
  - The 3rd-party app intercepts the Checkout button and breaks something (Zapiet did this for KZG)
not_for:
  - Real shipping rate calculation (use Shopify's Carrier Service API or a real shipping app)
  - Booking systems with availability windows / capacity (use a real booking app)
  - Anything that requires payment up-front (use a custom checkout extension)
---

## Why

Shopify cart attributes (`attributes[my-field-name]`) are key-value pairs that attach to the cart, persist into Order, and are visible in the admin. They cost nothing, render anywhere in cart Liquid, and don't intercept checkout.

For "delivery date + service tier + customer phone" — which is what 80% of Zapiet-style apps actually do — a `<form>` with 6 inputs is identical functionally and saves $50-200/month per store.

The trigger: KZG was using Zapiet for SoCal local delivery scheduling. Zapiet's cart widget intercepted the cart's Checkout submission (to validate "service area + lead time") and then sometimes failed silently — customer clicks Checkout, nothing happens, customer leaves. We swapped to native cart attributes in 2 hours.

## How

### Cart attribute form (Liquid)

```liquid
{%- comment -%} sections/cart-delivery-form.liquid {%- endcomment -%}
<div class="kzg-delivery-form">
  <h3>Delivery & Installation</h3>

  <label>
    Service tier
    <select name="attributes[service-tier]" form="cart-form">
      <option value="">— Choose —</option>
      <option value="pickup" {% if cart.attributes['service-tier'] == 'pickup' %}selected{% endif %}>Pickup at warehouse — Free</option>
      <option value="delivery" {% if cart.attributes['service-tier'] == 'delivery' %}selected{% endif %}>Local delivery only — $150</option>
      <option value="delivery+install" {% if cart.attributes['service-tier'] == 'delivery+install' %}selected{% endif %}>Delivery + Pro install — $350 (per set)</option>
      <option value="delivery+install+removal" {% if cart.attributes['service-tier'] == 'delivery+install+removal' %}selected{% endif %}>Delivery + Install + Old door removal — $430</option>
    </select>
  </label>

  <label>
    Preferred delivery date
    <input type="date" name="attributes[delivery-date]" form="cart-form" min="{{ 'now' | date: '%Y-%m-%d' }}" value="{{ cart.attributes['delivery-date'] }}">
  </label>

  <label>
    Time window
    <select name="attributes[delivery-window]" form="cart-form">
      <option value="9-12" {% if cart.attributes['delivery-window'] == '9-12' %}selected{% endif %}>Morning (9 AM – 12 PM)</option>
      <option value="12-3" {% if cart.attributes['delivery-window'] == '12-3' %}selected{% endif %}>Afternoon (12 – 3 PM)</option>
      <option value="3-6" {% if cart.attributes['delivery-window'] == '3-6' %}selected{% endif %}>Late afternoon (3 – 6 PM)</option>
    </select>
  </label>

  <label>
    Delivery address
    <textarea name="attributes[delivery-address]" form="cart-form" rows="3" placeholder="Street, city, zip — same as shipping if blank">{{ cart.attributes['delivery-address'] }}</textarea>
  </label>

  <label>
    Phone (for delivery coordinator)
    <input type="tel" name="attributes[customer-phone]" form="cart-form" value="{{ cart.attributes['customer-phone'] }}" placeholder="(555) 123-4567">
  </label>

  <p class="kzg-fee-summary">
    {%- assign tier = cart.attributes['service-tier'] -%}
    {%- if tier == 'delivery' -%}<strong>Delivery fee:</strong> $150 (added to invoice separately, not auto-charged at checkout)
    {%- elsif tier == 'delivery+install' -%}<strong>Delivery + install:</strong> $350 per set, multiplied by quantity. Invoiced separately.
    {%- elsif tier == 'delivery+install+removal' -%}<strong>Delivery + install + removal:</strong> $430 per set + $80 removal. Invoiced separately.
    {%- endif -%}
  </p>
</div>

<style>
  .kzg-delivery-form { padding: 16px; border: 1px solid #e5dfd3; border-radius: 8px; margin-bottom: 16px; }
  .kzg-delivery-form label { display: block; margin-bottom: 12px; font-size: 14px; }
  .kzg-delivery-form input, .kzg-delivery-form select, .kzg-delivery-form textarea {
    display: block; width: 100%; margin-top: 4px; padding: 8px; border: 1px solid #e5dfd3; border-radius: 4px;
  }
  .kzg-fee-summary { padding: 8px; background: rgba(204,120,92,0.08); border-radius: 4px; }
</style>
```

### Why `form="cart-form"` matters

Cart attributes only persist when the form is submitted with the cart. Putting `form="cart-form"` on the inputs ties them to the existing cart `<form>` (which Whisper, Dawn, Refresh all use). Without it, inputs would be orphaned.

### Updating attributes on input change (optional UX upgrade)

Default behavior: attributes persist when user clicks "Update cart" or "Checkout." But you can save on input via the `/cart/update.js` endpoint:

```javascript
document.querySelectorAll('.kzg-delivery-form select, .kzg-delivery-form input').forEach(el => {
  el.addEventListener('change', async (e) => {
    const name = e.target.name.match(/attributes\[(.+)\]/)[1];
    await fetch('/cart/update.js', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ attributes: { [name]: e.target.value } }),
    });
  });
});
```

This keeps the cart attribute fresh even if the user navigates back/forward.

### Where attributes show up

| Surface | Visible? | How |
|---|---|---|
| Cart page Liquid | ✅ | `{{ cart.attributes['service-tier'] }}` |
| Checkout — line item details (top-right summary) | ✅ | Built-in |
| Order confirmation email (admin) | ✅ | Add `{{ order.attributes }}` block |
| Order confirmation email (customer) | ❌ default | Edit Settings → Notifications → Order confirmation, add `{{ order.attributes }}` |
| Order admin view | ✅ | "Additional details" section |
| Order export CSV | ✅ | Add columns for the keys you care about |
| Shopify Flow trigger | ✅ | "Order created" → check attribute → notify Slack |

### Reading attributes server-side

Via Admin GraphQL `order.customAttributes`:

```graphql
query {
  order(id: "gid://shopify/Order/...") {
    customAttributes {
      key
      value
    }
  }
}
```

Or via Webhook `orders/create` payload: attributes land at `note_attributes: [{ name: 'service-tier', value: 'delivery+install' }]`.

## Gotchas

- **Attributes don't auto-multiply.** If customer buys 3 shower doors and selects "delivery+install", the form value is just the tier string. **Merchant manually multiplies $250 × set count when invoicing.** Communicate this in the form's helper text.
- **No native validation.** A customer can submit empty attributes. Either add JS validation, OR let the order through and reach out manually if missing.
- **Attribute keys are case-sensitive.** `service-tier` ≠ `Service-Tier`. Stick to lowercase-hyphenated.
- **Free-text addresses are messy.** Consider using Google Places API (paid) for autocomplete + structured data. For local-only delivery in a known area, free-text is fine.
- **Don't auto-charge.** Cart attributes don't affect cart total. If you need to add the fee to checkout, use a "shipping rate" via Carrier Service API or a "tip" line item — both are heavier patterns. Most local-delivery merchants prefer to invoice separately and pre-warm the customer in the form's helper copy.

## Reference

- KZG fee schedule (real-world numbers):
  - Pickup: $0
  - Delivery only: $150
  - Delivery + Install: $350 (one set) — $150 + $250 - $50 combo discount
  - Delivery + Install + Old door removal: $430 (one set) — adds $80
- Switched from: Zapiet ($50/mo, intercepted Checkout button)
- Switched to: native attributes (free, 30 lines of Liquid)
- Customer-facing copy must say "Invoiced separately, not auto-charged at checkout" to avoid surprise

## Anti-pattern

Don't put the fee schedule INSIDE the form as an "auto-add line item" via `cart/add.js` of a $150 service product. That double-charges customers who picked "Delivery only" but already have free shipping rules. Keep fees separate; invoice manually.
