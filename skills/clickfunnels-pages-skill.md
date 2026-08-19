---
name: clickfunnels-pages
description: >
  Create, read, and update ClickFunnels 2.0 pages programmatically - funnel
  pages, site pages, checkout pages, opt-ins. The page body is authored in PML
  (Page Markup Language); the DSL itself lives in the Page Markup Skill.
  Writing pages as a third-party platform on another team's workspaces requires trusted
  platform access; acting on your own account does not.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/page-markup/skill.md
    description: PML reference - elements, attributes, design system, examples. Read this before authoring any `markup` field.
  - path: /.well-known/blogs/skill.md
    description: Companion API for managing blog posts (also use PML for the body).
  - path: /.well-known/funnels/skill.md
    description: Funnel workflow APIs that wire pages into split tests and conditional splits.
---

# ClickFunnels Pages Skill

The HTTP surface for creating and updating CF2 pages. The body of every page is
authored in **PML (Page Markup Language)** - for the DSL itself (elements,
attributes, design system), see the
[Page Markup Skill](/.well-known/page-markup/skill.md). This document covers the
API surrounding that field: authentication, the page CRUD endpoints, and the
checkout-products attachment shape.

## When to use this skill

- Creating a new funnel or site page from a brief or prompt
- Reading back the current markup of an existing page (`GET ... ?expand[]=markup`)
- Replacing or extending an existing page's content programmatically
- Attaching or removing products from a checkout page
- Building multi-page funnels via the API (one page per call, then compose with the [Funnels Skill](/.well-known/funnels/skill.md))

## Authentication

OAuth2 password grant using OAuth application credentials.

> **When is trusted platform access required?** Only when a **third-party** platform
> acts on workspaces owned by **other** teams (page `markup` and head/footer code
> writes). Acting on your own account (your own API key, or an OAuth app within its own
> team's workspaces) never needs it; those writes always pass, and reads are never gated.

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

## Reading Current Markup

Before editing a page, read its current markup to understand the existing structure:

```
GET /api/v2/pages/:page_id?expand[]=markup
Authorization: Bearer <access_token>
```

Response: **200 OK** (the page resource, with the `markup` field included)
```json
{
  "id": 42,
  "name": "Landing Page",
  "markup": "<section bg=\"#0f172a\" padding=\"100 40\">...</section>"
}
```

Markup serialization is per-page work, so it is opt-in via `expand[]=markup` on
every endpoint (show, list, create, update). Returns an empty string `""` when
the page has no content yet. Always read first when the task is to modify or
extend an existing page rather than replace it.

**This read is lossy.** The serializer only has an equivalent for the elements
documented in the Page Markup Skill. Anything else the ClickFunnels editor
supports is dropped from the string with no error and no placeholder, so what
you get back can be an incomplete picture of the real page. Never treat a
round-tripped read as a safe basis for a rewrite: see
[STOP: get approval before overwriting an existing page](/.well-known/page-markup/skill.md#stop-get-approval-before-overwriting-an-existing-page).

## Add a tracking pixel or custom code

**Never use `markup` for this.** Adding a Meta/Facebook pixel, a Google tag, or
any other script or style is what `head_code`, `footer_code` and `custom_css` are
for. They are additive: `append` mode (the default) adds to whatever is already
there and touches nothing else on the page. A `markup` write, by contrast,
replaces the page's entire tree, so using it to inject a pixel means rewriting
every section from scratch and risking the whole page for a one-line change.

```
PATCH /api/v2/pages/:page_id
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "page": {
    "head_code": "<script>!function(f,b,e,v,n,t,s){...}</script>",
    "head_code_mode": "append"
  }
}
```

- `head_code` injects before `</head>` - use this for pixel base code.
- `footer_code` injects before `</body>` - use this for anything that must run
  after the page renders.
- `custom_css` is raw CSS with no `<style>` wrapper.
- Each field takes a matching `*_mode` of `append` (default) or `replace`. Prefer
  `append`. Use `replace` only when you intend to discard the existing code, and
  read the current value first (`?expand[]=head_code`) so you know what you are
  discarding. Send `null` to clear a field.
- Internal pages only. These fields return 422 on external (SDK) pages - on those,
  the pixel belongs in the HTML you host yourself.
- Trusted platforms only: a third-party platform app whose team lacks
  trusted-platform access gets a 403 on these fields. On that 403, stop and tell
  the user to apply for trusted platform access - do NOT fall back to `markup`
  (it is gated by the same check, and rewriting the page is never the answer to
  a permissions error).

If someone asks you to "add a pixel to the page", this section is the whole job.
Do not read the markup, do not rewrite the body, and do not treat a pixel request
as permission to touch the page's content.

## Building a Page

Markup is validated as part of building. There is no separate validate endpoint:
the build call is validate-then-apply atomically, and an invalid markup returns
a 400 response without persisting any state. For the parser error shapes, see
[Page Markup Skill -> Error Reference](/.well-known/page-markup/skill.md#18-error-reference).

### Apply markup to an existing page

**STOP before you send this.** A `markup` write replaces the page's entire tree,
published version and editor draft alike, and PML covers only a subset of what
the ClickFunnels editor can build. If this page could have been created or edited
in the editor, ask the person for explicit approval first and tell them
editor-only content will not survive. "Update the page" is not that approval.
The full rule, including the cases where you do not need to ask, is in
[STOP: get approval before overwriting an existing page](/.well-known/page-markup/skill.md#stop-get-approval-before-overwriting-an-existing-page).

```
PATCH /api/v2/pages/:page_id
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "page": {
    "markup": "<section bg=\"#0f172a\" padding=\"100 40\">...</section>"
  }
}
```

On success: **200 OK** with the updated page resource. Pass `?expand[]=markup`
to also receive the round-tripped PML in the response.

### Create a new page with markup

```
POST /api/v2/workspaces/:workspace_id/pages
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "page": {
    "name": "Landing Page",
    "markup": "<section bg=\"#0f172a\" padding=\"100 40\">...</section>"
  }
}
```

On success: **201 Created** with the new page resource. The build is transactional
- if the markup is invalid, no page is created and you get a 400 response.

On invalid markup (either endpoint): **400 Bad Request**
```json
{
  "error": "Bad request: Markup is not valid PML: unknown top-level element(s) <bad-tag>. ... (line 3)"
}
```

### Setting a page's path (`current_path`)

`current_path` is the page's **own** path/slug within its site or funnel (e.g.
`"/order"`, `"/middle"`). It is optional on create - omit it and ClickFunnels
derives one.

! **FORBIDDEN - never build a nested `current_path`.** A page's `current_path`
must be its **own standalone path/slug only** - never a parent path (the funnel's
path, a site path, etc.) concatenated with the page's own slug. When a page is a
funnel step, ClickFunnels composes the full, nested public URL automatically from
the funnel/step hierarchy; the caller must **never** guess, prefix, or build that
composed URL itself when setting `current_path`.

Real-world example - funnel path `mangia-mangia-catering`, with a funnel step of
its own:

- **Wrong** (funnel path prefixed onto the step's slug) -> `"current_path": "mangia-mangia-catering/catering"` -> yields the broken nested URL `https://<workspace>.myclickfunnels.com/mangia-mangia-catering/catering`
- **Right** (the funnel step's own standalone path) -> `"current_path": "mangia-mangia-catering"` -> yields the correct URL `https://<workspace>.myclickfunnels.com/mangia-mangia-catering`

### Manage products on a checkout page

A `<checkout/>` element renders the products attached to its show_page_step.
The page-create and page-update endpoints accept a sibling `show_page_step`
block for managing that list.

! **Do not use `funnel.products` or `funnel.product_ids`** - those shapes
were briefly considered during development and are now explicitly rejected
with a 400. The supported field is `show_page_step.product_ids` as a
sibling of (not nested inside) `funnel`.

Attach on create (the page must be linked to a funnel via the `funnel` block):

```
POST /api/v2/workspaces/:workspace_id/pages
{
  "page": {
    "name": "Order",
    "current_path": "/order",
    "funnel": { "funnel_id": "<funnel_public_id>" },
    "show_page_step": {
      "product_ids": ["<product_public_id>", "<product_public_id>"]
    }
  }
}
```

Append on update (the page must already be part of a funnel):

```
PATCH /api/v2/pages/:page_id
{
  "page": {
    "show_page_step": {
      "product_ids": ["<product_public_id>"]
    }
  }
}
```

Both are **additive only** - products listed are appended; duplicates (across
requests or within a single batch) are skipped; nothing is ever removed via
this field. All-or-nothing - if any public ID does not resolve to a workspace
product, the page is not created/updated and a 400 is returned.

#### Order bumps

An **order bump** is an optional add-on offered next to the main products.
Bumps live on the same step as the main products, distinguished by a flag on
the attachment. Attach them with `show_page_step.bumps`, a sibling of
`product_ids` with the same additive, all-or-nothing semantics - one object per
bump, each carrying its own copy:

```
PATCH /api/v2/pages/:page_id
{
  "page": {
    "show_page_step": {
      "product_ids": ["<main_product_public_id>"],
      "bumps": [
        {"product_id": "<bump_product_public_id>", "preheadline": "One-time offer"}
      ]
    }
  }
}
```

The two blocks are **independent** - each affects only the products it names,
and neither is paired with the other by position. Send either, both, or neither:

| Sent | Result |
|------|--------|
| `product_ids` only | Those products attach as main products. Bumps already on the step keep their flag and copy. |
| `bumps` only | Those products attach (or are promoted) as order bumps. Main products already on the step are untouched. |
| Both | Each list is applied on its own terms. Mains are appended first, so they sort ahead of the bumps. |
| 2 in `product_ids`, 1 in `bumps` | Two main products and one bump. No pairing is inferred - the bump is not "the bump for" either main product, and its `preheadline` applies to that one bump only. |
| The same product in both | 400. On a given step a product is one or the other, so nothing is saved. |

- A product **already attached** as a main product is promoted to a bump when
  sent in `bumps` - it keeps its position, variants and prices. This is how an
  existing attachment becomes a bump.
- `product_ids` never demotes a bump back to a main product. To do that,
  detach the product (see below) and re-attach it via `product_ids`.
- `preheadline` is the short line of copy shown above that one bump's offer.
  Sending it for a product that is already a bump replaces that bump's copy,
  so it is also how you edit it; omit it to leave the current copy alone.
- Every `bumps` entry needs a `product_id` (400 otherwise), and the block must
  be an array of objects - an array of bare ids is a 400, not a silent no-op.
- `bumps` is accepted on page create and update, but **not** on
  `POST /api/v2/workspaces/:workspace_id/pages/external` (422) - create the
  external page first, then PATCH it.

To remove products, use the dedicated DELETE:

```
DELETE /api/v2/pages/:page_id/products
Content-Type: application/json

{ "product_ids": ["<product_public_id>", "<product_public_id>"] }
```

Returns **204 No Content** on success. Transactional batch - if any id does
not resolve, or any id is not currently attached to the step, no products are
detached. Detaching is by product id and ignores the bump flag: main products
and order bumps are removed the same way.

The live list of attached products is exposed on every page response under
`show_page_step.products` - read it back to confirm state. Each entry carries
`bump` (true for order bumps) and `bump_preheadline` (the copy sent as that
bump's `preheadline`).

#### Common product errors

| Error | Cause |
|-------|-------|
| 400 "Cannot attach products to a page that isn't part of a funnel." | `show_page_step.product_ids` or `bumps` sent on a standalone page (POST without a `funnel` block, or PATCH on a non-funnel page) |
| 400 "Product(s) cannot be listed in both product_ids and bumps: ..." | The same product was sent as both a main product and an order bump in one request |
| 400 "Every show_page_step.bumps entry requires a product_id." | A `bumps` entry carried only a `preheadline` |
| 400 "show_page_step.bumps must be an array of objects, each with a product_id." | `bumps` was sent as an array of bare product ids, or as something other than an array |
| 400 "show_page_step.bumps has unsupported keys: ..." | A `bumps` entry carried a key other than `product_id` / `preheadline` (a typo like `bump_preheadline`) |
| 400 "Product(s) not found: ..." | One or more public IDs do not resolve to a product in the workspace |
| 400 "`funnel.products` is not supported. Use `show_page_step.product_ids` ..." | Legacy shape - move the array out of the `funnel` block into a sibling `show_page_step` block |
| 400 "product_ids is required" (DELETE) | Empty/missing `product_ids` in the DELETE body |
| 404 "Product(s) not attached to this page: ..." (DELETE) | Resolved product is not currently attached to the step |

## Positioning a page within a funnel

When you attach a page to a funnel via `funnel.funnel_id`, you can control where
the new step lands with a top-level **`sort_order`** (0-based):

- `sort_order: 0` inserts before the first step.
- `sort_order: N` inserts at index N, shifting later steps down by one.
- Omit `sort_order` to append at the end.
- Out-of-bounds values (negative, or greater than the current step count) -> 422.

```
POST /api/v2/workspaces/:workspace_id/pages
{ "page": { "name": "Middle", "current_path": "/middle", "sort_order": 1, "funnel": { "funnel_id": "<funnel_public_id>" } } }
```

On update, `sort_order` repositions the page's existing step (the page must
belong to exactly one step), and `funnel.show_page_step_id` swaps the page onto
an existing step (the previous page is orphaned - kept in the workspace, no
longer attached to any step). When both are sent, the swap wins.

## External (SDK) Pages

**Closed Alpha** - not yet enabled for all workspaces. Request access at
https://developers.myclickfunnels.com/page/code-support.

An *external* page is hosted on the customer's own domain; ClickFunnels tracks
and embeds it via the SDK rather than rendering PML. An external page can be
created directly inside a funnel, or **standalone** (registered with an SDK
token but not yet a step in any funnel - see below).

### Create an external page

```
POST /api/v2/workspaces/:workspace_id/pages/external
{
  "page": {
    "name": "Upsell",
    "external_url": "https://acme.example.com/upsell",
    "sort_order": 1,
    "funnel": { "funnel_id": "<funnel_public_id>" }
  }
}
```

- **Required:** `external_url`.
- **Funnel block (optional):** `funnel.funnel_id` (append/insert) or
  `funnel.show_page_step_id` (swap onto an existing step). Omit the whole
  block to create a standalone external page.
- Positioning (with `funnel.funnel_id`): `sort_order` for a 0-based index, or
  `funnel.after_show_page_step_id` to insert right after a specific step - pass
  the `show_page_step_id` you read from [Funnel Structure](/.well-known/funnels/skill.md#funnel-structure).
  Omit both to append. Positioning params without their funnel target are a
  422: `sort_order` and `funnel.after_show_page_step_id` both require
  `funnel.funnel_id`.
- **Rejected** (422): `markup`, `theme_id`, `current_path`, `seo_*`,
  `head_code`, `footer_code`, `custom_css`, `live_data_changes`, `show_page_step.product_ids`.

### Standalone external pages

Creating with only `external_url` (+ optional `name`/`description`) registers
the page and mints its SDK token without placing it in any funnel. A
standalone page:

- serves the SDK for visit tracking (`/init` works as normal);
- cannot take form submissions or checkout yet - those return a
  `page_not_in_funnel` error until the page becomes a funnel step;
- is the way to place an external page inside a **split branch**: create it
  standalone, then pass its id as `branches[].page_id` on a conditional split
  or as the fresh-variant `page_id` on a split test (see
  [Funnels Skill](/.well-known/funnels/skill.md#placing-an-external-sdk-page-in-a-branch)).
  You can also attach it later by swapping it onto an existing step
  (`PATCH /api/v2/pages/:id` with `funnel.show_page_step_id`).

The response carries an `sdk` block:

```json
{ "sdk": { "token": "cfp_...", "external_url": "https://acme.example.com/upsell" } }
```

Paste the token into the page's `<head>` and load the SDK script:

```html
<meta name="cf-page-token" content="cfp_...">
<script src="https://sdk.myclickfunnels.com/sdk.js" defer></script>
```

### Update an external page

`PATCH /api/v2/pages/:id` is unified for internal and external pages. On an
external page only `name`, `description`, `sort_order`,
`funnel.show_page_step_id`, and `external_url` are valid (internal-only fields
like `markup`/`seo_*` return 422). Updating `external_url` preserves the SDK
token - no need to re-paste the meta tag.

## Delete a page

```
DELETE /api/v2/pages/:id
```

`204 No Content` on success. Works for internal and external pages - the page's
funnel steps are torn down first, then the page (and, for external pages, its
SDK registration).

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| 400 "Markup is not valid PML: ..." | Invalid DSL | See [Page Markup Skill -> Error Reference](/.well-known/page-markup/skill.md#18-error-reference) |
| 400 "Markup appears to be truncated: ..." | The `markup` you sent stops mid-construct, so it was cut off before you finished writing it. The page was left untouched. | Re-send the complete document; never send just the missing tail. If you only needed to add a script, use [Add a tracking pixel or custom code](#add-a-tracking-pixel-or-custom-code) instead. |
| 401 "API key missing or invalid" | Missing, revoked, or wrong token | Re-authenticate |
| 403 "External (SDK) pages are in Closed Alpha..." | Closed Alpha access is not enabled for the workspace | Request access at https://developers.myclickfunnels.com/page/code-support |
| 403 "This endpoint is restricted to trusted developer platforms..." | You are a third-party platform app and the team lacks trusted-platform access. Gates `head_code`/`footer_code`/`custom_css` AND `markup` writes alike. | Stop and tell the user to apply for trusted platform access at https://developers.myclickfunnels.com. Do NOT retry via `markup` - it is behind the same gate. |
| 404 "Not found" | Page does not exist, or token does not have access | Check page id and authorization |
| 422 "external_url is required for external pages." | Missing `external_url` on `/pages/external` | Provide a valid http(s) URL |
| 422 "sort_order ... is out of bounds" | Position past the funnel's step count | Use a value between 0 and the step count |
