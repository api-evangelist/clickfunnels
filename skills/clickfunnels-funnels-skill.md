---
name: clickfunnels-funnels
description: >
  Build and modify funnel workflows programmatically - split tests, conditional
  splits, branch positioning, multi-page branches, and stored-filter wiring.
  Use this skill when you need to add A/B testing or audience-based routing to
  a funnel via the public API.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/pages/skill.md
    description: Companion skill for building/editing pages programmatically via the Page Markup API.
  - path: /.well-known/page-markup/skill.md
    description: PML reference - the DSL used to author the `markup` field on pages and blog posts.
  - path: /.well-known/refine-filters/skill.md
    description: Companion skill for creating reusable Refine filters consumed by conditional splits and broadcasts.
---

# ClickFunnels Funnels Skill

A discovery hub for the funnel workflow APIs - split tests and conditional
splits - and how they wire together with stored filters and pages.

## When to use this skill

- Wrap an existing page in an A/B split test (variant page swap, weight tuning).
- Insert a conditional split that routes contacts down a "matched" or
  "unmatched" branch based on a stored filter.
- Grow a branch with multiple sequential pages (Welcome -> Onboarding -> CTA).
- Position a split between specific steps of a funnel using `show_page_step_id`.
- Read back the resulting tree (with the filter inline via `expand[]=filter`).

## Authentication

OAuth2 password grant using a trusted OAuth application's credentials. See the
[Refine Filters Skill](/.well-known/refine-filters/skill.md#authentication) for
details - the same access token is used here.

Reads (including `/structure`) and split-test steps are never gated. Some funnel
**writes** (e.g. conditional split steps) require **trusted platform access, but only
when a third-party platform acts on workspaces owned by other teams.** When you act on
your own account (your own API key, or an OAuth app within its own team's workspaces),
these writes always pass. Request trusted platform access at
https://developers.myclickfunnels.com/page/code-support.

## Endpoint reference

| Action                    | Method | Path                                                                |
|---------------------------|--------|---------------------------------------------------------------------|
| List split tests          | GET    | `/api/v2/funnels/:funnel_id/split_test_steps`                        |
| Create split test         | POST   | `/api/v2/funnels/:funnel_id/split_test_steps`                        |
| Show split test           | GET    | `/api/v2/funnels/:funnel_id/split_test_steps/:id`                    |
| Update split test         | PATCH  | `/api/v2/funnels/:funnel_id/split_test_steps/:id`                    |
| Destroy split test        | DELETE | `/api/v2/funnels/:funnel_id/split_test_steps/:id?branch=...`         |
| List conditional splits   | GET    | `/api/v2/funnels/:funnel_id/conditional_split_steps`                 |
| Create conditional split  | POST   | `/api/v2/funnels/:funnel_id/conditional_split_steps`                 |
| Show conditional split    | GET    | `/api/v2/funnels/:funnel_id/conditional_split_steps/:id`             |
| Update conditional split  | PATCH  | `/api/v2/funnels/:funnel_id/conditional_split_steps/:id`             |
| Destroy conditional split | DELETE | `/api/v2/funnels/:funnel_id/conditional_split_steps/:id?branch=...`   |

## Split test steps

A split test step wraps an existing page in a funnel into a 2-branch A/B test.
Each variant carries the `show_page_step` + `page` attached to that branch
(both `null` when the branch is empty), plus the branch's `weight` (0..100;
the two weights must sum to 100).

The first variant's `page_id` must already be attached to a step in this
funnel's workflow (that page gets wrapped in place - no cloning). The second
variant's `page_id` must be a page with **no existing show-page step** - an
internal page not yet attached to any funnel, or a standalone external (SDK)
page (see
[Placing an external (SDK) page in a branch](#placing-an-external-sdk-page-in-a-branch)).
Variant entries accept only `page_id` and `weight`; any other key is a 422.

### Positioning the split

By default (no `show_page_step_id`), the split lands at the funnel entry -
`sort_order 0` under the workflow root - with all surviving steps moved into
a `Root Step after Sequence End` convergence sequence positioned right after
the split.

When `show_page_step_id` references an existing show-page step in this funnel,
the split is inserted **after the workflow step that wraps it**, and steps
N+1...end are migrated under the convergence sequence.

`show_page_step_id` accepts a numeric or obfuscated
`Workflows::Steps::ShowPageStep` id - pass `show_page_step.id` (or
`show_page_step.public_id`) from any page response.

**Positioning caution (applies to split tests AND conditional splits):**
creating a split MOVES the wrapped step inside a branch sequence, so a step id
you fetched earlier may no longer sit where you think. Read
[Funnel Structure](#funnel-structure) immediately before creating a split to
pick the anchor from the current tree, and re-read it immediately after to
confirm placement. Anchoring on a step that sits inside another split's branch
nests the new split inside that branch - if you want a split to apply to ALL
traffic after an A/B test, create it before the A/B test (or anchor it on a
step outside the branches), not on a page that is already a variant.

### Example - wrap step N+1 as the test variant

```http
POST /api/v2/funnels/123/split_test_steps
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "split_test_step": {
    "show_page_step_id": "aBcDeF",
    "variants": [
      {"page_id": 500, "weight": 60},
      {"page_id": 501, "weight": 40}
    ]
  }
}
```

The response is the new `SplitTestStep` resource with both variants resolved.

### Destroying a split test

```http
DELETE /api/v2/funnels/123/split_test_steps/9876?branch=left
DELETE /api/v2/funnels/123/split_test_steps/9876?branch=right
DELETE /api/v2/funnels/123/split_test_steps/9876?branch=both
```

`branch` defaults to `both`. **A 1-branch split is not a supported state**, so
removing either single branch (`left` or `right`) collapses the entire split:
the surviving branch's pages are promoted up under the split's parent (so they
remain attached to the funnel's main workflow) and the split test wrapper is
destroyed. `branch=both` is the same end-state with both sides discarded.

## Conditional split steps

A conditional split routes contacts down one of two branches based on whether
they match a stored Refine filter:

- `branches[0]` - **matched** branch (filter matches the contact)
- `branches[1]` - **unmatched** branch (filter does not match)

Branches can be empty, hold a single page, or grow into multi-page sequences
(see below).

### Empty create

```http
POST /api/v2/funnels/123/conditional_split_steps
Authorization: Bearer <access_token>
Content-Type: application/json

{}
```

Creates an empty unconfigured split that the caller can fill in via PATCH.

### Multi-page branches via `page_ids`

```http
POST /api/v2/funnels/123/conditional_split_steps
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "conditional_split_step": {
    "branches": [
      {"page_ids": [600, 601, 602]},
      {"page_ids": [603]}
    ],
    "show_page_step_id": 100
  }
}
```

`page_id` (single) and `page_ids` (array) are mutually exclusive per branch,
and branch entries accept **only** those two keys - anything else (e.g.
`external_url`, `show_page_step_id`) is a 422. Each referenced page must be a
workspace-owned page with **no existing show-page step**: an internal page not
yet attached to any funnel, or a standalone external (SDK) page.

### Placing an external (SDK) page in a branch

Branch entries never take a URL - they take a page id. To route a branch to a
page you host yourself:

1. Create a **standalone** external page (no `funnel` block) - see
   [External (SDK) Pages](/.well-known/pages/skill.md#standalone-external-pages):

   ```http
   POST /api/v2/workspaces/42/pages/external
   Authorization: Bearer <access_token>
   Content-Type: application/json

   {"page": {"name": "External OTO", "external_url": "https://acme.example.com/oto"}}
   ```

2. Pass the returned page id in the branch (or split-test variant) slot:

   ```http
   POST /api/v2/funnels/123/conditional_split_steps
   Authorization: Bearer <access_token>
   Content-Type: application/json

   {
     "conditional_split_step": {
       "branches": [{}, {"page_id": "<page_public_id>"}]
     }
   }
   ```

This works the same for a split test's second variant (`variants[1].page_id`).
The same rule applies to internal pages: create the page first (unattached to
any funnel), then reference it by id.

### Attaching a filter

Once a stored filter exists (see the
[Refine Filters Skill](/.well-known/refine-filters/skill.md#applying-filters-to-conditional-split-steps)),
attach it via `condition.filter_id`:

```http
PATCH /api/v2/funnels/123/conditional_split_steps/9876
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "conditional_split_step": {
    "condition": {"filter_id": "AbCdEf"}
  }
}
```

`filter_id` is the public id of a stored filter in the same workspace. Raw
numeric ids are rejected for cross-tenant safety. Filters that were created
under the standard public API are subject to the
[safe-condition whitelist](/.well-known/refine-filters/skill.md#safe-condition-whitelist) -
once persisted, they can be referenced freely from this surface (no extra gating
on the consumer side).

### Reading back with the inline filter

```http
GET /api/v2/funnels/123/conditional_split_steps/9876
GET /api/v2/funnels/123/conditional_split_steps?expand[]=filter
```

On the show endpoint, `condition.filter` is always included as the full
`RefineFilter` object. On the list endpoint it is included only when
`expand[]=filter` is passed.

### Destroying a conditional split

```http
DELETE /api/v2/funnels/123/conditional_split_steps/9876?branch=matched
DELETE /api/v2/funnels/123/conditional_split_steps/9876?branch=unmatched
DELETE /api/v2/funnels/123/conditional_split_steps/9876?branch=both
```

`branch` defaults to `both`. **A 1-branch conditional split is not a supported
state**, so removing either single branch (`matched` or `unmatched`) collapses
the entire split: the surviving branch's pages are promoted up under the
split's parent (so they remain attached to the funnel's main workflow) and the
conditional split wrapper is destroyed. `branch=both` is the same end-state
with both sides discarded.

## Funnel structure

```http
GET /api/v2/funnels/:id/structure
```

Returns a lean, flattened tree of the funnel's **visible** steps - show-page,
conditional-split, and split-test steps, in order. The workflow's internal
container steps (the root container, split-branch containers, and the
post-split convergence) are spliced out, so `steps` is a flat ordered list you
can walk directly. Useful for rendering an outline, finding a step, or picking a
`sort_order` before inserting a page.

! **FORBIDDEN - a funnel step's path is its own standalone slug, never the funnel
path + the step slug.** The `url` returned per step is composed **automatically**
by ClickFunnels from the funnel/step hierarchy - do not reconstruct or guess it.
When you create the pages that become these steps (via the
[Pages skill](/.well-known/pages/skill.md#setting-a-pages-path-current_path)),
each step's `current_path` must be its **own** slug only - never a parent path
concatenated with the child's slug. e.g. for funnel path `mangia-mangia-catering`,
set the step's `current_path` to its own path (`mangia-mangia-catering`),
**never** the nested `mangia-mangia-catering/catering`.

A worked example - `Entry -> Conditional Split (matched: VIP Welcome / unmatched:
Guest Welcome) -> Split Test (Premium Offer A 60% / Premium Offer B 40%)`. Note the
flattening: split children are inline under `branches`/`variants`, internal sequence
containers are gone, `sort_order` is container-local (it resets inside each branch),
and split nodes carry a `convergence_step_id` (the step where every path rejoins):

```json
{
  "funnel": { "id": 98, "public_id": "mYLmMN", "name": "Coaching Upsell Funnel" },
  "steps": [
    {
      "id": 1178, "public_id": "jxGkbJ", "step_type": "show_page_step",
      "name": "Entry", "sort_order": 0, "show_page_step_id": "WJmQqJ",
      "page": { "id": 1413, "public_id": "joXkgY", "name": "Entry",
        "url": "https://acme.myclickfunnels.com/entry", "external": false, "sdk_token": null }
    },
    {
      "id": 1184, "public_id": "JEDdre", "step_type": "conditional_split_step",
      "name": "Conditional Split Step", "sort_order": 1, "convergence_step_id": 1195,
      "branches": [
        { "branch": "matched", "steps": [
          { "id": 1189, "public_id": "vbaRZj", "step_type": "show_page_step",
            "name": "VIP Welcome", "sort_order": 0, "show_page_step_id": "neonkv",
            "page": { "id": 1418, "public_id": "YGBWDj", "name": "VIP Welcome",
              "url": "https://acme.myclickfunnels.com/vip-welcome", "external": false, "sdk_token": null } }
        ] },
        { "branch": "unmatched", "steps": [
          { "id": 1190, "public_id": "vyggrJ", "step_type": "show_page_step",
            "name": "Guest Welcome", "sort_order": 0, "show_page_step_id": "RJgLRj",
            "page": { "id": 1419, "public_id": "ewMRAj", "name": "Guest Welcome",
              "url": "https://acme.myclickfunnels.com/guest-welcome", "external": false, "sdk_token": null } }
        ] }
      ]
    },
    {
      "id": 1195, "public_id": "vXkkrj", "step_type": "split_test_step",
      "name": "Split Test Step", "sort_order": 1, "convergence_step_id": null,
      "variants": [
        { "weight": 60, "steps": [
          { "id": 1191, "public_id": "vpzzzJ", "step_type": "show_page_step",
            "name": "Premium Offer A", "sort_order": 0, "show_page_step_id": "MJnnnJ",
            "page": { "id": 1422, "public_id": "JygoNY", "name": "Premium Offer A",
              "url": "https://acme.myclickfunnels.com/premium-offer-a", "external": false, "sdk_token": null } }
        ] },
        { "weight": 40, "steps": [
          { "id": 1193, "public_id": "jQwwqe", "step_type": "show_page_step",
            "name": "Premium Offer B", "sort_order": 0, "show_page_step_id": "QeOPbJ",
            "page": { "id": 1420, "public_id": "eVNQme", "name": "Premium Offer B",
              "url": "https://acme.myclickfunnels.com/premium-offer-b", "external": false, "sdk_token": null } }
        ] }
      ]
    }
  ]
}
```

Common node fields: `id`, `public_id`, `step_type` (short token, e.g.
`show_page_step`), `name`, `sort_order`. Fields by `step_type`:

| `step_type` | Extra fields |
|--------|--------------|
| `show_page_step` | `show_page_step_id` (pass this to `funnel.show_page_step_id` on Create/Update Page to swap), `page: { id, public_id, name, url, external, sdk_token }` |
| `conditional_split_step` | `convergence_step_id` (where the branches rejoin), `branches: [{ branch: "matched"\|"unmatched", steps: [...] }]` |
| `split_test_step` | `convergence_step_id`, `variants: [{ weight, steps: [...] }]` |

Branch and variant `steps` arrays are flattened the same way (no nested
container wrappers). `page.external` and `page.sdk_token` are always present
on show-page nodes; `sdk_token` is `null` unless `external: true`. The funnel
structure endpoint is a read and is not gated by trusted platform access.

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| 422 "condition.filter_id must be a stable identifier, not a raw numeric id" | Sent the database PK | Use the filter's public id (e.g. `AbCdEf`). |
| 422 "condition.filter_id not found in this workspace" | Filter belongs to another workspace | Use a filter created in this workspace. |
| 400 "conditional_split_step.branches[i].page_id not found in this workspace" | Page id from another workspace | Use a page in this workspace. |
| 422 "conditional_split_step.branches[i].page_id is already used by another funnel step" | Page already wrapped in a `ShowPageStep` | Use a fresh page. |
| 400 "conditional_split_step.branches[i] must provide either page_id or page_ids, not both" | Mixed shapes per entry | Pick one. |
| 404 "Not found" | Funnel / step id does not belong to this workspace's accessible scope | Verify the funnel id and access token. |
| 401 "API key missing or invalid" | Missing, revoked, or wrong token | Re-authenticate. |
