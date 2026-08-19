---
name: clickfunnels-subscription-changes
description: >
  Manage existing subscription orders programmatically - discover allowed
  variant/price upgrade or downgrade targets for a subscription line item,
  preview the upcoming invoice impact, then commit the change. Use this skill
  any time you need to move a customer's subscription onto a different plan or
  billing cadence without cancelling and re-buying. Requires trusted platform access
  only when a third-party platform acts on another team's workspaces; acting on your own
  account does not.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/funnels/skill.md
    description: Funnel workflow APIs - where a subscription was sold from in the first place.
---

# ClickFunnels Subscription Changes Skill

A self-serve flow for switching the variant or price on an existing subscription
line item. Modeled on the change-plan flows partners are used to from other
billing platforms, but with one CF2-specific constraint: legal transitions are
pre-configured by the merchant on the source variant. You can only target a
variant/price combination the merchant has declared "allowed" for the source
price.

## The three-step flow

| Step | Method | Path | Purpose |
|------|--------|------|---------|
| Discover | `GET` | `/api/v2/products/prices/:price_id/change_options` | List allowed upgrade and downgrade targets for a source price. |
| Preview | `POST` | `/api/v2/orders/line_items/:line_item_id/changes` | Non-destructive quote of the upcoming invoice for a proposed change. |
| Commit  | `POST` | `/api/v2/orders/line_items/:line_item_id/changes/perform` | Apply the change. Flips the line item's variant + price. |

All three accept either the integer database id or the obfuscated `public_id`
string in the path; the `products_price_id` body param accepts both forms too.

## Step 1 - Discover what's allowed

Pass the **current** price id of the line item you want to change. Response is
grouped by **target variant**; each group lists the prices on that variant the
subscription may switch to.

```bash
curl -X GET \
  -H "Authorization: Bearer YOUR_TOKEN" \
  "https://YOUR_WORKSPACE.myclickfunnels.com/api/v2/products/prices/66/change_options"
```

```json
{
  "products_prices_change_options": {
    "upgrade": [
      {
        "products_variant": {
          "id": 124,
          "public_id": "k7xQzR",
          "name": "Pro",
          "product_id": 88,
          "prices": [
            {
              "id": 456,
              "public_id": "aBcDeF",
              "name": "Pro Monthly",
              "key": "pro_monthly_2024",
              "amount": "20.00",
              "currency": "USD",
              "duration": null,
              "interval": "months",
              "interval_count": 1
            },
            {
              "id": 457,
              "public_id": "gHiJkL",
              "name": "Pro Annual",
              "key": "pro_annual_2024",
              "amount": "200.00",
              "currency": "USD",
              "duration": null,
              "interval": "years",
              "interval_count": 1
            }
          ]
        }
      }
    ],
    "downgrade": []
  }
}
```

Notes on the shape:
- `upgrade` and `downgrade` are siblings; the array key is the direction.
- Groups are keyed by `products_variant` because multiple billing options on a
  single target variant are common (monthly + annual on the same "Pro" tier).
- Each price carries the minimal display payload (`id`, `public_id`, `name`,
  `key`, `amount`, `currency`, `duration`, `interval`, `interval_count`) - enough
  to render a plan selector without a second fetch.
- When no options are configured, you get `{ "upgrade": [], "downgrade": [] }`
  with status 200 - never a 404. Empty arrays let you hide the upgrade UI
  cleanly when a price has no targets.

## Step 2 - Preview the change

Pass `products_price_id` (the target) - wrapped under `orders_line_items_change`. The target variant is derived from the price's `variant` association.

```bash
curl -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"orders_line_items_change": {"products_price_id": 456}}' \
  "https://YOUR_WORKSPACE.myclickfunnels.com/api/v2/orders/line_items/204/changes"
```

```json
{
  "orders_line_items_change": {
    "preview": true,
    "upcoming_invoice": {
      "total_amount": "35.83",
      "subtotal_amount": "35.83",
      "tax_amount": "0.00",
      "shipping_amount": "0.00",
      "discount_amount": "0.00",
      "currency": "USD",
      "next_invoice_date": "2026-06-14T03:49:45Z"
    },
    "previous_line_item": { /* current line item state, pre-change */ },
    "new_line_item":      { /* proposed line item state with the new variant + price */ }
  }
}
```

The preview is a non-destructive quote - no charge happens, the line item is
not mutated. `upcoming_invoice` holds the upstream invoice fields (totals,
currency, next invoice date) so you can show the customer what they'll be
charged. `previous_line_item` reflects the **current** state and
`new_line_item` reflects what it would become after the change, so you can diff
the two directly without having to construct the comparison yourself.

Optional body params:

- `prorate` (boolean) - override the default proration behaviour for this call.
- `effective_time` (`"now"` or `"next_renewal"`) - override default timing.

Defaults if you omit them:

| Direction | Default `prorate` | Default `effective_time` |
|-----------|-------------------|--------------------------|
| Upgrade   | `true`            | `"now"`                  |
| Downgrade | `false`           | `"next_renewal"`         |

Physical line items always force `prorate=false`, regardless of what you send.

## Step 3 - Commit the change

Same body shape as preview. The response envelope is identical; only the
`preview` flag flips to `false`. `upcoming_invoice` carries the upstream
invoice quote snapshotted just before the commit was applied - so you
don't have to call preview separately to know what the customer was
charged.

```bash
curl -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"orders_line_items_change": {"products_price_id": 456, "effective_time": "now"}}' \
  "https://YOUR_WORKSPACE.myclickfunnels.com/api/v2/orders/line_items/204/changes/perform"
```

```json
{
  "orders_line_items_change": {
    "preview": false,
    "upcoming_invoice": {
      "total_amount": "35.83",
      "subtotal_amount": "35.83",
      "tax_amount": "0.00",
      "shipping_amount": "0.00",
      "discount_amount": "0.00",
      "currency": "USD",
      "next_invoice_date": "2026-06-14T03:49:45Z"
    },
    "previous_line_item": { /* line item state before the change */ },
    "new_line_item":      { /* updated state - variant + price flipped */ }
  }
}
```

On success:

- The persisted `line_item.variant_id` and `line_item.products_price_id` are now the values shown in `new_line_item`.
- The `subscription.line_items.changed` webhook fires.

If the upstream quote snapshot returns no data, the commit still succeeds -
`upcoming_invoice`'s amount fields fall back to `null` and `currency`/`next_invoice_date`
fall back to the order's stored values.

## Allowed transitions, in plain words

A change is allowed if the merchant has configured an upgrade/downgrade option
row matching `(current variant, current price -> target variant, target price)`.
If no such row exists, you get **422 `target_not_allowed`**. The merchant adds
options in the admin UI under **Product -> Variant -> Upgrade/Downgrade
options**.

If the target price you send is the **same as the current price**, the call
returns **422 `same_target`** - there's nothing to change.

## Eligibility

The line item's order must qualify for self-serve change. This means:

- The order is a subscription order.
- It has exactly **one** subscription-type line item.
- It has only `subscription` and/or `one_time` payment types - no payment plans.

If not, you get **422 `not_eligible`**. One-time orders, mixed payment-plan
orders, and orders with multiple subscription line items can't be self-served
through this API.

## Error responses

| Code | When |
|------|------|
| 401  | Missing or invalid bearer token. |
| 403  | A third-party platform previewing or committing a change on another team's workspace without trusted platform access. This trusted-access check does not apply to first-party / own-account writes (option discovery via `change_options` is a read and is never gated). |
| 404  | Resource not found, including a cross-workspace resource. |
| 422  | `invalid_effective_time`, `not_eligible`, `same_target` (target price matches the current price), `target_not_allowed`, `Invalid products_price_id` (target belongs to a different workspace on the same team), `preview_failed`, `commit_failed`. |

## Gotchas

- **Empty discovery is not 404.** When no options are configured, you get a
  200 response with empty arrays. Don't treat empty as an error.
- **Public IDs work everywhere.** Path `:price_id` / `:id` and body
  `products_price_id` both accept the obfuscated `public_id` strings - no need
  to integer-cast them client-side.
- **Posting the current price as the target returns 422.** Both preview and
  commit reject "change to the same price" as `same_target`. Skip the call
  client-side when the user hasn't picked a different price.
- **A preview without a payment method on the order fails.** The upstream
  payment platform's quote endpoint requires the order to have a billing
  payment method - wire one up first.
