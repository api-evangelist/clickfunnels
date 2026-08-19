---
name: clickfunnels-refine-filters
description: >
  Generate contact filters from plain English, define and manage reusable
  contact filters for conditional splits and broadcasts, and apply existing
  order-filter tokens to the orders list. Use this skill when a CF2 surface
  takes a `filter_id` or when listing orders through a filter already built in
  ClickFunnels. Reading and one-time generation require Contacts read access;
  persisting or modifying a contact filter requires Contacts write access.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/funnels/skill.md
    description: Funnel workflow APIs (split tests, conditional splits) - primary consumer of stored filters.
  - path: /.well-known/pages/skill.md
    description: Companion skill for building/editing pages programmatically via the Page Markup API.
  - path: /.well-known/page-markup/skill.md
    description: PML reference - the DSL used to author the `markup` field on pages and blog posts.
---

# ClickFunnels Refine Filters Skill

Build, store, and reuse audience filters that any CF2 surface accepting a
`filter_id` can attach. The two main consumers are:

- **Conditional Split Steps** - branch contacts down a workflow path based on a stored filter.
- **Email Broadcasts** - scope which contacts a broadcast is sent to.

Other surfaces (segments, workflow branches, store upsells, shipping zones, etc.)
also accept stored filter ids when applicable.

## When to use this skill

- Creating a reusable filter once and attaching it to multiple workflow
  branches or conditional splits
- Routing contacts down a conditional split based on tag membership, opt-in
  history, or other allow-listed contact attributes
- Scoping an email broadcast audience without re-listing recipients on every
  send
- Listing orders through a filter already built or saved in the ClickFunnels
  Orders UI
- Updating the audience for an already-attached filter without re-touching the
  consumer (the broadcast or split will continue to use the same `filter_id`,
  but the criteria are now different)

A filter created via this API is just a `Refine::StoredFilter` row scoped to a
workspace - it is **shared across every consumer** that takes a `filter_id` in
that same workspace.

## Authentication

OAuth2 password grant using a trusted OAuth application's credentials.

All endpoints require an authorized token. Category-scoped tokens need Contacts
read access for reads and one-time generation; writes, including generation with
`save: true`, need Contacts write access. **Creating, updating, or deleting a
filter requires trusted platform access only when a third-party platform acts on
workspaces owned by other teams**. When you act on your own account (your own API
key, or an OAuth app within its own team's workspaces), writes always pass.

```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
client_id=<app_uid>
client_secret=<app_secret>
username=<app_uid>
password=<app_secret>
scope=read write delete
```

Store the `access_token` from the response. It is long-lived.

## Endpoints

| Action  | Method | Path                                                        |
|---------|--------|-------------------------------------------------------------|
| List    | GET    | `/api/v2/workspaces/:workspace_id/refine_filters`           |
| Show    | GET    | `/api/v2/refine_filters/:id`                                |
| Create  | POST   | `/api/v2/workspaces/:workspace_id/refine_filters`           |
| Update  | PATCH  | `/api/v2/refine_filters/:id`                                |
| Delete  | DELETE | `/api/v2/refine_filters/:id`                                |
| Generate from text | POST | `/api/v2/workspaces/:workspace_id/contacts/filters`   |
| Show generated filter | GET | `/api/v2/contacts/filters/:id`                      |

The manual CRUD endpoints use the wrapped `refine_filter` body documented
below. The natural-language generation endpoint uses a separate flat body.

List has two modes, and `after` is the switch. Omit it and you get the original
full-registry response: every saved filter in one payload, no `Pagination-Next`
header. Send it and you get bounded pages of 20.

On the first paged request you have no cursor yet, so send `after=0`. It is the
opt-in itself rather than a row id (real ids start at 1), and it means "start at
the beginning" in whichever direction `sort_order` asks for, so
`after=0&sort_order=desc` walks the registry newest-first. From there, send each
`Pagination-Next` value back as the next `after` until that response header is
absent, which is how you know you read the last page.

List also supports `filter[name]=...` for an exact-match lookup, so a saved
filter can be resolved by name instead of id. A legacy row whose stored state can
no longer be decoded is returned with `filter_class: null` and empty `criteria`
instead of failing the list; it can still be deleted.

The list returns **every** filter class saved in the workspace, not only the
ones this API authors. Alongside `ContactsFilter` and `OrdersFilter` you will
see classes saved by other ClickFunnels surfaces - `ContactsSegmentsFilter`,
`ProductsFilter`, `FunnelsFilter`, `ContactUpsellsFilter` and others. Treat
`filter_class` as an open set: select the class you want rather than assuming
the rest are absent.

## Generate a contact filter from plain English

Use the generation endpoint when the audience is easier to describe than to
construct manually, or when it needs a condition outside the manual API's safe
whitelist. It selects from the full ContactsFilter catalog, while still
validating condition names, clauses, structure, and workspace-owned ids.

The request body is flat (there is no `refine_filter` wrapper):

```http
POST /api/v2/workspaces/5/contacts/filters
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "text": "contacts tagged VIP who purchased the Cool Shirt",
  "save": false
}
```

- `text` is required.
- `save` defaults to `false`. The endpoint performs a read-grade computation,
  persists nothing, and returns a one-time `stable_id` token.
- `save: true` persists the filter and requires Contacts write access. `name` is
  optional and only applies when the filter is saved.

An unsaved response has `null` identity fields because there is no database
record to fetch later:

```json
{
  "id": null,
  "public_id": null,
  "workspace_id": null,
  "name": null,
  "stable_id": "H4sI...",
  "filter": {
    "conjunction": "and",
    "criteria": [
      {"attribute": "tags.id", "clause": "in", "value": ["12"]}
    ]
  }
}
```

Pass `stable_id` as the `stable_id` query parameter on the contacts list. Treat
it as opaque and let the HTTP client encode the query parameter. With curl, use
`-G --data-urlencode "stable_id=$STABLE_ID"`; do not paste the token directly
into a URL or decode it first.

A `stable_id` copied from the URL of a filtered Contacts page also works, even
when the UI wrapped the criteria in a group, as long as every conjunction in
the token is the same word (all `and` or all `or`). Tokens mixing `and`/`or`
return 422, grouped or not - a flat filter cannot express the mix, so it is
refused rather than resolved to one of the two words.

When `save: true`, the response also includes `id`, `public_id`, `workspace_id`,
and `name`. That saved result can be fetched from
`GET /api/v2/contacts/filters/:id` and managed through the regular
`refine_filters` CRUD endpoints.

## Safe condition whitelist

The public RefineFilter API accepts a deliberately small set of conditions and
clauses. The underlying Refine engine supports many more, but a number of those
either bypass indexes (text `contains`, regex), rely on cross-tenant tables
(segments, custom attributes), or scan event tables open-ended (broadcast or
opt-in conditions without a scoping id). To keep things safe for partners
running at scale, every public-API request is checked against the table below
before the criteria are persisted.

Requests that violate the policy return **422 Unprocessable Entity** with every
violation listed in the message and a pointer to the Developer Community for
new-condition requests:
[https://developers.myclickfunnels.com/page/code-support](https://developers.myclickfunnels.com/page/code-support).

If you need filter conditions that fall outside this whitelist, request them
via the [Developer Community](https://developers.myclickfunnels.com/page/code-support) -
the team adds approved conditions to the policy directly.

| Condition                                                         | Allowed clauses                                                                                                                  | Notes                                                              |
|-------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| `email_address`, `first_name`, `last_name`                        | `eq`, `sw`                                                                                                                       | **Text - equals & starts_with only.** No `contains`/regex/`dcont`. |
| `anonymous`                                                       | `eq`, `dne`                                                                                                                      | Boolean-style option.                                              |
| `created_at`, `unsubscribed_at`, `last_activity`                  | All clauses supported by the underlying date condition (`eq`, `gte`, `lte`, `btwn`, `nbtwn`, `st`, `nst`, etc.)                 | Indexed datetime columns.                                          |
| `tags.id`                                                         | `eq`, `dne`, `in`, `nin`                                                                                                         | Tag membership - must reference workspace-owned tag ids.           |
| `has_affiliate_attribution`, `has_active_affiliate_attribution`, `referred_by_affiliate` | All standard clauses for the underlying condition.                                                                  | Affiliate attribution lookups.                                     |
| `email_suppression`                                               | `st`, `nst`, `eq`, `in`                                                                                                          | Email suppression reason set/not-set or membership.                |
| `received_broadcast`, `opened_broadcast`, `clicked_broadcast`     | `eq`, `in`                                                                                                                       | **Must include a specific broadcast id** - no open-ended scans.    |
| `opted_in_funnel_step`, `opted_in_funnel_at`, `opted_in_on_standalone_page` | `eq`, `in`                                                                                                             | **Must include a specific funnel / step / page id** - no open-ended scans. |

**Conjunction:** `and` only. `or` is rejected. Multi-criterion filters are
joined with AND across the board.

**Implicitly rejected:** text `contains`/`dcont`/regex on name/email; segment
membership; viewed funnel/product; any `events.product_id`-style condition;
custom contact attributes; community membership/topic conditions;
product/variant/price/course conditions; bounced/unsubscribed/did-not-open
broadcast conditions; and any open-ended broadcast/opt-in scan that omits its
scoping id.

If the policy is too narrow for your use case, request the missing condition
in the [Developer Community](https://developers.myclickfunnels.com/page/code-support).

## Request body shape

```json
{
  "refine_filter": {
    "name": "VIP newsletter audience",
    "filter_class": "ContactsFilter",
    "conjunction": "and",
    "criteria": [
      { "attribute": "tags.id",     "clause": "in", "value": ["tag-pub-id-1", "tag-pub-id-2"] },
      { "attribute": "created_at",  "clause": "gte", "value": "2026-01-01T00:00:00Z" }
    ]
  }
}
```

- `name` is optional. When set, it must be unique within the workspace.
- `filter_class` defaults to `ContactsFilter` (the only class supported in v1).
- `conjunction` is fixed to `"and"` on the public API. Mixed/nested grouping is
  not supported.
- `criteria` must be a non-empty array. Each entry is `{attribute, clause, value}`.

## Response shape

```json
{
  "id": 42,
  "public_id": "AbCdEf",
  "workspace_id": 5,
  "name": "VIP newsletter audience",
  "filter_class": "ContactsFilter",
  "conjunction": "and",
  "criteria": [
    { "attribute": "tags.id",    "clause": "in",  "value": ["tag-pub-id-1", "tag-pub-id-2"] },
    { "attribute": "created_at", "clause": "gte", "value": "2026-01-01" }
  ],
  "created_at": "2026-04-01T12:00:00.000Z",
  "updated_at": "2026-04-01T12:00:00.000Z"
}
```

`value` is normalized by condition type. Option/select values come back as
arrays (even when you sent a scalar), absolute date values are ISO dates
(`YYYY-MM-DD`), and relative date values are objects with `days` and
`modifier`.

Create, show, and update return one filter object. List returns an envelope, not
a bare array:

```json
{
  "refine_filters": [
    {
      "id": 42,
      "public_id": "AbCdEf",
      "workspace_id": 5,
      "name": "VIP newsletter audience",
      "filter_class": "ContactsFilter",
      "conjunction": "and",
      "criteria": [
        {"attribute": "tags.id", "clause": "in", "value": ["12"]}
      ],
      "created_at": "2026-04-01T12:00:00.000Z",
      "updated_at": "2026-04-01T12:00:00.000Z"
    }
  ]
}
```

## Validation

- Unknown `attribute` -> 422
- Attribute or clause outside the safe whitelist (gate off) -> 422 with policy message
- Unknown or unsupported `clause` for the attribute -> 422
- `value` referencing IDs not in the current workspace (tags, products,
  variants, courses) -> 422
- Unparseable `value` for date attributes -> 422
- `filter_class` other than `ContactsFilter` -> 422
- `conjunction: "or"` (gate off) -> 422

## Clause cheatsheet

Refine uses short codes for clauses. The most common ones:

| Code   | Meaning                                |
|--------|----------------------------------------|
| `eq`   | equals (single value)                   |
| `dne`  | does not equal                          |
| `in`   | in (multi-value)                        |
| `nin`  | not in (multi-value)                    |
| `sw`   | starts with                             |
| `st`   | is set (column has a value)             |
| `nst`  | is not set                              |
| `gt`   | greater than                            |
| `gte`  | greater than or equal                   |
| `lt`   | less than                               |
| `lte`  | less than or equal                      |
| `btwn` | between (date range; value is `[d1, d2]`) |
| `nbtwn`| not between                             |
| `exct` | exactly (relative date with `days`)     |

For a date attribute using `gt`, `lt`, or `exct`, send `value` as
`{"days":"30","modifier":"ago"}`. `modifier` must be `ago` or
`from_now`. Absolute date clauses such as `gte` and `lte` continue to take an
ISO 8601 string.

## Worked examples

### 1. Contact has the "VIP" tag

```http
POST /api/v2/workspaces/5/refine_filters
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "refine_filter": {
    "name": "VIPs",
    "criteria": [
      {"attribute": "tags.id", "clause": "in", "value": ["12"]}
    ]
  }
}
```

### 2. Contact's email starts with `vip+`

```http
POST /api/v2/workspaces/5/refine_filters
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "refine_filter": {
    "name": "VIP-prefixed",
    "criteria": [
      {"attribute": "email_address", "clause": "sw", "value": "vip+"}
    ]
  }
}
```

### 3. Contacts created in 2026

```http
POST /api/v2/workspaces/5/refine_filters
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "refine_filter": {
    "name": "Joined in 2026",
    "criteria": [
      {"attribute": "created_at", "clause": "gte", "value": "2026-01-01T00:00:00Z"}
    ]
  }
}
```

### 4. Contacts created in the last 30 days

```http
POST /api/v2/workspaces/5/refine_filters
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "refine_filter": {
    "name": "Joined in the last 30 days",
    "criteria": [
      {
        "attribute": "created_at",
        "clause": "lt",
        "value": {"days": "30", "modifier": "ago"}
      }
    ]
  }
}
```

### 5. Contacts that received a specific broadcast

```http
POST /api/v2/workspaces/5/refine_filters
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "refine_filter": {
    "name": "Received Spring Promo",
    "criteria": [
      {"attribute": "received_broadcast", "clause": "in", "value": ["77"]}
    ]
  }
}
```

A `received_broadcast` criterion **must** include a specific broadcast id -
omitting it returns 422 (open-ended scans are not allowed).

## Applying existing order filters

The Refine Filter CRUD endpoints above create contact filters only. The orders
list can still apply an order filter that already exists in ClickFunnels:

- `stable_id` is the token from the URL of a filtered Orders page. Keep the
  token unchanged, but send it through your HTTP client's normal query-parameter
  encoding. Do not concatenate the raw token into a URL.
- `stored_filter_id` is the id or public id of a saved order filter in the same
  workspace. Use the List endpoint above and choose an entry whose
  `filter_class` is `OrdersFilter`.

Treat order-filter entries returned by List or Show as read-only in this API.
The Update endpoint authors contact filters only; edit order filters in the
ClickFunnels Orders UI.

If both are supplied, `stable_id` takes precedence.

```bash
ORDER_FILTER_TOKEN='<stable_id copied from the filtered Orders page>'

curl -skG \
  -H "Authorization: Bearer <access_token>" \
  --data-urlencode "stable_id=${ORDER_FILTER_TOKEN}" \
  "https://<workspace-subdomain>.myclickfunnels.com/api/v2/workspaces/<workspace_id>/orders"
```

```http
GET /api/v2/workspaces/{workspace_id}/orders?stored_filter_id={saved_order_filter_id}
Authorization: Bearer <access_token>
```

The API does not build a new order filter from raw criteria. Build it in the
Orders UI or use an existing saved order filter, then pass one of the selectors
above. Tokens copied from the filtered UI often wrap criteria in groups; a
grouped token is accepted as long as every conjunction in it is the same word
(all `and` or all `or` - the grouping is redundant and is flattened). A
malformed token, wrong filter type, cross-workspace id, or a token mixing
`and`/`or` returns 422 (grouped or not).

## Applying filters to consumers

A filter is only useful once attached to a consumer. The two primary surfaces:

### Applying filters to conditional split steps

Conditional splits route contacts down one of two branches based on whether
they match a stored filter. Once you have an `id` (or `public_id`) for the
filter, hand it to any conditional split:

```http
PATCH /api/v2/funnels/{funnel_id}/conditional_split_steps/{split_id}
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "conditional_split_step": {
    "condition": {"filter_id": "AbCdEf"}
  }
}
```

Read it back with the inline filter resolved:

```http
GET /api/v2/funnels/{funnel_id}/conditional_split_steps/{split_id}
GET /api/v2/funnels/{funnel_id}/conditional_split_steps?expand[]=filter
```

For the full conditional split surface (creating splits, attaching pages to
branches, positioning), see the [Funnels Skill](/.well-known/funnels/skill.md#conditional-split-steps).

### Applying filters to email broadcasts

Email broadcasts that are scoped via a stored filter accept the filter id
directly on the broadcast resource:

```http
POST /api/v2/workspaces/{workspace_id}/emails/broadcasts
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "emails_broadcast": {
    "name": "Spring Promo",
    "subject": "Spring is here",
    "from_email": "hello@example.com",
    "html_body": "<p>...</p>",
    "filter_id": 42,
    "send_immediately": false
  }
}
```

`filter_id` on a broadcast is a numeric `Refine::StoredFilter` id (not the
public id). Create the filter first via this skill, then pass its `id` field
verbatim to the broadcasts endpoint. When `filter_id` is omitted the broadcast
falls back to the workspace's full contact list.

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| 422 "criteria[0].attribute '...' is not supported by the public API" | Attribute is not in the safe whitelist | Use an allowed attribute, or request the new condition at the dev community link. |
| 422 "criteria[0].clause '...' is not allowed for attribute '...'" | Disallowed clause for a text attribute (e.g. `cont` on `email_address`) | Use `eq` or `sw`. |
| 422 "criteria[0].value must include a specific id for attribute '...'" | Open-ended broadcast/opt-in scan | Include the broadcast / funnel / step / page id you want to scope to. |
| 422 "conjunction 'or' is not supported by the public API" | `conjunction: "or"` is not supported on the public API | Use `"and"`. |
| 422 "criteria[0].attribute '...' is not a known attribute" | Misspelled attribute | Compare against the table above. |
| 422 "criteria[0].value contains ids not found in this workspace" | Tag/product/course id is from another workspace | Use ids you fetched from this workspace's `/contacts/tags`, `/products`, etc. |
| 422 "criteria[0].value '...' must be an ISO8601 date or datetime" | Date attribute received a non-ISO string | Send `YYYY-MM-DD` or full ISO8601. |
| 422 "filter_class must be 'ContactsFilter' (only supported class in v1)" | Only contact filters can be authored here, whatever classes the list returns | Omit `filter_class` to default it; edit other classes in the UI surface that owns them. |
| 422 from the orders list for `stable_id` | Token is malformed, is not an order filter, contains unsupported grouping, or references another workspace | Copy the token from the Orders UI and send it with normal query-parameter encoding. |
| 422 from the orders list for `stored_filter_id` | Saved filter is missing, belongs to another workspace, or is not an order filter | Use an order filter saved in the same workspace. |
| 422 "The request is too large to translate into a contact filter" from generation | The description implies too many separate criteria (a list of individual contacts or emails), so the model's answer runs past the request budget | Deterministic, not transient - retrying the same text fails identically. Describe the audience by attributes (tags, dates, purchases), or split it into smaller requests. |
| 503 from contact-filter generation | The AI model timed out or another temporary dependency was unavailable | Retry the same request after a short delay. |
| 200 with an empty `refine_filters` array on the first paged request | `after` carried something that is neither `0` nor a `Pagination-Next` cursor; a value that is not a real row id pages nothing | Start the walk with `after=0`, then echo each `Pagination-Next` value. |
| 401 "API key missing or invalid" | Missing, revoked, expired, or wrong token | Re-authenticate. |
| 404 "Not found" | Filter id does not exist or belongs to another workspace | Verify the id is reachable for this workspace's token. |
