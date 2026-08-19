---
name: clickfunnels-products
description: >
  Create and manage sellable products, their prices, and discounts via the public API -
  products, the implicit default variant, one-time / subscription / payment-plan prices,
  compare-at (strike-through) pricing, repricing safely once orders exist, discount codes
  (percentage or fixed amount, workspace-wide or scoped to specific products/variants),
  and attaching products to funnel pages so a checkout can sell them. Use this skill when
  setting up something to sell programmatically: a membership, a course offer, an
  order-form product, a launch discount code, or when changing an existing product's price.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, API docs.
  - path: /.well-known/pages/skill.md
    description: Attach products to a page's checkout step (show_page_step.product_ids).
  - path: /.well-known/sdk/add-checkout/skill.md
    description: Sell the attached products from an externally-hosted page via the SDK.
  - path: /.well-known/subscription-changes/skill.md
    description: Change the variant/price of an existing subscription line item.
---

# ClickFunnels Products & Discounts Skill

Products are the sellable goods of a workspace. Every product owns at least one
variant (a "default variant" is created implicitly), and every variant owns the
prices a buyer can pay: one-time, subscription, or payment plan. A funnel page
sells whatever products are attached to its step. A **discount** is a separate,
reusable object (a coupon code or an amount/percentage off) that a checkout applies
on top of a price - use it for launch codes, member perks, or a real limited-time
sale, instead of hard-coding a discounted price.

## When to use this skill

- Create a product and give it a price (one-time or recurring) via the API.
- Add a founding/discount price with a strike-through compare-at amount.
- Create a discount code (percentage or fixed amount off) buyers redeem at checkout.
- Change the price of a product that has already been sold (repricing).
- Attach or detach products on a funnel page's checkout step.
- Understand why editing a price returns 422 after the first order.

## Authentication

Requests use the ClickFunnels V2 API with a Bearer token:

```
Authorization: Bearer <access_token>
```

## Endpoint reference

| Action | Method | Path |
|---|---|---|
| List products | GET | `/api/v2/workspaces/:workspace_id/products` |
| Create product | POST | `/api/v2/workspaces/:workspace_id/products` |
| Fetch / Update product | GET/PATCH | `/api/v2/products/:id` |
| Archive / Unarchive | POST | `/api/v2/products/:id/archive` / `/unarchive` |
| List prices (of a product) | GET | `/api/v2/products/:product_id/prices?filter[variant_id]=<id>` |
| Create price (on a variant) | POST | `/api/v2/products/variants/:variant_id/prices` |
| Fetch / Update price | GET/PATCH | `/api/v2/products/prices/:id` |
| Attach products to a page step | PATCH | `/api/v2/pages/:id` with `show_page_step.product_ids` |
| Detach products from a page step | DELETE | `/api/v2/pages/:id/products` with `product_ids` |
| List discounts | GET | `/api/v2/workspaces/:workspace_id/discounts` |
| Create discount | POST | `/api/v2/workspaces/:workspace_id/discounts` |
| Fetch / Update discount | GET/PATCH | `/api/v2/discounts/:id` |
| Delete discount | DELETE | `/api/v2/discounts/:id` |

## Create a product

```http
POST /api/v2/workspaces/:workspace_id/products
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "product": {
    "name": "Acme Club - Community",
    "visible_in_store": false,
    "visible_in_customer_center": true,
    "seo_title": "Acme Club Community Membership",
    "seo_description": "A vetted community for practitioners. Community, kit, and coaching."
  }
}
```

Only `name` is required. The response includes the ids you need next:

```json
{
  "id": 1018736,
  "public_id": "jMZmdR",
  "name": "Acme Club - Community",
  "default_variant_id": 5632221,
  "archived": false,
  "visible_in_store": false,
  "visible_in_customer_center": true
}
```

- A **default variant** is created implicitly with `product_type: "digital"`. For a
  physical product, update the default variant's type in a follow-up request.
- `default_variant_id` is the variant to hang prices on for a single-variant product.
- The user-facing description lives on the default variant (not the product); the
  product carries `seo_title` / `seo_description`.

## Create a price

Prices belong to a **variant** (note the path). A monthly subscription:

```http
POST /api/v2/products/variants/5632221/prices
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "products_price": {
    "name": "Monthly",
    "amount": "97.00",
    "currency": "USD",
    "payment_type": "subscription",
    "interval": "months",
    "interval_count": "1",
    "trial_interval": "months",
    "trial_duration": "0",
    "trial_amount": "0.00",
    "visible": true
  }
}
```

```json
{
  "id": 5104590,
  "public_id": "yyGxAr",
  "variant_id": 5632221,
  "name": "Monthly",
  "amount": "97.00",
  "cost": "0.00",
  "currency": "usd",
  "duration": null,
  "interval": "months",
  "interval_count": 1,
  "trial_interval": null,
  "trial_duration": null,
  "trial_amount": "0.00",
  "payment_type": "subscription",
  "compare_at_amount": null,
  "visible": true,
  "archived": false
}
```

Things that will save you a round-trip:

- **`interval` is plural**: `months`, not `month`. The singular form is rejected with
  `422 "Interval is not included in the list"`.
- `payment_type` is one of `one_time` (default), `subscription`, `payment_plan`.
- Amounts are decimal **strings** (`"97.00"`), not numbers.
- A one-time price only needs `amount` + `currency` + `payment_type: "one_time"`
  (the interval/trial fields can be null).

### Compare-at (strike-through) pricing

`compare_at_amount` renders the "was" price a buyer sees crossed out:

```json
{
  "products_price": {
    "name": "Monthly - Founding rate",
    "amount": "97.00",
    "compare_at_amount": "897.00",
    "currency": "USD",
    "payment_type": "subscription",
    "interval": "months",
    "interval_count": "1",
    "trial_interval": "months",
    "trial_duration": "0",
    "trial_amount": "0.00",
    "visible": true
  }
}
```

The buyer is charged `amount`; `compare_at_amount` is display-only - it does not
change what anyone pays and there is no redemption logic behind it.

If you actually want a *discount* - a coupon code, a percentage/amount off, or a
real time-boxed sale rather than a permanently marked-down price - use the Discounts
API below instead of baking a second price into the product. A discount is one
reusable object the checkout applies on top of the price, so you can start, scope,
and expire it without touching the product's prices. (Enforcing a hard
limited-quantity cap like "first 10 members" is still your own process - the discount
carries scheduling and a per-customer limit, but not a global redemption cap.)

## Repricing - once orders exist

A subscription price that has been purchased is **immutable**:

```json
{ "base": ["Cannot edit a subscription price that is attached to an order."] }
```

Existing subscribers keep billing on the price they bought. To change what NEW
buyers pay:

1. **Create a new price** on the same variant with the new `amount`.
2. **Retire the old price**: `PATCH /api/v2/products/prices/:id` with
   `{"products_price": {"visible": false, "archived": true}}`.
3. If the product is attached to a funnel step, verify what the checkout now
   offers (see the add-checkout skill) - client catalogs can cache the old
   price list for a while after a swap.

```http
PATCH /api/v2/products/prices/5104591
Authorization: Bearer <access_token>
Content-Type: application/json

{ "products_price": { "visible": false, "archived": true } }
```

## Discounts

A discount is a reusable, workspace-scoped object the checkout applies on top of a
price. It is the right tool for a coupon code, a percentage/amount off, or a
time-boxed sale - anything you would otherwise fake with a second "sale" price.

**A discount reaches an order as its own record, not as a field you set.** Creating a
discount publishes the code; the order-to-discount link is a separate *applied discount*
record, written when a buyer redeems the code at checkout or when a ClickFunnels user
applies it in the admin UI. That is why `POST /api/v2/workspaces/:workspace_id/orders`
takes no `coupon_code` or `discount_id`, and why an order's `discount_ids` are
read-only: they report the applied discounts, they do not create them. Writing applied
discounts is not part of this API yet, and when it lands it will be that record you
create rather than a discount field on the order. Until then the surface here is create,
read, update, expire, delete - plus reading back which orders ended up carrying the
discount.

Create a **percentage** discount code:

```http
POST /api/v2/workspaces/:workspace_id/discounts
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "discount": {
    "name": "Launch 20",
    "code": "LAUNCH20",
    "discount_type": "code",
    "discount_method": "percentage",
    "percent": "20.0",
    "applies_to": "all_products"
  }
}
```

```json
{
  "id": 771,
  "public_id": "gBEDMo",
  "workspace_id": 5,
  "name": "Launch 20",
  "code": "LAUNCH20",
  "discount_type": "code",
  "discount_method": "percentage",
  "amount": null,
  "percent": "20.0",
  "currency": "usd",
  "applies_to": "all_products",
  "apply_from": "2026-07-14T12:00:00.000Z",
  "apply_until": null,
  "limit_1_per_customer": false,
  "require_minimum_spend": false,
  "require_minimum_spend_amount": null,
  "discounts_per_redemption": null,
  "redemptions_count": 0,
  "product_ids": [],
  "variant_ids": [],
  "products_collection_ids": [],
  "active": true,
  "expired": false,
  "scheduled": false
}
```

A **fixed-amount** discount instead sets `discount_method: "fixed"` with an `amount`:

```json
{
  "discount": {
    "name": "$10 off",
    "code": "TENOFF",
    "discount_method": "fixed",
    "amount": "10.00",
    "currency": "USD"
  }
}
```

Things that will save you a round-trip:

- **`code` is required** for a `code`-type discount, is uppercased on save, and must
  be unique within the workspace (it also cannot collide with a coupon code). A 422
  `Coupon Code is already being used in another Discount or Coupon` means the code is
  in use.
- **`code` accepts letters, numbers and hyphens only.** No spaces, underscores or other
  punctuation - `SUMMER_20` and `SUMMER 20` both return 422 `Coupon Code only allows
  letters, numbers, and hyphens`. Use `SUMMER-20`.
- **`discount_method` decides which amount field is required**: `percentage` needs
  `percent` (0.01-100), `fixed` needs a positive `amount`. The method defaults to
  `percentage`; omitting its required `percent` returns 422 `Percent must be greater
  than 0` (and omitting `amount` for `fixed` returns `Amount must be greater than 0`).
  `shipping` and `bogo` also exist but are not part of this simple flow.
- All validation errors come back as `{"error": "Request unprocessable: <message>"}`.
- **`currency`** defaults to the workspace currency; `amount` is a decimal string and
  echoes back with two decimals (`"10.00"`).
- **`apply_from`** defaults to now, so a discount is live immediately. Set a future
  `apply_from` to schedule it (`scheduled: true` in the response) and an `apply_until`
  to auto-expire it.
- **`applies_to`** is `all_products` by default. To scope it, set `applies_to` to
  `specific_products` / `specific_variants` / `specific_collections` and pass the
  matching `product_ids` / `variant_ids` / `products_collection_ids` array. The response
  echoes those same arrays back (empty for an `all_products` discount) so you can read
  the current scope.
- **`require_minimum_spend_amount` is in whole currency units, not cents.** On a `usd`
  discount, `50` means $50. It must be positive whenever `require_minimum_spend` is true.
- The response's `active` / `expired` / `scheduled` booleans are derived from
  `apply_from` / `apply_until` - read them instead of recomputing from the dates.
- **`redemptions_count` is not a deletability signal.** It counts completed sales only, so
  a discount can report `0` and still be attached to an order, which is what DELETE
  actually checks. Treat a 422 on DELETE as the answer, not the count.

### Editing and expiring a discount - once redeemed

Two lock-out rules govern the end of a discount's life, and they bite in this order.

**Once applied to an order**, a discount locks its money-affecting fields (`amount`,
`percent`, `discount_method`, `applies_to`, `apply_from`, the scoping arrays,
`limit_1_per_customer`, `discounts_per_redemption`). PATCHing one of those returns 422
`<Field> cannot be changed because it has already been applied`; renaming and expiring
still work.

**Once expired**, the discount freezes its other fields: an update to one returns 422
`Discount cannot be changed because it has expired`, a rename included. Expiring is
reversible though, so editing an expired discount is three calls: lift the expiry, change
it, expire it again. Sending `expired: true` alongside the edit does not shortcut that -
the switch does nothing on a discount that is already expired, so the edit is refused on
its own.

Expiring a discount and renaming it in the **same** call is fine. It is a rename sent to a
discount that was already expired that gets refused.

To stop offering a live discount, **expire it**. Send the virtual `expired: true` flag -
do NOT try to set `apply_until` to a past time, because `apply_until` must be after
`apply_from` (which defaults to the creation time), so a past date returns 422 "Apply
until ... must be after the scheduled start date":

```http
PATCH /api/v2/discounts/:id
Authorization: Bearer <access_token>
Content-Type: application/json

{ "discount": { "expired": true } }
```

`expired: true` sets `apply_until` to now (`active: false`, `expired: true` in the
response). `expired: false` lifts an expiry, which is what the Remove Expiration button
in the app does. Both directions are the same field, accepted on **update only**:

- `expired: false` on an expired discount un-expires it. A discount that has been
  **redeemed** cannot be un-expired: 422 `Apply until cannot be removed after it has been
  redeemed`. It only clears `apply_until` and leaves `apply_from` alone, so a discount
  whose start date is still in the future comes back `scheduled: true` rather than
  `active: true` - read the flags off the response instead of assuming.
- `expired` on create returns 422 at **any** value, `false` included - it could never
  work there, since it would set `apply_until` to now and `apply_from` then defaults to a
  hair later. Create the discount, then PATCH `expired: true`.
- Sending the state it is already in changes nothing and returns 200: `expired: false` on
  a live discount will not clear a scheduled end date, and `expired: true` on an expired
  one will not push `apply_until` forward.
- `expired: true` also ends a discount that has not started yet (`scheduled: true`), which
  discards its future `apply_from` because nothing can end before it starts. This is the
  one case where `apply_until` alone will not do it, since the end would fall before the
  start, and `apply_from` cannot be moved into the past on its own (422 `Apply from
  scheduled date cannot be changed since the date is in the past`).
- Only `true` and `false` are accepted. Anything else returns 422 `` `expired` must be
  true or false `` instead of being read as "expire it".

To *remove* a future end date without expiring anything, PATCH `apply_until: null`. Use a
**future** `apply_until` to schedule an end date in advance - or to push an expired
discount's end date out, which revives it the same way `expired: false` does.

`DELETE /api/v2/discounts/:id` hard-deletes the discount, but only one that is **not**
attached to an order - an attached discount returns 422 (expire it instead, to preserve
its order history). `redemptions_count: 0` does not mean it is deletable: that counter
only tracks completed sales, while DELETE checks every order and invoice the discount is
attached to. A discount that no order or invoice ever carried - the usual case for a code
you created and never published - is always deletable.

## Attach products to a funnel page

A checkout sells the products attached to its page's step. Attachment is
**additive** - PATCHing never removes existing products:

```http
PATCH /api/v2/pages/:page_id
Authorization: Bearer <access_token>
Content-Type: application/json

{ "page": { "show_page_step": { "product_ids": [1018736] } } }
```

Detaching is its own endpoint (the batch is transactional - one bad id and
nothing is detached):

```http
DELETE /api/v2/pages/:page_id/products
Authorization: Bearer <access_token>
Content-Type: application/json

{ "product_ids": [1018736] }
```

Read back what a step sells via `GET /api/v2/pages/:id` - the response's
`show_page_step.products` lists the attached products.

## Worked example - a paid membership, end to end

```http
# 1) product                                     -> capture id + default_variant_id
POST /api/v2/workspaces/5/products               { "product": { "name": "Acme Club - Community" } }

# 2) monthly subscription price on the default variant
POST /api/v2/products/variants/5632221/prices    { "products_price": { "name": "Monthly", "amount": "97.00", "currency": "USD", "payment_type": "subscription", "interval": "months", "interval_count": "1", "trial_interval": "months", "trial_duration": "0", "trial_amount": "0.00", "visible": true } }

# 3) attach to the checkout page's step
PATCH /api/v2/pages/24809428                     { "page": { "show_page_step": { "product_ids": [1018736] } } }

# 4) verify what the variant sells
GET /api/v2/products/1018736/prices?filter[variant_id]=5632221
```

## Common errors

| Error | Cause | Fix |
|---|---|---|
| 422 "Interval is not included in the list" | `interval: "month"` (singular) | Use the plural: `months`, `years`, ... |
| 422 "Cannot edit a subscription price that is attached to an order." | PATCHing a purchased subscription price | Create a new price, archive the old one (see Repricing). |
| 422 "A product can only be archived if it's not in the live state." | Archiving a live product | Remove it from stores/customer center/live funnels first. |
| Price list shows unexpected entries | `GET /products/:id/prices` without a variant filter | Always pass `filter[variant_id]=<id>` to scope the list. |
| Checkout still offers a retired price | Client-side catalog caching after a repricing | Allow time to propagate; verify with a fresh session (see the add-checkout skill). |
| 422 "Coupon Code is already being used in another Discount or Coupon" | Discount `code` collides with an existing discount or coupon in the workspace | Pick a unique code. |
| 422 "Percent must be greater than 0" / "Amount must be greater than 0" | `discount_method` without its amount field | `percentage` needs `percent`; `fixed` needs a positive `amount`. |
| 422 "Apply until ... must be after the scheduled start date" | Tried to expire by setting `apply_until` to a past date | Use `expired: true` to expire now; `apply_until` only schedules a future end. |
| 422 editing a discount's amount/percent | The discount has already been redeemed on an order | Money fields are locked once applied; create a new discount, expire the old one. |
| 422 "Coupon Code only allows letters, numbers, and hyphens" | Discount `code` contains a space, underscore or punctuation | Use letters, numbers and hyphens only (`SUMMER-20`). |
| 422 "Discount cannot be changed because it has expired" | A field PATCHed on a discount that is already expired, including one sent alongside `expired: true` | Lift the expiry with `expired: false`, make the change, then expire it again. |
| 422 "Apply until cannot be removed after it has been redeemed" | `expired: false` on a discount that has been redeemed on an order | A redeemed discount cannot be un-expired. Create a replacement discount. |
| 422 "`expired` must be true or false" | `expired` sent as something else (`""`, `1`, a typo) | Send a JSON boolean. Nothing else counts as "expire it". |
| 422 "`expired` is not a create field" | `expired` sent to POST /discounts at any value, `false` included | Create the discount, then PATCH `expired: true`. |
| 422 "`&lt;field&gt;` must be one of: ..." | `discount_type`, `discount_method` or `applies_to` set to a value outside its list | The message names the values that would work. |
| 422 on DELETE of a discount showing `redemptions_count: 0` | The counter tracks completed sales; DELETE checks every order and invoice the discount is attached to | Expire it instead of deleting it. |
| 404 "Not found" | Product/variant/price/discount id outside the token's scope | Verify the id and the workspace the token can reach. |
