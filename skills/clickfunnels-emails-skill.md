---
name: clickfunnels-emails
description: >
  Author and manage marketing email programmatically - email templates (create,
  update, delete, with html_body/text_body), one-off broadcasts, and the
  sending domains and addresses they go out from. Use this skill when you need
  to create an email template via the API, send a broadcast, choose an address
  whose usable_as_sender flag is true, or attach an inline HTML email body to a
  workflow send-email step.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, API docs.
  - path: /.well-known/workflows/skill.md
    description: Companion - workflow send_email_step can carry an inline html_body that creates a template (this skill documents the template side).
  - path: /.well-known/refine-filters/skill.md
    description: Companion - build the audience filters a broadcast or workflow targets.
---

# ClickFunnels Emails Skill

A discovery hub for the email-content API: reusable **templates** (the HTML/text
body an email pulls from), **broadcasts** (a one-off send to an audience), and the
**addresses** an email is sent from. Templates created through the API are usable
for sending and previews, but are hidden from the visual template pickers and
cannot be edited in the visual editor - they are managed entirely
through the API.

## When to use this skill

- Create, update, or delete an email **template** from raw HTML/text.
- Provision and verify a custom email-sending **domain**.
- List a workspace's sending **addresses** and pick one whose `usable_as_sender` is true.
- Configure the workspace's business mailing address for marketing email.
- Create, update, or delete an email **broadcast**.
- Attach an inline HTML body to a workflow **send-email step** (see the Workflows skill).

## Authentication

OAuth2 with a trusted application's access token. Send it as a bearer token:

```http
Authorization: Bearer <access_token>
```

Reads (list/fetch) are open. Template and broadcast **writes** (create, update,
delete) and **provisioning a sending domain** require **trusted platform access only
when a third-party platform acts on workspaces owned by other teams**; such a
cross-team write without access returns `403`. Sender-address writes and
`POST /emails/domains/{id}/verify` are not gated. When you act on your own account
(your own API key, or an OAuth app within its own team's workspaces), writes always
pass.

## Endpoint reference

| Action | Method | Path |
|---|---|---|
| List templates | GET | `/api/v2/workspaces/{workspace_id}/emails/templates` |
| Fetch template | GET | `/api/v2/emails/templates/{id}` |
| Create template | POST | `/api/v2/workspaces/{workspace_id}/emails/templates` |
| Update template | PATCH | `/api/v2/emails/templates/{id}` |
| Delete template | DELETE | `/api/v2/emails/templates/{id}` |
| List addresses | GET | `/api/v2/workspaces/{workspace_id}/emails/addresses` |
| Create address | POST | `/api/v2/workspaces/{workspace_id}/emails/addresses` |
| Fetch address | GET | `/api/v2/emails/addresses/{id}` |
| Update address | PATCH | `/api/v2/emails/addresses/{id}` |
| Delete address | DELETE | `/api/v2/emails/addresses/{id}` |
| List sending domains | GET | `/api/v2/workspaces/{workspace_id}/emails/domains` |
| Provision sending domain | POST | `/api/v2/workspaces/{workspace_id}/emails/domains` |
| Fetch sending domain | GET | `/api/v2/emails/domains/{id}` |
| Verify sending domain | POST | `/api/v2/emails/domains/{id}/verify` |
| List broadcasts | GET | `/api/v2/workspaces/{workspace_id}/emails/broadcasts` |
| Create broadcast | POST | `/api/v2/workspaces/{workspace_id}/emails/broadcasts` |
| Update broadcast | PATCH | `/api/v2/emails/broadcasts/{id}` |
| Delete broadcast | DELETE | `/api/v2/emails/broadcasts/{id}` |
| Send/schedule broadcast | POST | `/api/v2/emails/broadcasts/{broadcast_id}/send_actions` |
| List send actions | GET | `/api/v2/emails/broadcasts/{broadcast_id}/send_actions` |
| Fetch send action | GET | `/api/v2/emails/broadcasts/send_actions/{id}` |
| Cancel scheduled send | DELETE | `/api/v2/emails/broadcasts/send_actions/{id}` |
| Show settings | GET | `/api/v2/workspaces/{workspace_id}/emails/settings` |
| Update settings | PUT | `/api/v2/workspaces/{workspace_id}/emails/settings` |

## Business mailing address prerequisite

Before you can create a broadcast or workflow `send_email_step`, the workspace must
have a complete business mailing address. Check `address_complete` first:

```http
GET /api/v2/workspaces/5/emails/settings
```

If it is `false`, set both the business name and address. This is the same endpoint
named in the `422` response from either create operation:

```http
PUT /api/v2/workspaces/5/emails/settings
Content-Type: application/json

{
  "marketing_setting": {
    "name": "Acme Inc.",
    "address": {
      "address_one": "123 Main St",
      "city": "San Francisco",
      "postal_code": "94105",
      "country_code": "US",
      "region": "CA"
    }
  }
}
```

A successful response has `address_complete: true`. Creating a broadcast or workflow
send-email step while it is incomplete returns `422` and creates nothing.

## Templates

A template carries `subject`, an HTML body (`html_body`) and a plain-text body
(`text_body`), and an optional `from_address`. Provide the body as
`html_body`/`text_body` on write. On **read** the body fields are **heavy** -
`html_body` is a per-template cloud-storage read - so they are returned only when
you ask for them with `expand[]=html_body` / `expand[]=text_body`; list and show
omit them by default (so a bare list never fans out to one read per row). This
works on writes too: pass the expand params on the `POST`/`PATCH` to get the body
echoed back in the response.

### Create a template

```http
# expand[]=html_body/text_body so the response echoes the body you just set
# (omit them and those two fields are absent from the response).
POST /api/v2/workspaces/5/emails/templates?expand[]=html_body&expand[]=text_body
Content-Type: application/json

{
  "emails_template": {
    "name": "Welcome Email",
    "subject": "Hello",
    "html_body": "<h1>Hi</h1>",
    "text_body": "Hi",
    "from_address_id": 12
  }
}
```

```json
{
  "id": 111,
  "public_id": "OezZVe",
  "workspace_id": 5,
  "name": "Welcome Email",
  "template_type": "email",
  "created_at": "2026-06-29T14:44:18.176Z",
  "updated_at": "2026-06-29T14:44:18.271Z",
  "subject": "Hello",
  "html_body": "<h1>Hi</h1>",
  "text_body": "Hi",
  "created_via_api": true,
  "from_address": {
    "id": 12,
    "public_id": "EjRpNb",
    "email_address": "marketing@example.com",
    "name": "My Team"
  }
}
```

Notes:

- `created_via_api` is `true` for templates made through this API. They are hidden
  from the visual template pickers and cannot be opened in the visual editor.
- `subject` may be blank.
- `from_address_id` must belong to the same workspace, or you get a `422`. Use an
  address whose `usable_as_sender` is true (see Addresses below).
- `template_type` is always `email`; passing any other value returns `422`.

### Update a template

Only the fields you send are changed; the rest are left intact.

```http
PATCH /api/v2/emails/templates/111
Content-Type: application/json

{ "emails_template": { "html_body": "<h1>Updated</h1>" } }
```

Invalid Liquid in `html_body` is rejected with `422` on update.

### Delete a template

```http
DELETE /api/v2/emails/templates/111
```

Returns `204` when the template is unused. Returns `422` if the template is shared
or is still referenced by a workflow send-email step (remove the step first).

## Sending domains

A sending domain is the part after the `@` of a custom sender address. Before
anything can be sent from it, that one domain name has to do **two independent
jobs**, proved in completely different ways. Sending needs both:

```
                       "example.com"
                              |
        +---------------------+----------------------+
        |                                            |
  WEBSITE DOMAIN                              SENDING DOMAIN
  connected in ClickFunnels                   (this API)
        |                                            |
  proves the workspace                        authenticates email
  owns the name                               transport
        |                                            |
  the customer connects it                    DKIM + SPF + DMARC records
  in the ClickFunnels app;                    at the DNS provider
  there is NO API route                       (+ a recommended return-path
  for this step                                CNAME), or a verified SMTP
        |                                      setting
        |                                            |
  "ownership_verified": true                  "verified": true
        |                                            |
        +---------------------+----------------------+
                              |
                   "ready_for_sending": true
                              |
             an address on this domain then reports
                   "usable_as_sender": true
```

Neither half substitutes for the other. Perfect DNS records do not prove
ownership, and connecting the website domain does not authenticate transport.
**`ready_for_sending` is the flag to poll; when it turns `true`, stop.** Check it
at most once every 10 minutes (results are cached for about that long, so asking
sooner returns the same answer), and do not sit in a polling loop: both of the
remaining steps need a human. After a check or two, hand the outstanding
`next_steps` back to the user and re-check when they tell you they are done.

**Provision the root domain (`example.com`), not a subdomain
(`news.example.com`).** Ownership is confirmed at the root domain: a website
domain is always connected as a subdomain (`www.example.com`, `news.example.com`,
`shop.example.com`), and connecting any of them confirms ownership of the one
sending domain `example.com`. Because nothing could ever confirm ownership of a
subdomain sending domain, provisioning one is rejected with `422` naming the root
domain to use instead:

```json
{"error": "Request unprocessable: Sending domains are set up on the root domain. Use \"example.com\" instead of \"news.example.com\"."}
```

Sender addresses are created at the sending domain's own name, so a root sending
domain gives you `hello@example.com`.

Sending domains cannot be renamed or removed through this API, so confirm the
exact spelling with the customer before the first `POST`.

### End-to-end flow

**Step 1 - provision the sending domain.**

```http
POST /api/v2/workspaces/5/emails/domains
Content-Type: application/json

{"emails_domain": {"name": "example.com"}}
```

```json
{
  "id": 11,
  "public_id": "OezDJd",
  "workspace_id": 5,
  "name": "example.com",
  "verified": false,
  "dkim_verified": false,
  "spf_verified": false,
  "dmarc_verified": false,
  "smtp_verified": false,
  "ownership_verified": false,
  "ready_for_sending": false,
  "return_address_verified": false,
  "dkim_record": {"type": "TXT", "name": "second._domainkey.example.com", "value": "v=DKIM1; k=rsa; p=MIIBIjAN...", "ttl": 300, "verified": false},
  "spf_record": {"type": "TXT", "name": "example.com", "value": "v=spf1 include:mailer.myclickfunnels.com -all", "ttl": 300, "verified": false},
  "dmarc_record": {"type": "TXT", "name": "_dmarc.example.com", "value": "v=DMARC1; p=none;", "ttl": 300, "verified": false},
  "return_address_record": {"type": "CNAME", "name": "cf2mail.example.com", "value": "mailer.myclickfunnels.com", "ttl": 300, "verified": false},
  "next_steps": [
    "Connect example.com to this workspace in ClickFunnels so ownership is confirmed. Addresses on this domain cannot send until it is, even once every record below verifies.",
    "Add dkim_record, spf_record, and dmarc_record at the DNS provider for example.com to authenticate email transport.",
    "Recommended for ClickFunnels bounce handling: add return_address_record at the DNS provider.",
    "DNS changes can take up to 48 hours to propagate. Re-check with POST /api/v2/emails/domains/OezDJd/verify; results are cached for about 10 minutes.",
    "Manual setup help: https://support.myclickfunnels.com/docs/email-settings-how-to-verify-email-dns-records"
  ],
  "created_at": "2026-06-29T14:44:18.176Z",
  "updated_at": "2026-06-29T14:44:18.271Z"
}
```

Creating the same name twice is safe and never duplicates: `201` when the domain
was newly provisioned, `200` when the workspace already had it. Two things about
the response that are easy to miss:

- `dkim_record` is `null` when the key has not finished generating. Hand over the
  other records meanwhile, wait a minute, then POST the same name again to retry.
  Stop after a few attempts and tell the customer rather than retrying in a loop.
- Provisioning also creates a `no-reply@example.com` sender address on the new
  domain, so the workspace already has one address on it before step 5. It reports
  `usable_as_sender: false` until the domain is ready.

This `POST` is the one gated call in the sending-domain flow. A third-party
platform provisioning on a workspace owned by another team needs trusted platform
access; without it nothing is provisioned and the call returns:

```json
{"error": "Forbidden: This endpoint is restricted to trusted developer platforms. Apply for trusted platform access at https://developers.myclickfunnels.com or contact support."}
```

Reads and `verify` on a domain the workspace already has are never gated.

**Step 2 - the customer publishes those records at their DNS provider.**
Hand over `dkim_record`, `spf_record` and `dmarc_record` verbatim (`name`, `type`,
`value`). `spf_record.value` may merge the domain's existing SPF record, so send
the returned value rather than a canned one. `return_address_record` is optional
but recommended: it is what routes bounce handling back to ClickFunnels.

**Step 3 - re-check with `verify`.** This is the only call that re-checks the
verification flags; listing and fetching return the stored result.

```http
POST /api/v2/emails/domains/OezDJd/verify
```

The usual result here is the state that trips people up (records omitted for
brevity; this customer also added the optional return-path record, so the
bounce-handling recommendation has dropped out of `next_steps`):

```json
{
  "id": 11,
  "public_id": "OezDJd",
  "name": "example.com",
  "verified": true,
  "dkim_verified": true,
  "spf_verified": true,
  "dmarc_verified": true,
  "smtp_verified": false,
  "ownership_verified": false,
  "ready_for_sending": false,
  "return_address_verified": true,
  "next_steps": [
    "Connect example.com to this workspace in ClickFunnels so ownership is confirmed. Addresses on this domain cannot send until it is, even once every record below verifies.",
    "DNS changes can take up to 48 hours to propagate. Re-check with POST /api/v2/emails/domains/OezDJd/verify; results are cached for about 10 minutes.",
    "Manual setup help: https://support.myclickfunnels.com/docs/email-settings-how-to-verify-email-dns-records"
  ]
}
```

Transport is fully authenticated, yet nothing can be sent: the workspace has not
proved it owns the name, so `ready_for_sending` stays `false` and every address on
the domain still reports `usable_as_sender: false`. **Do not read `verified: true`
as "done".** A `200` from `verify` only means the check ran, not that it passed -
always read the booleans back. Results are cached for about 10 minutes (polling
faster returns the same answer), and DNS itself can take up to 48 hours to
propagate.

**Step 4 - the customer connects the domain to the workspace in ClickFunnels.**
This is the website-domain connection in the app (connected as a subdomain, e.g.
`www.example.com`, which confirms ownership of the root sending domain
`example.com`), and **there is no API route for it** - an agent cannot do it, so
ask the customer to. Any read of
the domain reflects the result as soon as they are done: `ownership_verified` and
`ready_for_sending` both turn `true`, and `next_steps` ends with a "Ready to send"
line (the recommended return-path line stays in the list until that record
verifies, so do not treat a one-element `next_steps` as the ready signal):

```json
{
  "id": 11,
  "public_id": "OezDJd",
  "name": "example.com",
  "verified": true,
  "ownership_verified": true,
  "ready_for_sending": true,
  "next_steps": [
    "Ready to send - create addresses on this domain with POST /api/v2/workspaces/5/emails/addresses (emails_domain_id 11)."
  ]
}
```

**Step 5 - create a sender address on the domain.** Pass the domain's numeric `id`
as `emails_domain_id` to `POST /api/v2/workspaces/5/emails/addresses` (full example
under [Addresses](#addresses-pick-a-usable-sender) below); the new address comes
back with `usable_as_sender: true` and can be used as a template `from_address_id`,
a broadcast `from_email`, or a workflow send-email `from_address_id`. You may
create the address earlier in the flow - it is allowed, it just reports
`usable_as_sender: false` until step 4 lands.

A usable sender is only half of what a send needs: the workspace must also have a
complete business mailing address (see [Business mailing address
prerequisite](#business-mailing-address-prerequisite)) before a broadcast or a
workflow send-email step can be created.

### What each flag means

| Field | Meaning |
|---|---|
| `verified` | Transport authenticated: DKIM + SPF + DMARC all confirmed, or a verified SMTP setting. Says nothing about ownership. |
| `ownership_verified` | The workspace proved it owns the name by connecting it, or any subdomain of it, as a website domain in ClickFunnels. No API route; ask the customer. |
| `ready_for_sending` | Both of the above. This is the flag to poll for a domain authenticated by DNS records. One exception: a domain configured with its own SMTP settings reports its addresses as `usable_as_sender: true` even while this stays `false`. |
| `dkim_verified`, `spf_verified`, `dmarc_verified` | Per-record results of the last check - use them to tell the customer exactly which record is still missing. |
| `return_address_verified` | The recommended return-path CNAME. Improves bounce handling; not required for `ready_for_sending`. |
| `smtp_verified` | The domain sends through a verified SMTP setting instead of DNS records. When `true`, `verified` is `true` even with the DNS flags `false`. |
| `next_steps` | Plain-English list of what is left for this specific domain. Safe to surface to a human as-is. |

Each DNS record object (`dkim_record`, `spf_record`, `dmarc_record`,
`return_address_record`) carries `type`, `name`, `value`, `ttl` and its own
`verified` flag. `dkim_record` is `null` until the key finishes generating;
`return_address_record` is always returned.

## Addresses (pick a usable sender)

List the workspace's sending addresses, then use only an address whose top-level
`usable_as_sender` is `true` for `from_address_id` (templates),
`from_email`/`reply_to_email` (broadcasts), or
`from_address_id`/`reply_to_address_id` (workflow send-email steps).

```http
GET /api/v2/workspaces/5/emails/addresses
```

```json
[
  {
    "id": 12,
    "email_address": "my-team-aBcDeF@clickfunnelsmail.com",
    "name": "My Team",
    "public_id": "EjRpNb",
    "workspace_id": 5,
    "is_workspace_default": true,
    "usable_as_sender": false,
    "emails_domain": null
  },
  {
    "id": 20,
    "email_address": "no-reply@example.com",
    "name": "My Team",
    "public_id": "wjVaeR",
    "workspace_id": 5,
    "is_workspace_default": false,
    "usable_as_sender": true,
    "emails_domain": {
      "name": "example.com",
      "verified": true,
      "dkim_verified": true,
      "spf_verified": true,
      "dmarc_verified": true,
      "smtp_verified": false
    }
  }
]
```

Do not pick a sender from `emails_domain.verified` or `is_workspace_default`
alone. As the diagram above shows, `verified` only covers transport
authentication; `usable_as_sender` is the final contract, because it also
accounts for domain ownership, SMTP, and whether the shared-domain fallback is
still allowed. In the example, the ready custom sender is what makes the shared
fallback ineligible. The `no-reply@` address is the one that was created
automatically when the sending domain was provisioned.

The nested `emails_domain` object carries the transport flags only, and it does
**not** include the domain's `id`. For `ownership_verified`, `ready_for_sending`,
the DNS records or `next_steps`, match `emails_domain.name` against
`GET /api/v2/workspaces/{workspace_id}/emails/domains` and then fetch that record
with `GET /api/v2/emails/domains/{id}`. `usable_as_sender: false` while
`emails_domain.verified` is `true` almost always means ownership is still missing.

To create an address on a custom domain, pass the sending domain's numeric `id`
(step 5 of the flow above):

```http
POST /api/v2/workspaces/5/emails/addresses
Content-Type: application/json

{"emails_address": {"username": "hello", "name": "My Team", "emails_domain_id": 11}}
```

Omit `emails_domain_id` to get a managed address on the shared ClickFunnels
sending domain instead.

## Broadcasts

A broadcast is a one-off send. Provide `from_email` (required, must belong to the
workspace) and either an inline body (`html_body`/`text_body`) or an existing
`template_id`. When you pass a body without a `template_id`, a template is created
automatically from it. Creation also requires the complete business mailing address
described above, even though the new broadcast starts as a draft.

```http
POST /api/v2/workspaces/5/emails/broadcasts
Content-Type: application/json

{
  "emails_broadcast": {
    "name": "Weekly Newsletter",
    "subject": "This Week",
    "from_email": "marketing@example.com",
    "html_body": "<h1>Hello</h1>",
    "text_body": "Hello"
  }
}
```

Broadcasts are created in `draft`. `template_id` takes precedence over an inline
body when both are given. Use only `from_email`/`reply_to_email` addresses whose
list response has `usable_as_sender: true`.

While a broadcast is still `draft` you can **swap its template** by `PATCH`ing
`template_id` (the target must be a template in the same workspace; an unknown id
returns `422`). The swap wins over an inline body, same as on create. Once a
broadcast leaves `draft`, updates are rejected. Unlike a workflow send-email step
(which copies the template), a broadcast **references** the template you point it at.

On **read**, a broadcast nests the template it references as a lean `template` object -
`{ id, public_id, name }`. The rendered body is **not** inlined on the broadcast; fetch
it from the templates endpoint via that id: `GET /api/v2/emails/templates/{public_id}?expand[]=html_body`
(the templates endpoint expand-gates the body to keep its own list cheap).

### Sending or scheduling a broadcast

Creating a broadcast only drafts it. To actually send it, create a **send action**
against it. Omit `send_at` to send now; pass a future `send_at` (date-time) to
schedule. The broadcast must be ready to send: it needs a topic, a usable
`from_email`, and a complete marketing (physical) address on the workspace's email
settings - otherwise you get a `422`.

```http
POST /api/v2/emails/broadcasts/{broadcast_id}/send_actions
Content-Type: application/json

{ "emails_broadcasts_send_action": { "send_at": "2026-07-15T18:00:00Z" } }  # omit send_at to send now
```

Returns **201 Created** with the send action; it runs asynchronously, so fetch it
at `GET /api/v2/emails/broadcasts/send_actions/{id}` for `started_at`/`completed_at`.
`DELETE /api/v2/emails/broadcasts/send_actions/{id}` cancels a scheduled send.

## Inline body on a workflow send-email step

A workflow send-email step can take an inline `html_body`/`text_body` instead of a
pre-made `template_id`. The step mints its own template from that body. If you pass
both a `template_id` and a body, the `template_id` wins and the inline body is
ignored. Creating the step requires the complete business mailing address described
above. See the [Workflows skill](/.well-known/workflows/skill.md).

```http
POST /api/v2/workflows/{workflow_id}/steps
Content-Type: application/json

{
  "workflows_step": {
    "step_type_settings": {
      "send_email_step": {
        "subject": "Welcome",
        "html_body": "<h1>Welcome</h1>",
        "from_address_id": 12
      }
    }
  }
}
```

```json
{
  "id": 1206,
  "public_id": "jPoomJ",
  "workflow_id": 186,
  "step_type": "send_email_step",
  "state": "active",
  "step_type_settings": {
    "send_email_step": {
      "subject": "Welcome",
      "preheadline": null,
      "template_id": 96,
      "from_address_id": 12,
      "reply_to_address_id": null
    }
  }
}
```

`html_body`/`text_body` are write-only - they are never returned in
`step_type_settings`; the created `template_id` is.

The step's `template_id` (whether you passed one or it was minted from an inline body)
is **copied into a step-owned template**, mirroring the builder UI, and is
**create-only**: a later `PATCH` that includes `template_id` returns `422`. To change
the step's email after creation, send `html_body`/`text_body` (patched into its
existing template in place); to use a different template, create a new step.

## Worked example - template, then send it from a workflow

```
# 1) pick a usable sender
GET  /api/v2/workspaces/5/emails/addresses          -> choose an address with usable_as_sender: true

# 2) author a template
POST /api/v2/workspaces/5/emails/templates
     { "emails_template": { "name": "Welcome", "subject": "Hi", "html_body": "<h1>Hi</h1>", "from_address_id": 12 } }
     -> capture template id

# 3) reference it from a workflow send-email step
POST /api/v2/workflows/<workflow_pid>/steps
     { "workflows_step": { "step_type_settings": { "send_email_step": { "template_id": <id>, "from_address_id": 12, "subject": "Hi" } } } }
```

## Common errors

| Error | Cause | Fix |
|---|---|---|
| 403 Forbidden | A third-party platform writing a template or broadcast, or provisioning a sending domain, on another team's workspace without trusted access (own-account / same-team writes are never blocked) | Use your own API key, or request trusted platform access |
| 422 "A complete business mailing address is required..." | Email settings have no complete business name/address | Use `PUT /api/v2/workspaces/{workspace_id}/emails/settings`, then retry |
| 422 "from_address_id not found" | `from_address_id` is not in this workspace | Use an address id from `GET .../emails/addresses` |
| 422 "template_type must be 'email'" | Sent a non-email `template_type` | Omit `template_type` (it is always `email`) |
| 422 "Liquid syntax error: ..." | Invalid Liquid in `html_body` on update | Fix the Liquid markup |
| 422 "Template cannot be deleted ..." | Template is shared or used by a workflow step | Remove the referencing step first, then delete |
