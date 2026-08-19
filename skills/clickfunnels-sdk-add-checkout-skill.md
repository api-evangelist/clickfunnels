---
name: clickfunnels-sdk-checkout
description: >
  Add checkout to an external page - a page you host yourself that is already
  registered as a ClickFunnels funnel step with the SDK embedded. Covers
  attaching products in ClickFunnels, the data-cf-element checkout form
  (payment-method, pay, shipping, billing), the CfSdk.products() /
  CfSdk.checkout.* JavaScript API, order bumps, one-click one-time offers
  (oto-accept / oto-decline), styling the payment fields with
  --cfsdk-checkout-* design tokens, and testing in test mode. Use this skill
  when selling products from an externally-hosted page: order forms, upsell
  pages, and multi-step sales funnels.
version: "1.0"
author: ClickFunnels
references:
  - path: /llms.txt
    description: Product overview + skills index.
  - path: /.well-known/sdk/create-external-page/skill.md
    description: Prerequisite - register the page, embed sdk.js, and mark contact fields.
  - path: /.well-known/funnels/skill.md
    description: Build the funnel whose steps these pages are.
---

# ClickFunnels SDK - Add Checkout to an External Page

Checkout turns a registered external page into an order form: the SDK mounts
the payment gateway's card UI, prices the cart on the server (tax, discounts,
shipping), charges the buyer - including 3-D Secure challenges - and redirects
the browser to the funnel's next step. A follow-up page can then make a
**one-click one-time offer (OTO)** against the card saved on the first
purchase. Both ClickFunnels payment providers are supported (Stripe and
Payments AI); the page markup is identical for both.

## Prerequisites

Everything in the
[Create an External Page skill](/.well-known/sdk/create-external-page/skill.md)
applies here and is not repeated: the page must be registered as an external
page step, carry its `cfp_` token in `<meta name="cf-page-token">`, and load
`sdk.js`. Checkout adds product configuration and a few more elements on top.

## Configure ClickFunnels first

Checkout is driven by what's attached to the page's **funnel step**. The whole
catalog can be configured over the API - see "Attach products over the API"
below - or by hand in the funnel builder; both land in the same place.

- **Attach the product(s) you're selling** to the step: `PATCH
  /api/v2/pages/:page_id` with `show_page_step.product_ids`, or in the funnel
  builder click the step's products area, then **Add product**.
- **Set each variant's `product_type` to match the real-world goods** -
  `physical` makes checkout collect a shipping address (and shipping
  options); `digital` skips both. An UNSET product_type behaves like
  digital, so physical goods sold with it unset silently produce orders
  with no shipping address.
- **Order bump**: attach the product via `show_page_step.bumps`
  (one `{product_id, preheadline}` object per bump), or in the funnel
  builder drag a product into the step's **Order bump** section (or use the
  settings icon on an attached product). Only products marked as bumps are
  served to the page in `bumpProducts` - a regularly-attached product will not
  appear as one.
- **OTO**: attach the offered product to the *next* step, itself registered
  as another external page (see "One-time offers" below).
- The funnel needs a configured payment gateway, and checkout must be enabled
  for your workspace. If either is missing, the SDK logs
  `products(): no checkout configuration for this page` and no payment UI
  mounts.

While the funnel is not live, payments run in **test mode**: use the
gateway's test cards, and the SDK shows a debug overlay badge on the page
(see "Testing and troubleshooting").

### Attach products over the API

`PATCH /api/v2/pages/:page_id` configures the step's catalog in one call - both
blocks are **additive** (products are appended, already-attached products are
skipped, nothing is ever removed):

```
PATCH /api/v2/pages/<page_id>
Content-Type: application/json

{
  "page": {
    "show_page_step": {
      "product_ids": ["<main_product_id>"],
      "bumps": [
        {"product_id": "<bump_product_id>", "preheadline": "One-time offer"}
      ]
    }
  }
}
```

The two blocks are **independent** - each one only affects the products it
names, and neither is paired with the other by position. Send either, both, or
neither:

| Sent | Result |
| --- | --- |
| `product_ids` only | Those products are attached as main products. Bumps already on the step keep their flag and copy. |
| `bumps` only | Those products are attached (or promoted) as order bumps. Main products already on the step are untouched. |
| Both | Each list is applied on its own terms - mains as mains, bumps as bumps. Mains are appended first, so they sort ahead of the bumps. |
| 2 in `product_ids`, 1 in `bumps` | Two main products and one bump. No pairing is inferred: the bump is not "the bump for" either main product, and its `preheadline` applies to that one bump only. |
| The same product in both | `400`. On a given step a product is one or the other, so nothing is saved. |

- A product **already attached** as a main product is promoted to a bump when
  you send it in `bumps`; it keeps its position, variants and prices. That is
  how an existing attachment becomes a bump.
- `preheadline` is the line of copy above that one bump's offer - each bump
  carries its own, which is why they travel together in one object. Sending it
  for a product that is already a bump replaces that bump's copy, so it is
  also how you edit it; omit it to leave the current copy alone.
- The page must already be a funnel step (`400` otherwise), and the request is
  all-or-nothing: one unresolvable product id saves nothing. A bump with no
  sellable price is rejected with a `422`.
- Read the result back on any page response under `show_page_step.products[]` -
  each entry carries `bump` and `bump_preheadline` (the copy you sent as that
  bump's `preheadline`).
- To remove a product, or to turn a bump back into a main product, detach it
  with `DELETE /api/v2/pages/:page_id/products` (body `{"product_ids": [...]}`)
  and re-attach it if needed. Detaching works the same for bumps and main
  products.
- The same `show_page_step` block works on `POST
  /api/v2/workspaces/:workspace_id/pages` for ClickFunnels-hosted pages, but
  NOT on `POST /api/v2/workspaces/:workspace_id/pages/external` (`422`) -
  create the external page first, then PATCH it.

The products, variants and prices themselves are created through the Products
API; these fields only attach products that already exist in the workspace.

## The checkout form

Checkout uses the same single-attribute convention as opt-ins - mark elements
with `data-cf-element`, all inside **one `<form>`**:

```html
<form aria-labelledby="checkout-heading">
  <h2 id="checkout-heading">Checkout</h2>
  <div id="checkout-errors" role="alert" aria-live="assertive" aria-atomic="true" hidden></div>

  <label for="checkout-email">Email <span aria-hidden="true">*</span></label>
  <input id="checkout-email" name="email" data-cf-element="email" type="email" autocomplete="email" required>
  <label for="checkout-first-name">First name</label>
  <input id="checkout-first-name" name="first_name" data-cf-element="first-name" autocomplete="given-name">
  <label for="checkout-last-name">Last name</label>
  <input id="checkout-last-name" name="last_name" data-cf-element="last-name" autocomplete="family-name">

  <section id="shipping-section" aria-labelledby="shipping-heading">
    <h3 id="shipping-heading">Shipping address</h3>
    <div data-cf-element="shipping" aria-labelledby="shipping-heading"></div>
  </section>
  <section id="billing-section" aria-labelledby="billing-heading">
    <h3 id="billing-heading">Billing address</h3>
    <div data-cf-element="billing" aria-labelledby="billing-heading"></div>
  </section>
  <h3 id="payment-heading">Payment details</h3>
  <div data-cf-element="payment-method" aria-labelledby="payment-heading"></div>

  <button data-cf-element="pay" type="submit">Pay now</button>
</form>
```

Checkout-specific elements (contact fields and `custom-attribute:<key>` are
documented in the prerequisite skill):

| Value                      | Tag                                   | Purpose                                                        |
| -------------------------- | ------------------------------------- | -------------------------------------------------------------- |
| `payment-method`           | `<div>` (empty)                       | Payment mount point - the SDK renders the gateway's card UI    |
| `pay`                      | `<button>` or `<input type="submit">` | Submits the purchase; the SDK disables it while a charge runs  |
| `shipping`                 | `<div>` (empty)                       | Shipping-address mount point (checkout form or OTO page)      |
| `billing`                  | `<div>` (empty)                       | Billing-address mount point                                    |
| `billing-same-as-shipping` | `<input type="checkbox">`             | Optional; checked mirrors shipping into billing                |
| `oto-accept`               | `<button>`                            | OTO pages only - one-click buy; `value="variantId\|priceId"`   |
| `oto-decline`              | `<button>`                            | OTO pages only - advance the funnel without buying             |

Notes:

- The mount-point `<div>`s stay **empty** in your HTML - the SDK owns their
  contents. Write only the tag and the attribute.
- Do not read, replace, or attach behavior to the contents the SDK mounts.
  Those descendants belong to the SDK or payment provider and may change.
  Customize checkout through your own surrounding HTML, the public
  `CfSdk.checkout.*` API, CSS design tokens, and the supported class hooks
  documented below.
- Labels, IDs, native `required`, and `autocomplete` tokens belong to the host
  page's controls. The SDK-generated address form supplies those semantics for
  its own controls and marks the state/region required only when the selected
  country has a region list.
- **Best practice: always include the `shipping` element, even when today's
  product is digital.** The SDK shows `shipping`/`billing` only while the cart
  and gateway require them (a digital-only cart never shows an address form;
  Stripe collects billing inside its own payment UI), so the element costs
  nothing while hidden - and a product later changed to physical in the
  funnel settings just works, with no page change. Omit it and that same
  product change breaks checkout with a shipping-address error and nowhere
  to enter one. The same always-include rule applies on OTO pages (see
  "Physical-product OTOs"). They show/hide without any of your JS - wrap
  them in sections whose visibility is pure CSS:

  ```css
  #shipping-section { display: none; }
  #shipping-section:has([data-cf-element="shipping"] *):not(:has([data-cf-element="shipping"][hidden])) { display: block; }
  #billing-section { display: none; }
  #billing-section:has([data-cf-element="billing"] *):not(:has([data-cf-element="billing"][hidden])) { display: block; }
  ```

- Adding `data-cf-visible` to the element keeps the address form visible at
  all times, regardless of product type - for merchants who want to collect
  shipping information on a digital product that doesn't require it:
  `<div data-cf-element="shipping" data-cf-visible></div>`
- Contact identity fields (`email`, `first-name`, `last-name`, `name`,
  `phone`) prefill automatically for a visitor moving through the funnel in
  the same session - what they typed on an earlier opt-in step fills any
  still-empty inputs here. Values you set (or the visitor types) always win.
  Separate first/last values are combined for a single `name` input; a single
  `name` value is not split into separate first/last inputs.
- On success there is **no success event** - the browser follows the server's
  redirect to the funnel's next step. Failures surface through `onError`
  (below) and the pay button re-enables.

## Put the product in the cart

The form markup gives the SDK somewhere to mount and submit payment, but the
SDK does not guess which attached product or price to sell. For a
single-product order form, the only custom JavaScript required is selecting
that line after the catalog loads:

```html
<script>
  document.addEventListener('DOMContentLoaded', () => {
    CfSdk.products().then(({ products }) => {
      const variant = products[0]?.variants.find(({ prices }) => prices.length > 0)
      const price = variant?.prices[0]
      if (price) CfSdk.checkout.setQuantity(variant.id, price.id, 1)
    })
  })
</script>
```

Stop here unless the page needs a product picker, quantities, order bumps,
custom totals, coupons, or shipping-option selection. The SDK owns payment
mounting, charging, 3-D Secure, and redirects.

## Advanced: customize the cart with JavaScript

`CfSdk` is the supported host-page API, not access to SDK internals. Use it to
change cart intent or render server-computed checkout state; do not manipulate
the SDK's mounted payment or address DOM.

`window.CfSdk` is callable as soon as the `sdk.js` script tag has executed
(calls queue until the SDK finishes booting). With `defer` on the script tag,
run your code on `DOMContentLoaded` or later.

| Call | Returns | Purpose |
| ---- | ------- | ------- |
| `CfSdk.products()` | `Promise<{products, bumpProducts}>` | The step's catalog - main products and order bumps |
| `CfSdk.checkout.setQuantity(variantId, priceId, qty)` | `Promise<CheckoutState>` | Set a cart line (0 removes it) |
| `CfSdk.checkout.updateState({...})` | `Promise<CheckoutState>` | Partial update: `coupon_codes: []`, `shipping_address`, `billing_address` |
| `CfSdk.checkout.setShipping(optionId \| null)` | `Promise<CheckoutState>` | Pick a shipping option from `state.shipping.options` |
| `CfSdk.checkout.currentState()` | `Promise<CheckoutState>` | Read the current state |
| `CfSdk.checkout.onStateChange(fn)` | unsubscribe `fn` | Fires on **every** committed state - your single render path |
| `CfSdk.checkout.onPending(fn)` | unsubscribe `fn` | A mutation or charge is in flight |
| `CfSdk.checkout.onError(fn)` | unsubscribe `fn` | A state carrying errors was committed |
| `CfSdk.checkout.onOtoOutcome(fn)` | unsubscribe `fn` | OTO pages only - see "One-time offers" |

`CfSdk.products()` resolves with the catalog (empty arrays - never a
rejection - when checkout isn't configured for the page, alongside the console
warning above):

```jsonc
{
  "products": [
    { "id": "JWkEqJ", "name": "The Course",
      "variants": [
        { "id": "nJdDJd", "name": "Default", "product_type": "digital",
          "prices": [
            { "id": "BNAqxN", "name": null, "amount": "149.0", "currency": "usd",
              "payment_type": "one_time", "interval": null, "interval_count": null }
          ] }
      ] }
  ],
  "bumpProducts": []  // same Product shape; a bump may carry "bump_preheadline"
}
```

Every mutator resolves with the same `CheckoutState` record, priced by the
server:

```jsonc
{
  "line_items": [
    { "variant_id": "nJdDJd", "price_id": "BNAqxN", "quantity": 1,
      "name": "The Course", "unit_amount": 14900, "line_amount": 14900 }
  ],
  "totals": { "subtotal": 14900, "discount": 0, "tax": 0, "shipping": 599,
              "total": 15499, "currency": "USD" },
  "shipping": { "address_required": true,  // an address must be collected on this form
                "options": [ { "id": "hyNOQZ", "name": "Standard", "amount": 599 } ],
                "selected_option_id": "hyNOQZ" },
  "errors": []   // { code, message, field?, source: "validation"|"gateway"|"sdk" }
}
```

Two money formats to keep straight:

- **`CheckoutState` amounts are integer minor units** (cents; zero-decimal
  currencies like JPY are x1). These are the authoritative numbers to render.
- **Catalog prices** (`products()`) carry `amount` as a decimal string
  (e.g. `"149.0"`) - fine for displaying an offer, but totals come from state.

Render from `onStateChange`, not from each mutator's resolved value - it also
fires for SDK-internal repricing (e.g. an address change re-quoting shipping),
so it is the one place your totals can't go stale.

An empty cart reports a `no_products` validation error; treat it as the
normal starting state rather than something to alarm the buyer with.

## Advanced example - custom summary and order bump

This expands the basic form with a custom summary and optional order bump. Use
it only when the page needs those controls; the shorter snippet above is enough
for a basic single-product checkout.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="cf-page-token" content="cfp_aB3xY7zQ9pK2mNvT8r5JcDef">
</head>
<body>
  <div id="summary" aria-live="polite" aria-atomic="true"></div>
  <div id="checkout-errors" role="alert" aria-live="assertive" aria-atomic="true" hidden></div>
  <div id="bump"></div>

  <form aria-labelledby="checkout-heading">
    <h2 id="checkout-heading">Checkout</h2>
    <label for="email">Email <span aria-hidden="true">*</span></label>
    <input id="email" name="email" data-cf-element="email" type="email" autocomplete="email" required>

    <div id="shipping-section">
      <h3 id="shipping-heading">Shipping address</h3>
      <div data-cf-element="shipping" aria-labelledby="shipping-heading"></div>
    </div>
    <h3 id="payment-heading">Payment details</h3>
    <div data-cf-element="payment-method" aria-labelledby="payment-heading"></div>
    <button data-cf-element="pay" type="submit">Pay now</button>
  </form>

  <script src="https://sdk.myclickfunnels.com/sdk.js" defer></script>
  <script>
    const ZERO_DECIMAL = new Set(['BIF','CLP','DJF','GNF','JPY','KMF','KRW','MGA','PYG','RWF','VND','VUV','XAF','XOF','XPF'])
    function money(minor, currency) {
      if (currency == null) return '-'
      const amount = ZERO_DECIMAL.has(currency.toUpperCase()) ? minor : minor / 100
      return new Intl.NumberFormat(undefined, { style: 'currency', currency }).format(amount)
    }

    document.addEventListener('DOMContentLoaded', () => {
      CfSdk.checkout.onStateChange((state) => {
        const errs = state.errors.filter((e) => e.code !== 'no_products')
        const errorBox = document.getElementById('checkout-errors')
        errorBox.hidden = errs.length === 0
        errorBox.textContent = errs.map((e) => e.message).join(' ')

        const rows = state.line_items.map((li) => {
          const row = document.createElement('p')
          row.textContent = `${li.name} - ${money(li.line_amount, state.totals.currency)}`
          return row
        })
        const total = document.createElement('strong')
        total.textContent = `Total: ${money(state.totals.total, state.totals.currency)}`
        document.getElementById('summary').replaceChildren(...rows, total)
      })

      CfSdk.products().then(({ products, bumpProducts }) => {
        const variant = products[0]?.variants[0]
        const price = variant?.prices[0]
        if (price) CfSdk.checkout.setQuantity(variant.id, price.id, 1)
        renderBump(bumpProducts[0])
      })

      function renderBump(bump) {
        const variant = bump?.variants[0]
        const price = variant?.prices[0]
        if (!price) return
        const checkbox = document.createElement('input')
        checkbox.type = 'checkbox'
        checkbox.addEventListener('change', () => {
          CfSdk.checkout.setQuantity(variant.id, price.id, checkbox.checked ? 1 : 0)
        })
        const label = document.createElement('label')
        label.append(
          checkbox,
          ` ${bump.bump_preheadline ?? 'One-time offer'}: add ${bump.name} - ${price.amount} ${price.currency ?? ''}`,
        )
        document.getElementById('bump').replaceChildren(label)
      }
    })
  </script>
</body>
</html>
```

An order bump is not a special API: it's a catalog product served in
`bumpProducts`, and selecting it is one more `setQuantity` line. The server
prices it into totals - no client-side math.

Note the bump checkbox lives **outside** the `<form>` - keep any control of
your own (bump checkboxes, quantity steppers, anything not in the element
registry) out of the SDK form. A non-SDK field inside it draws a console
warning and is dropped server-side.

If the cart needs shipping, the SDK fills the `shipping` container with the
gateway's address UI (or its own country-aware fields), and any shipping
options the server quotes appear in `state.shipping.options` (minor-unit
`amount`, nullable `name`). The cheapest is selected automatically, so a
rate picker is optional. To offer the choice, add a container:

```html
<div id="shipping-options" role="radiogroup" aria-label="Shipping speed" hidden></div>
```

and render it from the same `onStateChange` handler (reusing `money()` from
the example above):

```js
CfSdk.checkout.onStateChange((state) => {
  const box = document.getElementById('shipping-options')
  const options = state.shipping.options
  box.hidden = options.length === 0   // empty until an address is known
  const radios = options.map((opt) => {
    const input = document.createElement('input')
    input.type = 'radio'
    input.name = 'shipping-option'
    input.value = opt.id
    input.checked = opt.id === state.shipping.selected_option_id
    input.addEventListener('change', () => CfSdk.checkout.setShipping(input.value))
    const label = document.createElement('label')
    label.append(input, ` ${opt.name || 'Shipping'} - ${money(opt.amount, state.totals.currency)}`)
    return label
  })
  box.replaceChildren(...radios)
})
```

Re-render on every commit rather than once: the options (and the selected
id) change whenever the address changes, and `selected_option_id` keeps the
right radio checked across re-prices.

## One-time offers (OTO pages)

An OTO page is the funnel step **after** the checkout: the buyer's card was
saved on the previous purchase, so accepting is one click - no email, no card
entry, no `<form>`. Register it as its own external page, attach the offered
product to its step, and give it the two buttons plus a `shipping` element
(kept hidden by the SDK unless the offered product needs a shipping
address - see "Physical-product OTOs" below):

```html
<div id="offer" role="status" aria-live="polite">Loading offer...</div>
<section id="oto-shipping" aria-labelledby="oto-shipping-heading">
  <h3 id="oto-shipping-heading">Where should we ship it?</h3>
  <div data-cf-element="shipping" aria-labelledby="oto-shipping-heading"></div>
</section>
<button data-cf-element="oto-accept" value="">Yes - add this to my order</button>
<button data-cf-element="oto-decline">No thanks, continue</button>

<script>
  document.addEventListener('DOMContentLoaded', () => {
    CfSdk.products().then(({ products }) => {
      const product = products[0]
      const variant = product?.variants[0]
      const price = variant?.prices[0]
      if (!price) return
      document.getElementById('offer').textContent =
        `${product.name} - ${price.amount} ${price.currency ?? ''}`
      document.querySelector('[data-cf-element="oto-accept"]').value = `${variant.id}|${price.id}`
    })

    CfSdk.checkout.onOtoOutcome((outcome) => {
      if (outcome.status === 'unavailable') {
        // No usable OTO session (direct visit, expired or already-used offer):
        // nothing was charged. Send the visitor to your order form.
        window.location.assign('/')
        return
      }
      // status === 'failed': a real charge attempt failed.
      alert(outcome.message)
      if (outcome.recoverable) {
        // Offer a fresh-card retry: reveal a hidden <form> containing email,
        // payment-method, and pay for the SAME (variant, price) - inject the
        // payment-method div only at reveal time so the SDK doesn't price an
        // empty selection while the page is in its one-click state, and
        // setQuantity the offered line as you reveal it.
      }
    })
  })
</script>
```

How it works and what to expect:

- The offered line rides the accept button's standard `value` attribute as
  `"variantId|priceId"`. Clicking it charges the saved payment method and the
  browser follows the redirect to the next step; `oto-decline` advances
  without charging. As with `pay`, success is a redirect, not an event.
- The OTO elements are **intentionally formless** - an OTO never collects an
  email or card. (They're the exception to the "submission elements need a
  `<form>`" rule.) A physical offered product still needs a shipping address,
  which the standalone `shipping` element collects - see the next section.
- The one-click grant only exists when the visitor **arrives via the previous
  step's successful checkout redirect**. Loading the OTO page directly fires
  `onOtoOutcome` with `status: "unavailable"` - always handle it (redirect to
  your order form), or direct visitors are stranded on a dead offer.
- **Developing the page?** While the funnel is in test mode, loading the OTO
  page directly just works: the offer prices and renders normally, the
  "unavailable" outcome is suppressed (so your redirect handler stays quiet),
  and accept/decline are inert. The redirect-on-direct-load behavior above
  applies once the funnel is live, so keep your redirect handling
  unconditional.
- Every OTO page needs `oto-decline` or a fallback checkout form
  (`payment-method` + `pay`); the SDK warns in the console otherwise, because
  a buyer whose saved-card charge fails would have no way forward.

### Physical-product OTOs (shipping)

A physical offered product must ship somewhere, and the buyer may have no
address on file (a digital course followed by a t-shirt upsell is the classic
case: the order form never asked for one, so the one-click charge has nowhere
to ship). **Always include the `shipping` element on an OTO page** - the SDK
decides whether to show it. It prices the offered product on load and shows
the address form only when the offer requires an address the initial
checkout didn't collect - a digital offer, or a buyer who already entered
their address, keeps it hidden. Because showing it is driven by the product rather than the
markup, changing the offered product to a physical one later in the funnel
settings needs no page change - the same markup just works.

The element is declarative - the same `shipping` element the checkout form
uses, next to the buttons, still no `<form>`, still no JS. Wrap it in a
section using the same CSS-visibility pattern as the checkout form (the SDK
toggles the `hidden` attribute):

```html
<section id="oto-shipping" aria-labelledby="oto-shipping-heading">
  <h3 id="oto-shipping-heading">Where should we ship it?</h3>
  <div data-cf-element="shipping" aria-labelledby="oto-shipping-heading"></div>
</section>
<button data-cf-element="oto-accept" value="">Yes - add this to my order</button>
<button data-cf-element="oto-decline">No thanks, continue</button>
```

What the SDK does with it:

- The SDK prices the offer on load: `currentState()` reflects its line
  items, totals, and `address_required` - you never call `setQuantity`.
- If an address is already known (entered at checkout, or saved on the
  contact), the SDK reuses it: the form comes prefilled and shipping options
  are quoted against it on load - the buyer never retypes an address.
- Shipping options land in `CheckoutState` with the **cheapest selected
  automatically** - on load when an address is known, otherwise once the
  buyer completes the form.
- To let the buyer pick a rate instead, use the same rate-picker snippet as
  the checkout form (see "Complete example") - entirely optional, and it
  works even while the address form is hidden.
- If the initial checkout collected a shipping address, the form stays
  hidden and the one-click charge ships to it. A contact's saved default
  prefills the form but keeps it visible, so the buyer can correct a stale
  address. A completed form (shown via `data-cf-visible`) takes precedence.
- Accepting a physical offer with no address available fails **recoverably**
  through `onOtoOutcome` (the buyer can complete the address and click
  again) - but don't rely on that as the flow; include the element so the
  address is collected up front.
- Adding `data-cf-visible` to the element keeps the address form visible
  even for a digital offer (see "The checkout form" notes).
- Copy tip: "charged to the card you just used" stays true, but don't promise
  "nothing to enter" on a physical offer - the buyer may be typing an
  address.

## Styling the payment fields

Your CSS can't reach inside the gateway's payment iframes (a PCI boundary),
so the SDK forwards a set of `--cfsdk-checkout-*` CSS custom properties -
define them once on `:root` and both sides of the iframe match. How much of
the set a gateway reads varies: Stripe's Payment Element honors all of it,
while Payments AI honors only the text-level tokens (see its subsection
below).

```css
:root {
  --cfsdk-checkout-color-scheme: light;       /* or dark */
  --cfsdk-checkout-color-primary: #2b6cb0;    /* focus accent / CTA */
  --cfsdk-checkout-color-text: #1a202c;
  --cfsdk-checkout-color-danger: #c53030;
  --cfsdk-checkout-color-placeholder: #a0aec0;
  --cfsdk-checkout-color-background: #ffffff; /* input background */
  --cfsdk-checkout-color-border: #e2e8f0;
  --cfsdk-checkout-input-shadow: 0 1px 1px rgb(0 0 0 / .03);
  --cfsdk-checkout-border-radius: 6px;
  --cfsdk-checkout-font-family: ui-sans-serif, system-ui, sans-serif;
  --cfsdk-checkout-font-size: 15px;
  --cfsdk-checkout-spacing-unit: 4px;
}
```

Everything the SDK renders **in your document** ships with **no stylesheet**
of its own - it inherits your page styles directly, with class hooks when you
want to target it: `.cf-sdk-address-*` (address fields), `.cf-sdk-card-*`
(card-field boxes on Payments AI - next section), and `.cf-places-*` (address
autocomplete). Your plain `input, select, label` rules already cover most of
it.

### Payments AI (Rebilly FramePay) card fields

On Payments AI, the iframe contains only the input text, so this gateway reads
just the text-level custom properties: `color-text`,
`color-background`, `font-family`, `font-size`, `color-placeholder`, and
`color-primary` / `color-danger` as the focus / invalid **text** color.
`--cfsdk-checkout-color-border`, `--cfsdk-checkout-border-radius`,
`--cfsdk-checkout-input-shadow`, and `color-scheme` are ignored.

The supported host-page hooks are `.cf-sdk-card-field`,
`.cf-sdk-card-number`, `.cf-sdk-card-expiration`, and `.cf-sdk-card-cvv`.
Use them for layout and outer treatment only. Their descendants belong to the
payment provider; do not query, replace, or style provider class names.

```css
[data-cf-element="payment-method"] {
  display: grid;
  gap: 12px;
}

.cf-sdk-card-field {
  min-width: 0;
}

.cf-sdk-card-field:focus-within {
  outline: 2px solid #2b6cb0;
  outline-offset: 1px;
}
```

Stripe draws the full Payment Element inside its iframe, so style it with the
custom properties rather than these Payments AI-only wrapper hooks.

### The address autocomplete dropdown

When Google Places autocomplete is active on the street-address field, style
it through the supported `.cf-places-*` hooks below. Do not depend on the
generated element nesting.

- **Positioning is yours, and not optional**: `.cf-places-dropdown` ships
  with no positioning at all. Left unstyled it renders in-flow and pushes the
  rest of the page down every time suggestions appear - cosmetic styles alone
  (colors, radius) are not enough. The SDK prepares the anchor - the
  `.cf-sdk-address-field--address` wrapper gets `position: relative` if it
  wasn't already positioned - and your CSS floats the dropdown off it.
- `.cf-places-dropdown--visible` is added (alongside removing the `hidden`
  attribute) while suggestions are showing - the hook for entrance
  transitions.
- `.cf-places-item--active` marks the highlighted suggestion (arrow keys or
  pointer hover) - style it like a hover state.
- `.cf-places-item-main` and `.cf-places-item-secondary` are a suggestion's
  two text lines (street vs. locality; the second only appears when Google
  returns one). `.cf-places-highlight` is a `<strong>` wrapping each
  substring that matched the buyer's typing.
- `.cf-places-status` is a visually hidden live region for screen readers -
  leave it unstyled.

A working baseline:

```css
.cf-places-dropdown {
  position: absolute;   /* float over the page instead of pushing it down */
  top: 100%;
  left: 0;
  right: 0;
  z-index: 10;
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  box-shadow: 0 4px 12px rgb(0 0 0 / 0.1);
  max-height: 16rem;
  overflow-y: auto;
}
.cf-places-item { padding: 8px 12px; cursor: pointer; }
.cf-places-item--active { background: #eef2f7; }
.cf-places-item-main { display: block; }
.cf-places-item-secondary { display: block; font-size: 0.875em; color: #64748b; }
```

## Accessibility responsibilities (WCAG 2.2 AA)

The SDK guarantees the semantics of the same-document checkout UI it creates:

- SDK address controls have labels, unique IDs, autocomplete tokens, native
  required state, and field-specific error associations.
- Country and checkout loading states use `aria-busy` and live status regions;
  SDK errors are announced, and field errors use `aria-invalid`,
  `aria-describedby`, and `aria-errormessage`.
- Google Places suggestions use the WAI-ARIA 1.2 editable-combobox pattern,
  including keyboard navigation and click/up-event pointer activation.
- The test-mode debug badge exposes its expanded state, controls a named
  dialog, moves focus into the panel, and restores focus when it closes.

The host page is still responsible for document language, landmarks and
heading order, labels and required indicators on host-authored fields, visible
error rendering from `onError`, keyboard-visible focus styles, contrast,
reflow and zoom, forced-colors support, and pointer target sizes. The SDK does
not validate the contrast of `--cfsdk-checkout-*` colors.

Stripe Payment Element and Rebilly FramePay render vendor-owned cross-origin
iframes. Test the rendered gateway version with the keyboard and screen readers
you support. Verify iframe names, validation errors, focus movement, 200% and
400% zoom/reflow, normal contrast, and forced-colors/high-contrast mode. Those
vendor-owned internals cannot be certified from the SDK source alone.

## Testing and troubleshooting

Test the funnel as a buyer would: start on the order form, pay with a test
card, land on the OTO, accept or decline, reach the thank-you page. Don't
deep-link into the OTO page and expect the one-click to work (see above).

While the funnel is in test mode, the SDK shows a **debug overlay badge** on
the page. Open it - it shows the token, init status, the page URL vs. the
expected URL, and the failure reason. It answers most "why isn't this
working" questions directly. Beyond that:

- **Nothing loads, or the card fields never mount, on a page served with a
  Content Security Policy**: a checkout page needs the three SDK directives
  from the
  [Create an External Page skill](/.well-known/sdk/create-external-page/skill.md#content-security-policy)
  (`script-src`, `connect-src`, `form-action`) **plus** the payment provider's
  own assets - the card fields are provider iframes, loaded from
  `https://js.stripe.com` or `https://framepay.payments.ai` depending on the
  gateway, so those hosts need `script-src` and `frame-src` (FramePay also
  loads a stylesheet, so `style-src`). Each provider documents its full CSP
  requirements; start there if a field renders blank.
- **`403 url_mismatch` on init**: the browsed URL must match the registered
  page URL exactly, path included - `/oto` and `/oto.html` are different
  pages. Fix either the registered URL or the link.
- **A fix doesn't seem to take**: init responses (including errors) are
  browser-cached for ~2 minutes. Hard-refresh after changing the
  registration.
- **`products(): no checkout configuration for this page`**: init failed
  (see the overlay), no products are attached to this page's step, the
  funnel has no payment gateway, or checkout isn't enabled for the
  workspace.
- **No bump appears**: the product is attached as a main product instead of a
  bump. Check `show_page_step.products[]` on `GET /api/v2/pages/:page_id` - the
  entry's `bump` must be `true`. Send the product in
  `show_page_step.bumps` (it promotes an already-attached product), or move it
  into the step's **Order bump** section in the funnel builder.
- **No address fields appear**: expected for digital-only carts; in a
  checkout form they mount only when the cart requires shipping. (Same on an
  OTO page - the shipping element shows only when the offered product needs
  one; see "Physical-product OTOs".) If you ARE selling physical goods,
  check the variant's `product_type` - it must be `physical` (unset behaves
  like digital).
- **A one-click OTO for a physical product fails with a shipping-address
  error**: the buyer has no address on file (a digital-only main offer never
  collects one) and the OTO page is missing its `shipping` element, so there
  is nowhere to enter one. Add `<div data-cf-element="shipping"></div>` next
  to the OTO buttons - see "Physical-product OTOs".
- **`FramePay tokenization failed` (or similar) in the console, with no
  readable detail**: the console message itself does not carry the
  underlying reason - always render `state.errors` (from `onStateChange` /
  `onError`) into a visible element on the page, as in the example above;
  the real reason lands there instead. A common cause: the payment
  processor requires the buyer's first and last name to tokenize a card, so
  a form that only collects `email` fails until `first-name` / `last-name`
  fields are added.
