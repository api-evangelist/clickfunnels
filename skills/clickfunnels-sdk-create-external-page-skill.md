---
name: clickfunnels-sdk-external-page
description: >
  Add the ClickFunnels SDK to a page you host yourself - your own domain, a
  static host, a Netlify/Vercel site, or even a local file:// - so it tracks
  visits, captures opt-ins, and carries contact identity across funnel steps.
  Use this skill when embedding the SDK into an HTML page: the cf-page-token
  meta tag, the sdk.js script, and the data-cf-element form markup the SDK
  reads. The page is registered with ClickFunnels - as a funnel step, or
  standalone (registered for tracking now, placed in a funnel later).
version: "1.0"
author: ClickFunnels
references:
  - path: /llms.txt
    description: Product overview + skills index.
  - path: /.well-known/funnels/skill.md
    description: Build/modify the funnel that hosts this page (incl. placing standalone external pages in split branches).
  - path: /.well-known/pages/skill.md
    description: Author CF-hosted pages programmatically (companion to externally-hosted SDK pages).
  - path: /.well-known/sdk/add-checkout/skill.md
    description: Sell products from an external page - checkout, order bumps, one-click one-time offers.
---

# ClickFunnels SDK - Create an External Page

An **external page** is a page you host anywhere on the web that behaves like a
native ClickFunnels funnel step: visitors are tracked, forms submit into CF
(contact creation, opt-in events, automations), and contact + visit identity
carries across steps - even when steps live on different domains.

You wire this up with **three tags and one attribute**. That is the entire
public surface. Everything else is internal to the SDK and handled for you.

## When to use this skill

- You have an HTML page (own domain, static host, Netlify/Vercel, or localhost for local development, possibly also `file://` with certain limitations, like form submission not working)
  and want it to act as a ClickFunnels opt-in / capture step.
- You're fixing a page where the SDK "isn't doing anything" - usually a missing
  `data-cf-element`, a stray form-level marker, or `sdk.js` pointing at the
  wrong host.
- You're adding a new field to an SDK-driven form.

## Register the page in ClickFunnels first

The SDK only works on a registered page: registration mints the page token.
The page can be placed into a funnel at registration time, or registered
**standalone** (no funnel step yet) and placed later. Two ways to do it:

- **Funnel builder (UI).** Open your funnel, add a step, and pick
  **External page**. Enter the page's public URL and a name. ClickFunnels then
  shows the exact two snippets to paste - the token meta tag and the script
  tag for **your** account's SDK host.
- **API.** `POST /api/v2/workspaces/:workspace_id/pages/external` returns the
  same token and URL in an `sdk` block - see
  [Create the external page via the API](#create-the-external-page-via-the-api).

## The entire public surface

### 1. Page token - `<meta name="cf-page-token">`

Paste the `cfp_...` token ClickFunnels issued for this page into a meta tag in
`<head>`:

```html
<head>
  <meta name="cf-page-token" content="cfp_aB3xY7zQ9pK2mNvT8r5JcDef">
</head>
```

The token (format `cfp_` + 24 alphanumeric characters) is how the SDK identifies
the page - not the URL. **One token per page**; never paste the same token onto
two different pages. For a single-page app that swaps routes without a full
navigation, update the `content` attribute when the active "page" changes - the
SDK observes the meta tag and re-initializes against the new token.

### 2. The SDK script

Load the SDK once per page. Anywhere on the page works (it discovers and
enhances forms continuously, including ones added later):

```html
<script src="https://sdk.myclickfunnels.com/sdk.js" defer></script>
```

`defer` lets the page finish parsing before the SDK runs, which avoids race
conditions no matter where the tag sits. In a single-page app, load it **once**
for the whole app - the SDK keeps watching for forms (and for `cf-page-token`
changes) across client-side route changes, so you don't re-add it per route.

### 3. Mark elements with `data-cf-element`

A single attribute identifies every SDK-aware element. Its value names what the
element is:

```html
<form>
  <label for="signup-email">Email <span aria-hidden="true">*</span></label>
  <input id="signup-email" name="email" data-cf-element="email" type="email" autocomplete="email" required>
  <label for="signup-first-name">First name</label>
  <input id="signup-first-name" name="first_name" data-cf-element="first-name" autocomplete="given-name">
  <label for="signup-last-name">Last name</label>
  <input id="signup-last-name" name="last_name" data-cf-element="last-name" autocomplete="family-name">
  <button type="submit">Continue</button>
</form>
```

- Values are **flat, kebab-case** identifiers (`email`, `first-name`, `phone`).
  No flow prefixes - a field is a field regardless of which funnel it's in.
- Keep your own `name`/`id` attributes for labels, styling, and accessibility.
  The SDK reads **identity** from `data-cf-element`, not from `name`/`id`, and
  builds the submission payload itself.
- There is **no** form-level marker. The SDK finds the form by walking up from
  the `data-cf-element` elements inside it.

#### Custom contact attributes - `custom-attribute:<key>`

Anything beyond the fixed fields is a custom contact attribute. Use a colon
followed by your attribute key (the key is passed through verbatim, and the
attribute is created in your workspace on the fly the first time it's seen):

```html
<select data-cf-element="custom-attribute:business_type">...</select>
<input  data-cf-element="custom-attribute:monthly_revenue">
```

This is the only element value that contains a colon.

## Forms and submission

Any field that submits **must be inside a `<form>`** - the form is the
submission scope.

- **Per-form scoping.** Each `<form>` that contains a `data-cf-element` is an
  independent submission. Two such forms produce two separate submissions; the
  SDK never merges values across forms.
- **The SDK owns submission.** On submit it prevents the default, builds the
  submission payload itself, and POSTs to its own endpoint, then follows the
  redirect to the next funnel step. You don't set an `action` or `method`.
  That POST is a **real form navigation**, not a `fetch` - which is exactly what
  lets it follow the redirect to the next step - so a page served with a
  [Content Security Policy](#content-security-policy) must name the SDK host in
  `form-action`.
- **Don't mix non-SDK fields into an SDK form.** A form containing any
  `data-cf-element` is SDK-owned; put fields destined for a different endpoint
  in a separate form. (You don't need hidden UTM fields - visit tracking already
  captures UTMs from the landing-page URL.)

The SDK logs a console warning and ignores: an element on the wrong tag, a
submission field placed outside any `<form>`, and a non-SDK
`<input>/<select>/<textarea>` sitting inside an SDK form. **Open the console** -
it tells you exactly what it skipped and why.

**Regular DOM only - shadow roots are not currently supported.** The SDK
discovers elements with document-level queries and does not traverse shadow
roots (open or closed). Fields rendered inside a web component's shadow DOM -
including third-party embedded form widgets (an email provider's
`<vendor-form>` element, for example) - are invisible to it, and because the
SDK cannot see them it cannot even log a warning. Use a plain HTML form you
own, in the regular DOM. To also feed a third-party provider, run its widget
alongside your own form; the SDK cannot capture submissions from markup it
cannot see.

## Element registry

| Value                     | Tag(s)                                  | Purpose                                            |
| ------------------------- | --------------------------------------- | -------------------------------------------------- |
| `email`                   | `<input>`                               | Email capture                                      |
| `first-name`              | `<input>`                               | First name                                         |
| `last-name`               | `<input>`                               | Last name                                          |
| `name`                    | `<input>`                               | Single-field name (alternative to first/last)      |
| `phone`                   | `<input>`                               | Phone number                                       |
| `custom-attribute:<key>`  | `<input>`, `<select>`, `<textarea>`     | Arbitrary contact attribute (`<key>` is yours)     |

For an opt-in, you don't need a special submit element: a plain `type="submit"`
button works, because the SDK hijacks the form's submit.

Selling products from the page adds a few more elements (`payment-method`,
`pay`, `shipping`, `billing`, `oto-accept`, `oto-decline`) - see the
[Add Checkout skill](/.well-known/sdk/add-checkout/skill.md).

## Complete example - an opt-in page

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Join the list</title>

    <!-- 1. Page token from ClickFunnels -->
    <meta name="cf-page-token" content="cfp_aB3xY7zQ9pK2mNvT8r5JcDef">
  </head>
  <body>
    <form>
      <!-- 3. Fields the SDK operates on. Keep your own id/name for labels. -->
      <label for="first">First name <span aria-hidden="true">*</span></label>
      <input id="first" name="first_name" type="text" data-cf-element="first-name" autocomplete="given-name" required />

      <label for="email">Email <span aria-hidden="true">*</span></label>
      <input id="email" name="email" type="email" data-cf-element="email" autocomplete="email" required />

      <label for="biz">Business type</label>
      <select id="biz" name="business_type" data-cf-element="custom-attribute:business_type">
        <option value="agency">Agency</option>
        <option value="ecom">E-commerce</option>
      </select>

      <button type="submit">Continue</button>
    </form>

    <!-- 2. The SDK -->
    <script src="https://sdk.myclickfunnels.com/sdk.js" defer></script>
  </body>
</html>
```

## Create the external page via the API

Closed Alpha - request access at
https://developers.myclickfunnels.com/page/code-support. Requests use the
ClickFunnels V2 API with a Bearer token; see the
[Pages skill](/.well-known/pages/skill.md#authentication) for authentication.

Register the page by POSTing its public URL (and, optionally, a target
funnel):

```
POST /api/v2/workspaces/:workspace_id/pages/external
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "page": {
    "name": "Upsell",
    "external_url": "https://acme.example.com/upsell",
    "funnel": { "funnel_id": "<funnel_public_id>" }
  }
}
```

- **Required:** `external_url`.
- **Funnel block (optional):** `funnel.funnel_id` (append or insert into the
  funnel) or `funnel.show_page_step_id` (swap the page onto an existing step).
  Omit the whole block to register a **standalone** external page: the token
  is minted and visit tracking works, but form submission and checkout return
  a `page_not_in_funnel` error until the page becomes a funnel step. A
  standalone page's id is what you pass into a conditional split's
  `branches[].page_id` or a split test's fresh-variant slot - see
  [Placing an external page in a branch](/.well-known/funnels/skill.md#placing-an-external-sdk-page-in-a-branch).
- **Positioning** (with `funnel.funnel_id`): `sort_order` for a 0-based index,
  or `funnel.after_show_page_step_id` to insert right after a specific step -
  read step ids from the
  [Funnels skill's structure endpoint](/.well-known/funnels/skill.md#funnel-structure).
  Omit both to append at the end. Positioning params without `funnel.funnel_id`
  are a 422.

The response includes the page resource and an `sdk` block with everything the
page needs:

```json
{ "sdk": { "token": "cfp_aB3xY7zQ9pK2mNvT8r5JcDef", "external_url": "https://acme.example.com/upsell" } }
```

On external pages the page resource's `current_path` is `null` - the page has
no ClickFunnels-hosted URL; its address is your `external_url`.

! **FORBIDDEN - never build a nested path from a parent path.** This flow places
the page as a **funnel step**, and it is exactly the flow that has produced broken
nested URLs. Two rules:

- An external page carries **no** ClickFunnels `current_path` (it is `null`, as
  above) - its address is the `external_url` you host. Never fabricate a
  ClickFunnels-hosted path for one.
- Whenever you set a page's `current_path` (here, or when creating any CF-hosted
  funnel-step page), it must be that page's **own standalone slug only** - never a
  parent path (the funnel's path, a site path, etc.) concatenated with the page's
  own slug. ClickFunnels composes the full, nested public URL automatically from
  the funnel/step hierarchy; the caller must **never** guess, prefix, or build that
  composed URL itself. See the
  [Pages skill](/.well-known/pages/skill.md#setting-a-pages-path-current_path).

Real-world example - funnel path `mangia-mangia-catering`, with a funnel step of
its own:

- **Wrong** (funnel path prefixed onto the step's slug) -> `"current_path": "mangia-mangia-catering/catering"` -> yields the broken nested URL `https://<workspace>.myclickfunnels.com/mangia-mangia-catering/catering`
- **Right** (the funnel step's own standalone path) -> `"current_path": "mangia-mangia-catering"` -> yields the correct URL `https://<workspace>.myclickfunnels.com/mangia-mangia-catering`

Wire the token and script into your page exactly as described in
[The entire public surface](#the-entire-public-surface) - the meta tag carries
`sdk.token`, and the script host is your account's
`sdk.myclickfunnels.com` domain.

Afterwards:

- `PATCH /api/v2/pages/:id` updates `name`, `description`, `external_url`, or
  position. Changing `external_url` keeps the same token - no need to re-paste
  the meta tag.
- `DELETE /api/v2/pages/:id` detaches and removes the page.

Full parameter tables, rejected fields, and the error reference live in the
[Pages skill](/.well-known/pages/skill.md#external-sdk-pages).

## Split tests & conditional splits at the funnel entry

A **split test** or **conditional split** decides *which page a visitor sees*. When that
decision is the funnel's **first step**, there is no page to land on yet - the choice has
to be made *before* the visitor reaches any specific page. ClickFunnels makes that
decision at the **funnel's own URL** (the *Start of Funnel* URL in the funnel builder) and
then **302-redirects** the visitor on to the chosen step's `external_url`.

So when the entry step is a split, the entry point you advertise must be the **funnel
URL**, not a variant's `external_url`. A visitor who loads a variant page directly has
bypassed the decision - they just see that variant. (Loading a page only runs that page's
own visit tracking; the split itself is evaluated at the funnel URL.) This applies to both:

- a **split test** at the entry - the funnel URL allocates a variant by weight, then redirects to it;
- a **conditional split** at the entry - the funnel URL evaluates the stored filter, then redirects to the matched / unmatched branch's page.

(See the [Funnels skill](/.well-known/funnels/skill.md) for building split tests and conditional splits.)

### Branding the entry on your own domain

You usually want visitors to enter on *your* domain (e.g. `https://www.example.com/`). But
a hostname resolves to only one place - it is *either* your external host *or* ClickFunnels,
never both - and the funnel URL must be served by ClickFunnels. Bridge the two with a
redirect: keep the funnel URL on a ClickFunnels-served host, and point your domain's
**root** at it.

Three systems are involved:

1. **Your external host** (Netlify / Vercel / etc.) - serves the variant pages, and redirects the root.
2. **ClickFunnels** - the funnel whose URL performs the entry split and the redirect.
3. **The variant external pages** - each registered with its own `external_url` on your domain.

What to do:

1. **Register each entry variant at a non-root path.** The bare root `/` is going to be the
   redirect, so it can't also be a page. e.g. variant A -> `https://www.example.com/a.html`,
   variant B -> `https://www.example.com/b.html`. Wire these as the split's two branches.
2. **Put the funnel URL on a ClickFunnels host.** Use the funnel's public URL as-is, or add
   your own subdomain to ClickFunnels and assign it with **Set Domain** on the Start-of-Funnel
   step - e.g. `https://hostedonclickfunnels.example.com/your-funnel`.
3. **Redirect your root to the funnel URL** - only the exact root path. With Netlify
   (`netlify.toml`):

   ```toml
   [[redirects]]
     from = "/"
     to = "https://hostedonclickfunnels.example.com/your-funnel"   # the ClickFunnels funnel URL
     status = 302
     force = true
   ```

   Redirect **only `/`** - never the variant paths (`/a.html`, `/b.html`), or you create a
   loop (root -> funnel -> variant -> root -> ...).

Advertise `https://www.example.com/`. The flow is two redirects, no loop:

```
www.example.com/                          (your host: redirect)
  -> 302  hostedonclickfunnels.example.com/your-funnel        (ClickFunnels: runs the entry split / conditional)
  -> 302  www.example.com/a.html  (~50%)  -- or --  www.example.com/b.html  (~50%)
```

### It is sticky on reload

The funnel URL's choice is **per-visitor sticky** (carried by the visitor's cookie), so a
refresh or a return visit sends the same person to the **same** variant - a split test
won't flip between A and B on reload, and a conditional split keeps routing the contact to
the branch they already qualified for.

## Common mistakes

- **No `data-cf-element` anywhere.** The SDK discovers fields only by this
  attribute. Field `name`/`id` alone does nothing.
- **A form-level marker** (e.g. `data-...-optin`, `data-cf-sdk`). There is no
  form opt-in attribute - mark the fields, not the form.
- **Using undocumented attributes.** The SDK's author-facing attribute surface
  is exactly `data-cf-element` (this document) and `data-cf-visible`
  (documented in the Add Checkout skill). Any other attribute does nothing,
  silently.
- **Fields inside a shadow root.** The SDK does not traverse shadow DOM, so
  fields rendered by a web component - including third-party form widgets -
  are never discovered, with no console warning. Use your own form in the
  regular DOM.
- **Submission fields outside a `<form>`**, or **author fields mixed into an
  SDK form**. Both get a console warning; the second is dropped server-side.
- **Reusing one token across multiple pages.** Each page has its own.
- **Putting the token anywhere but the meta tag** (URL param, JS global, script
  attribute). The SDK reads it only from `<meta name="cf-page-token">`.
- **A Content Security Policy that omits the SDK host.** Nothing in your markup
  is wrong; the browser blocks the submit itself. See
  [Content Security Policy](#content-security-policy).

## Content Security Policy

Most hosts send no CSP at all, and then there is nothing to configure - the SDK
just works. But if your page **is** served with a `Content-Security-Policy` (a
response header, or a `<meta http-equiv="Content-Security-Policy">`), it has to
allow the SDK host in three directives:

| Directive     | Why the SDK needs it                                                          |
| ------------- | ----------------------------------------------------------------------------- |
| `script-src`  | `sdk.js` is a small loader; it injects the real SDK bundle from the same host  |
| `connect-src` | `/init`, the submit nonce, and visit/event tracking are fetch/XHR calls        |
| `form-action` | the opt-in submit is a **form navigation** to `/submit`, not a fetch           |

Every request the SDK makes goes to that one host, so a single origin in each
directive covers all of it (shown split across lines for readability; a real
header is one line):

```
Content-Security-Policy:
  script-src  'self' https://sdk.myclickfunnels.com;
  connect-src 'self' https://sdk.myclickfunnels.com;
  form-action 'self' https://sdk.myclickfunnels.com;
```

Checkout pages also load payment-provider assets - see the
[Add Checkout skill](/.well-known/sdk/add-checkout/skill.md).

### Symptom: submitting does nothing, and the console names `form-action`

```
Sending form data to 'https://sdk.myclickfunnels.com/submit' violates the
following Content Security Policy directive: "form-action 'self' ...".
The request has been blocked.
```

What makes this one confusing is that everything else looks healthy. The SDK
logs `Init response {ok: true, ...}`, reports the forms it found, sets its
tracking cookies, and rewrites each form's `action` correctly. Only the final
navigation is blocked, so the page just sits there.

Two things worth knowing while you debug it:

- **The directive is often not in your source.** Some app-hosting platforms
  inject a hardened CSP into every response they serve, so the page author never
  wrote the rule the error is quoting. Check what is actually served, not what
  you wrote:

  ```
  curl -sSI https://your-page.example.com | grep -i content-security-policy
  ```

  If the header comes from your platform rather than your own config, the SDK
  host has to be added there.
- **Retrying without a reload looks equally dead.** The SDK's double-submit
  guard latches on the first submit, so a second click after a blocked one is
  ignored on purpose. Reload the page between attempts.

To confirm CSP is the cause before you change anything, paste this into the
console on your page. It submits a throwaway form to a path that does not
exist, so no data is sent and nothing is created either way:

```js
const f = document.createElement('form')
f.method = 'GET'
f.action = 'https://sdk.myclickfunnels.com/__csp_probe__'
document.body.appendChild(f)
f.submit()
```

If the page navigates away, `form-action` is fine and the problem is elsewhere.
If you get the error above and the page stays put, add the SDK host to
`form-action`.

## Local & self-hosted testing

While developing, you can load the page two ways:

- **`http://localhost` (recommended).** Tracking *and* submits work, so you can
  exercise the full opt-in flow - including the redirect to the next funnel step -
  end to end.
- **`file://` (opened straight off disk).** The SDK loads and visit tracking
  fires, but **submits don't work**: browsers block the 302 redirect a `file://`
  POST has to follow to reach the next step. Use it only to confirm the script
  loads and tracks; switch to `http://localhost` to test submission.

For a hosted page, the page's URL must match what you registered on the page in
ClickFunnels (origin and path), or submits are rejected. Note that the API's
`external_url` accepts only `http`/`https` - a `file://` page can't be registered
as one; it's purely a local SDK-loading convenience.
