---
name: clickfunnels-sdk-migrate-external-page-domain
description: >
  Move existing externally hosted ClickFunnels funnel pages from one web
  origin to another without replacing their SDK page tokens. Use this skill
  when changing the domain or host for external funnel pages while preserving
  every page path, funnel branch, visit tracking, and form submission flow.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Product overview and skills index.
  - path: /.well-known/sdk/create-external-page/skill.md
    description: Register an external page and embed its SDK token and script.
  - path: /.well-known/funnels/skill.md
    description: Read funnel structure, including nested split branches and variants.
  - path: /.well-known/pages/skill.md
    description: Page API authentication, response fields, and update behavior.
---

# ClickFunnels SDK - Migrate an External Page Domain

This skill moves existing external funnel pages from an old origin such as
`https://old.example` to a new origin such as `https://new.example`. It keeps
each page's exact path and ClickFunnels SDK page token. The safe order is:
prepare the host and HTTPS, inventory the complete funnel tree, update one page
at a time, then verify the tree and browser flow.

## When to use this skill

- Move all external pages in a funnel to a new domain or subdomain.
- Move only the pages currently served from one origin while leaving pages on
  other origins unchanged.
- Preserve a root page as the exact URL `https://new.example/`.
- Migrate a funnel that contains split tests, conditional splits, or nested
  combinations of both.
- Recover or roll back a partial domain migration.

## Authentication

Use the same ClickFunnels V2 API Bearer token that can already read the funnel
and update its pages. A domain migration does not mint a new access token or a
new SDK page token.

```http
Authorization: Bearer <access_token>
Content-Type: application/json
```

Never put an access token into hosted HTML, DNS records, source control, or a
skill document. The hosted page contains its `cfp_...` SDK page token, not the
V2 API Bearer token.

## Endpoint reference

| Action | Method | Path |
|---|---|---|
| Read every funnel step | GET | `/api/v2/funnels/:funnel_id/structure` |
| Update one external page URL | PATCH | `/api/v2/pages/:page_id` |

There is no bulk domain-migration endpoint. Each page update is a separate API
request, so keep a before-state manifest for verification and rollback.

## 1. Prepare the new origin before API writes

Do not update ClickFunnels until the new origin is ready to serve the existing
pages. Complete these host-side tasks first:

1. Add the new domain to the page host.
2. Point DNS to that host using the records the host provides.
3. Wait until the TLS certificate is active.
4. Deploy every page at the same path it used on the old origin.
5. Confirm each exact HTTPS URL loads without redirecting to a different path.
6. Keep the old origin live until the migration and rollback window are done.

For this source set:

| Old URL | Required new URL |
|---|---|
| `https://old.example/` | `https://new.example/` |
| `https://old.example/offer.html` | `https://new.example/offer.html` |
| `https://old.example/thanks/` | `https://new.example/thanks/` |

The root URL must include `/`: use exactly `https://new.example/`, not a
placeholder path. Preserve path spelling, case, percent encoding, file
extensions, and trailing slashes. The HTTPS URL at which the browser settles
must match the URL stored for that ClickFunnels page. Redirecting
`/offer.html` to `/offer`, for example, changes the path and can cause SDK
submissions to be rejected.

Before continuing, check the deployed URLs directly:

```http
GET https://new.example/
GET https://new.example/offer.html
GET https://new.example/thanks/
```

Each response should be successful over HTTPS and should render the intended
page. An HTTP-to-HTTPS redirect is fine; a redirect from one HTTPS path to a
different HTTPS path is not.

## 2. Read and recursively inventory the funnel

Read the complete structure before changing anything:

```http
GET /api/v2/funnels/MjngvY/structure
Authorization: Bearer <access_token>
```

A structure can contain show-page steps at the top level and inside either
kind of split. This abbreviated but valid response includes both cases:

```json
{
  "funnel": {
    "id": 83,
    "public_id": "MjngvY",
    "name": "Launch Funnel"
  },
  "steps": [
    {
      "id": 689,
      "public_id": "zvbpDJ",
      "step_type": "show_page_step",
      "name": "Opt-In",
      "sort_order": 0,
      "show_page_step_id": "qelNpj",
      "page": {
        "id": 1401,
        "public_id": "JXkbke",
        "name": "Opt-In",
        "url": "https://old.example/",
        "external": true,
        "sdk_token": "cfp_aB3xY7zQ9pK2mNvT8r5JcDef"
      }
    },
    {
      "id": 691,
      "public_id": "RvppEv",
      "step_type": "conditional_split_step",
      "name": "Audience Split",
      "sort_order": 1,
      "convergence_step_id": null,
      "branches": [
        {
          "branch": "matched",
          "steps": [
            {
              "id": 698,
              "public_id": "mjLGPJ",
              "step_type": "show_page_step",
              "name": "Offer",
              "sort_order": 0,
              "show_page_step_id": "KvXWoj",
              "page": {
                "id": 1402,
                "public_id": "eLQBxJ",
                "name": "Offer",
                "url": "https://old.example/offer.html",
                "external": true,
                "sdk_token": "cfp_Z9yX8wV7uT6sR5qP4nM3kJ2h"
              }
            }
          ]
        },
        {
          "branch": "unmatched",
          "steps": []
        }
      ]
    }
  ]
}
```

Walk every `steps` array recursively:

```javascript
function collectExternalPages(steps, pages = []) {
  for (const step of steps) {
    if (step.step_type === "show_page_step" && step.page?.external) {
      pages.push(step.page);
    }

    for (const branch of step.branches || []) {
      collectExternalPages(branch.steps || [], pages);
    }

    for (const variant of step.variants || []) {
      collectExternalPages(variant.steps || [], pages);
    }
  }

  return pages;
}
```

Do not stop at the top-level `steps` array. A conditional split stores child
steps in `branches[].steps`; a split test stores them in
`variants[].steps`. Either kind can contain another split.

Save a manifest with each external page's `id`, `public_id`, current `url`, and
`sdk_token`. Exclude internal pages (`external: false`) and external pages whose
origin is not the old origin unless the requested migration explicitly
includes them.

## 3. Build and check the URL map

Replace the origin, not arbitrary matching text. Parse each URL, require its
origin to equal the old origin, and copy its pathname to the new origin.
External page URLs must not contain query parameters. Stop if a saved URL has
one instead of silently removing it. Preserve a fragment if one is present.

```javascript
function parseOrigin(value) {
  const parsed = new URL(value);
  if (parsed.pathname !== "/" || parsed.search || parsed.hash) {
    throw new Error("Expected an origin without a path, query, or fragment");
  }
  return parsed;
}

function migrateUrl(value, oldOrigin, newOrigin) {
  const current = new URL(value);
  const expectedOld = parseOrigin(oldOrigin);
  const targetOrigin = parseOrigin(newOrigin);

  if (current.origin !== expectedOld.origin) return null;
  if (current.search) {
    throw new Error("Registered external-page URLs cannot contain a query");
  }

  const target = new URL(current.href);
  target.protocol = targetOrigin.protocol;
  target.host = targetOrigin.host;
  return target.href;
}
```

For the response above, the manifest becomes:

```json
[
  {
    "page_id": 1401,
    "public_id": "JXkbke",
    "old_url": "https://old.example/",
    "new_url": "https://new.example/",
    "sdk_token": "cfp_aB3xY7zQ9pK2mNvT8r5JcDef"
  },
  {
    "page_id": 1402,
    "public_id": "eLQBxJ",
    "old_url": "https://old.example/offer.html",
    "new_url": "https://new.example/offer.html",
    "sdk_token": "cfp_Z9yX8wV7uT6sR5qP4nM3kJ2h"
  }
]
```

Before writing, reject the plan if two pages map to the same canonical new URL.
Also confirm that each new URL serves the HTML containing that page's matching
`<meta name="cf-page-token" content="cfp_...">` value. Do not copy one page's
SDK token onto another page.

## 4. Patch one page at a time

Update only `page.external_url`. Start with a non-entry page when possible, so
you can verify one migrated page before moving the advertised root page.

```http
PATCH /api/v2/pages/1402
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "page": {
    "external_url": "https://new.example/offer.html"
  }
}
```

Relevant fields in the `200 OK` response:

```json
{
  "id": 1402,
  "public_id": "eLQBxJ",
  "name": "Offer",
  "current_path": null,
  "url": "https://new.example/offer.html",
  "sdk": {
    "token": "cfp_Z9yX8wV7uT6sR5qP4nM3kJ2h",
    "external_url": "https://new.example/offer.html"
  }
}
```

When the update response includes `sdk.token`, compare it with the saved
manifest. It must be unchanged. A client or API version may omit some response
fields, so do not treat the PATCH response as the final inventory. The
authoritative verification is a fresh funnel structure read after the writes.

Then patch the root page with the numeric `page.id` captured from the structure
and the exact root URL:

```http
PATCH /api/v2/pages/1401
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "page": {
    "external_url": "https://new.example/"
  }
}
```

Stop on the first error. Do not continue with later pages until you understand
whether the failed page was changed. Because calls are independent, pages
updated before a failure stay updated.

## 5. Verify the API and browser flow

Fetch the structure again and recurse through it exactly as before:

```http
GET /api/v2/funnels/MjngvY/structure
Authorization: Bearer <access_token>
```

Confirm all of the following:

- Every intended external page now uses the new origin.
- Every pathname exactly matches its before-state pathname.
- The root page URL is exactly `https://new.example/`.
- Pages on unrelated origins are unchanged.
- Every `sdk_token` equals its saved before-state value.

Then load the public pages in a browser. For each representative path, confirm:

1. The page loads over HTTPS at the URL stored by ClickFunnels.
2. The HTML still has the page's matching `cf-page-token` meta tag.
3. The ClickFunnels `sdk.js` request succeeds.
4. SDK initialization succeeds for that token and new URL.
5. Visit tracking succeeds.
6. A form submission reaches the SDK submit endpoint and redirects to the
   correct next funnel step.

A useful network sequence is `sdk.js` -> `init` -> `visits`, followed by
`submit` -> an HTTP redirect when the form is submitted. Test at least one
top-level page and one page nested inside a branch or variant when the funnel
contains splits.

Browser verification is preferred because the browser sends the page's
`Origin` automatically and exercises the actual `window.location`. If you call
the SDK `init` endpoint directly with an HTTP client, send an `Origin` header
that exactly matches the tested page origin:

```http
GET https://sdk.<your-cf-root-domain>/init?sdk_page_token=cfp_aB3xY7zQ9pK2mNvT8r5JcDef&url=https%3A%2F%2Fnew.example%2F
Origin: https://new.example
```

A direct `init` request without the matching `Origin` is expected to return
`403`; that result alone does not mean the hosted page is broken.

## Rollback

Use the saved manifest to PATCH each changed page back to its exact `old_url`.
Rollback also happens one page at a time:

```http
PATCH /api/v2/pages/1402
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "page": {
    "external_url": "https://old.example/offer.html"
  }
}
```

Keep both hosts available while rolling back. The SDK page token remains the
same in either direction.

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `401 API key missing or invalid` | The Bearer token is missing, revoked, or for the wrong account | Re-authenticate and retry the read before any write |
| `404 Not found` | The funnel or page is not accessible with this token | Verify the public id and token scope |
| `422 external_url ... already ... taken` | Another external page in the workspace already owns the canonical target URL | Give every page a unique target URL and rerun the preflight map |
| `422 external_url ... valid URL` | The target is not a complete HTTP or HTTPS URL | Send a full URL such as `https://new.example/offer.html` |
| SDK `url_mismatch` | The browser settled at a different origin or path from the stored URL | Remove path-changing redirects or PATCH the exact final browser URL |
| Direct SDK `init` returns `403` | The HTTP probe omitted the page's matching `Origin` header | Prefer a browser, or send the exact page origin in `Origin` |
| Some pages still use the old origin | The inventory ignored nested split children | Recurse through both `branches[].steps` and `variants[].steps` |
| Form reaches the wrong page | Hosted files or SDK tokens were mapped to the wrong paths | Restore the per-page path and matching `cf-page-token` from the manifest |
| Migration stopped halfway | A later PATCH failed after earlier calls succeeded | Inspect the structure, then complete the remaining map or roll back changed pages |

## Final checklist

- [ ] New host, DNS, and HTTPS were ready before the first API write.
- [ ] Every nested split branch and variant was recursively inspected.
- [ ] Only pages on the requested old origin were selected.
- [ ] The new origin was substituted while each exact path was preserved.
- [ ] Root was stored as exactly `https://new.example/`.
- [ ] Pages were PATCHed and verified one at a time.
- [ ] Every SDK page token remained unchanged.
- [ ] The structure, SDK initialization, visit tracking, and submission flow passed.
- [ ] The old host remains available for the agreed rollback window.
