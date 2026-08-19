---
name: clickfunnels-workflows
description: >
  Build marketing-automation Workflows programmatically - triggers, ordered
  action steps (email, tagging, attributes, opportunities, course/community
  access, webhooks, delays), conditional / A-B branching, enabling, and manually
  enrolling contacts. Use this skill when you need to create or configure an
  automation (welcome series, lead routing, tagging, onboarding) via the public API.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/refine-filters/skill.md
    description: Companion - create reusable Refine filters consumed by conditional-split steps (also documents authentication).
  - path: /.well-known/funnels/skill.md
    description: The page-centric sibling - split tests / conditional splits inside a funnel's workflow.
  - path: /.well-known/emails/skill.md
    description: Companion - email templates, broadcasts, and sending addresses; a send_email_step can reference a template_id or carry an inline html_body.
---

# ClickFunnels Workflows Skill

A discovery hub for the automation **Workflows** API - standalone automations made of
**triggers** (events that enroll a contact) and ordered **steps** (the actions that run),
with conditional / A-B branching, plus **runs** (a contact's pass through the workflow).

## When to use this skill

- Stand up an automation: enroll on an event, then send an email, tag the contact, set a
  custom attribute, create a sales opportunity, grant course/community access, fire a
  webhook, wait, or hand off to another workflow.
- Route contacts down a **conditional split** (matches a stored filter -> matched/unmatched) or
  an **A-B split test** (weighted variants).
- Enable a workflow so it runs live, or **manually enroll** a specific contact.
- Read back the whole automation tree (`/structure`) or inspect a contact's runs.

## Authentication

OAuth2 password grant using a trusted OAuth application's credentials - the same access
token used across the V2 API. See the
[Refine Filters Skill](/.well-known/refine-filters/skill.md#authentication) for details.
Reads (list, show, `/structure`, runs) are never gated; any token authorized for the
workspace can read. **Trusted platform access is required only for writes, and only when
a third-party platform acts on workspaces owned by other teams.** When you act on your
own account (your own API key, or an OAuth app operating within its own team's
workspaces) writes always pass; only a cross-team third-party write without trusted
access gets `403`.

## Lifecycle (read this first)

A workflow is built in steps and only goes live when you explicitly enable it:

1. **Create** -> always starts as a **draft** (`status: "draft"`, `disabled: true`). You
   **cannot** create an already-enabled workflow, and `disabled` is not settable on create or
   update.
2. **Add >=1 trigger** (and usually some steps).
3. **Enable** (`POST .../enable`) -> flips to `status: "live"`. Enable **requires at least one
   active trigger** (422 otherwise) - which is why enabling can't happen at create time.
4. Live workflows enroll contacts when a trigger fires; you can also **manually enroll**.

## Endpoint reference

| Action | Method | Path |
|---|---|---|
| List workflows | GET | `/api/v2/workspaces/:workspace_id/workflows` |
| Create workflow | POST | `/api/v2/workspaces/:workspace_id/workflows` |
| Show / Update / Delete | GET/PATCH/DELETE | `/api/v2/workflows/:id` |
| Structure (full tree) | GET | `/api/v2/workflows/:id/structure` |
| Enable / Disable | POST | `/api/v2/workflows/:id/enable` / `/disable` |
| List / Create triggers | GET/POST | `/api/v2/workflows/:workflow_id/triggers` |
| Show / Update / Delete trigger | GET/PATCH/DELETE | `/api/v2/workflows/triggers/:id` |
| List / Create steps | GET/POST | `/api/v2/workflows/:workflow_id/steps` |
| Show / Update / Delete step | GET/PATCH/DELETE | `/api/v2/workflows/steps/:id` |
| List / Create runs | GET/POST | `/api/v2/workflows/:workflow_id/runs` |
| Show run | GET | `/api/v2/workflows/runs/:id` |

## Creating a workflow

```http
POST /api/v2/workspaces/:workspace_id/workflows
Authorization: Bearer <access_token>
Content-Type: application/json

{ "workflow": { "name": "Welcome Series" } }
```

`run_type` is server-controlled - API-created workflows are always `asynchronous` (steps run
via background jobs), so you don't pass it. The response includes `public_id`,
`status: "draft"`, `disabled: true`, and `active_runs_count` / `completed_runs_count`.

## Triggers

A trigger declares the `event_type_key` that enrolls a contact, plus optional **condition
FK ids** that narrow when it fires. Created triggers default to `active: true`.

```http
POST /api/v2/workflows/:workflow_id/triggers
Authorization: Bearer <access_token>
Content-Type: application/json

{ "workflows_trigger": { "event_type_key": "$contact.tag_applied", "contacts_tag_id": "<tag_public_id>" } }
```

**Valid `event_type_key` values** (anything else -> 422):

- Contacts: `$contact.tag_applied`, `$contact.tag_removed`
- Site: `$optin`, `$view`
- Orders / billing: `$order.successful_purchase`, `$order.cancelled`, `$order.churned`,
  `$transaction.renewal_failed`, `$subscription.upgraded`, `$subscription.downgraded`
- Sales: `$sales.contact_entered_stage`, `$sales.contact_exited_stage`
- Courses: `$course.enroll`, `$course.unenroll`, `$course.complete`,
  `$course.lesson_complete`, `$course.lesson_view`, `$course.section_complete`
- Community: `$community.join`, `$community.group_join`, `$community.topic_join`,
  `$community.post`, `$community.post_view`, `$community.ban`, `$community.unban`
- Email engagement: `$email.sent`, `$email.open`, `$email.click`, `$email.bounce`,
  `$email.spam_complaint`; and per-step `$send_email_step.sent`/`.open`/`.click`/`.bounce`/`.spam_complaint`
- Messaging: `$message_hub_event.contact_conversation_started`, `...contact_responded`,
  `...agent_conversation_started`, `...agent_responded`
- Appointments / survey: `$appointments.appointment_scheduled`,
  `$calendar_event.contact_registered`, `$survey.complete`

**Condition FKs** (all public ids, workspace-scoped + type-checked): `contacts_tag_id`,
`funnel_id`, `page_id`, `product_id`, `product_variant_id`, `sales_pipelines_id`,
`current_sales_pipelines_stage_id`, `course_id`, `courses_lesson_id`, `courses_section_id`,
`community_id`, `communities_space_id`, `communities_space_group_id`,
`appointments_calendar_id`, `appointments_event_type_id`, `emails_broadcasts_id`,
`calendar_event_id`, `survey_workflow_id`. Booleans: `active`, `allow_anonymous_contacts`.

> ! For **site events** (`$optin`, `$view`) scope with **`page_id`**, not `funnel_id` - these
> events carry no funnel association, so a `funnel_id` condition silently never matches.

## Steps

A step's body is `step_type_settings`: a **single-key map whose key both selects the step
type and types its value** (the key equals the read-only `step_type`).

```http
POST /api/v2/workflows/:workflow_id/steps
Authorization: Bearer <access_token>
Content-Type: application/json

{ "workflows_step": {
    "step_type_settings": { "contact_tag_step": { "action": "Add Tags", "contacts_tag_ids": ["<tag_public_id>"] } }
} }
```

Creating a `send_email_step` also requires a complete business mailing address in
the workspace's email settings; otherwise the request returns `422` and creates no
step. Check it with `GET /api/v2/workspaces/:workspace_id/emails/settings` and set it
with `PUT` to the same path. The [Emails skill](/.well-known/emails/skill.md#business-mailing-address-prerequisite)
has the request body.

**Placement** (optional): omit -> append to the end of the trunk; `position: <n>` -> insert at a
0-based trunk index; `after_step_id: <step_public_id>` -> insert right after that step. For a
branch, use `parent_step_id: <split_public_id>` + `branch`.

**Repositioning an existing step**: the same placement fields (`position`, `after_step_id`,
`parent_step_id` + `branch`) are also accepted on `PATCH /api/v2/workflows/steps/:id` - send
them (with or without `step_type_settings`) to move a step within the trunk or into/out of a
branch. `sort_order` is recomputed for you. Example - move a step to just after another:

```http
PATCH /api/v2/workflows/steps/:id
Authorization: Bearer <access_token>
Content-Type: application/json

{ "workflows_step": { "after_step_id": "<other_step_public_id>" } }
```

### Step catalog

| `step_type` key | What it does | Key settings (enums / notes) |
|---|---|---|
| `send_email_step` | Send a marketing email | `template_id` (the referenced template is **copied** into a step-owned template - like the builder UI, the step always owns its email, so later edits never touch the original; **create-only** - a `PATCH` that includes `template_id` returns `422`, edit in place with `html_body`/`text_body` or create a new step for a different template), or an inline `html_body`/`text_body` that mints a template for you (a `template_id` wins when both are given - see the [Emails skill](/.well-known/emails/skill.md)); plus `subject`, `preheadline`, `from_address_id`, `reply_to_address_id`. A subject-only step is accepted but `notSetup` and won't send. |
| `send_asset_step` | Email a downloadable asset | `asset_id` |
| `send_chat_message_step` | Send a chat message | `inbox_key`, `message`, `subject`, `start_new_conversation` |
| `contact_tag_step` | Add/remove tags | `action` in `"Add Tags"`/`"Remove Tags"`, `contacts_tag_ids` (array) |
| `contact_custom_attribute_step` | Set/clear a custom attribute | `action` in `"Add Attribute"`/`"Remove Attribute"`, `value`, `field_id` |
| `delay_step` | Wait before continuing | `delay_type` in `relative`/`day_of_week`/`day_of_month`; `interval` in `minutes`/`hours`/`days`/`weeks`/`months`/`years`; `duration`; `time_zone` or `use_contact_time_zone`; window fields |
| `notify_step` | Notify team members | `message`, `notify_type` within `["Email","Notification"]`, `user_ids` (array). **`user_ids` must be team members** - discover them via `GET /api/v2/teams/{team_id}/memberships` (use each membership's `user_id`); an id that is not a member of this workspace's team is rejected `422`. |
| `deliver_webhook_step` | POST to an outgoing webhook endpoint | `webhooks_outgoing_endpoint_id` |
| `enroll_step` | Enroll in course(s) | `course_ids`, `section_ids` (arrays). **Enrollment runs off `section_ids`** - pass the section ids you want to enroll in (find them via `GET /api/v2/courses/{course_id}/sections`). `course_ids` alone is accepted and the step shows `active`, but enrolls no one at run time. |
| `unenroll_contact_step` | Unenroll from course(s) | `course_ids`, `section_ids` (arrays); unenroll honors both. |
| `grant_community_access_step` | Grant community space access | `space_ids` (array) |
| `revoke_community_access_step` | Revoke community space access | `space_ids` (array) |
| `create_opportunity_step` | Create a sales opportunity | `name`, `value`, `assignment_strategy` in `unassigned`/`membership`/`pipeline`/`round_robin`, `move_existing`, `sales_pipeline_id`, `sales_pipelines_stage_id`, `membership_id`, `membership_ids` |
| `move_step` | Hand the contact to another workflow | `workflow_id` (cannot be its own workflow or an archived one -> 422) |
| `conditional_split_step` | Branch on a stored filter | `condition.filter_id` (a Refine filter public id) |
| `split_test_step` | Weighted A-B branch | `variants`: exactly 2 `{ "weight": n }`, integers summing to 100 (default 50/50) |

Each step response carries a `state`: `active` (set up and will run), `notSetup` (accepted but
non-functional - fix the settings), or `notDefined` (a placeholder). **Always check `state`** -
a `notSetup` step is silently skipped at run time.

## Branching

Both split types auto-create their two branch containers plus a convergence node (where the
branches rejoin); you never manage convergence by hand. Add steps into a branch with
`parent_step_id` + `branch`:

- **conditional_split_step** - `branch`: `"matched"` or `"unmatched"` (the legacy `match`/`fallback` aliases are still accepted).
- **split_test_step** - `branch`: `"0"` or `"1"` (variant index).

```http
POST /api/v2/workflows/:workflow_id/steps
Authorization: Bearer <access_token>
Content-Type: application/json

{ "workflows_step": {
    "parent_step_id": "<conditional_split_public_id>",
    "branch": "matched",
    "step_type_settings": { "contact_tag_step": { "action": "Add Tags", "contacts_tag_ids": ["<tag_public_id>"] } }
} }
```

A step added with no placement after a split lands **after** that split's convergence (i.e. on
the main trunk again). The `filter_id` for a conditional split is the public id of a stored
Refine filter in this workspace (see the
[Refine Filters Skill](/.well-known/refine-filters/skill.md)); raw numeric ids are rejected.

## Enabling, disabling

```http
POST /api/v2/workflows/:id/enable      # requires >=1 active trigger -> status: "live"
POST /api/v2/workflows/:id/disable     # -> status: "draft", disabled: true
```

## Runs - manual enrollment

```http
POST /api/v2/workflows/:workflow_id/runs
Authorization: Bearer <access_token>
Content-Type: application/json

{ "run": { "contact_id": "<contact_public_id>", "skip_communication": false } }
```

Enrolls the contact and executes the steps (the workflow must be enabled). Set
`skip_communication: true` to suppress email/chat sends while still applying tags/attributes,
etc. List with filters and inspect a run:

```http
GET /api/v2/workflows/:workflow_id/runs?filter[contact_id]=<pid>&filter[status]=completed
GET /api/v2/workflows/runs/:id
```

Run `status` is derived: `active`, `paused`, `completed`, or `canceled`. Event-driven runs
carry a non-null `event_id`; manual enrollments do not.

## Reading the structure

```http
GET /api/v2/workflows/:id/structure
```

Returns the workflow header, its `triggers[]`, and the ordered `steps[]` with each step's
`step_type` + `step_type_settings`. Splits expose their `branches` (`matched`/`unmatched`) or
`variants` (`weight`) inline with nested `steps`, plus a `convergence_step_id` - the id of the
step the branches rejoin at (the first step after the split; `null` if nothing follows it). In
the example below the `VIP Check` split's branches both continue to the `Offer A/B` split test,
so its `convergence_step_id` is that step's id (`1172`); the trailing split test has none
(`null`). Internal container/convergence nodes are spliced out, so it's a clean tree you can
walk directly. This is the best single view to verify what you built.

FK references (`workflow_id`, `*_id` inside `step_type_settings`, `convergence_step_id`) are raw
integers; each node also carries its own integer `id` + obfuscated `public_id`.

```json
{
  "workflow": {
    "id": 183, "public_id": "MJnBZJ",
    "name": "Welcome & VIP Routing",
    "status": "live", "disabled": false, "archived": false,
    "run_type": "asynchronous"
  },
  "triggers": [
    { "id": 78, "public_id": "YjxoaJ", "event_type_key": "$contact.tag_applied",
      "active": true, "allow_anonymous_contacts": true }
  ],
  "steps": [
    {
      "id": 1162, "public_id": "jdwVBJ", "step_type": "send_email_step",
      "name": "Welcome Email", "sort_order": 0, "state": "active",
      "step_type_settings": {
        "send_email_step": {
          "subject": "Welcome aboard!", "preheadline": null,
          "template_id": 90, "from_address_id": 11, "reply_to_address_id": null
        }
      }
    },
    {
      "id": 1163, "public_id": "eqRXWJ", "step_type": "delay_step",
      "name": "Wait 3 Days", "sort_order": 1, "state": "active",
      "step_type_settings": {
        "delay_step": { "delay_type": "relative", "duration": 3, "interval": "days",
          "time_zone": null, "use_contact_time_zone": false, "use_execution_window": false,
          "day_of_week": null, "date_of_month": null, "start_time": null, "end_time": null,
          "execution_window_days": [] }
      }
    },
    {
      "id": 1165, "public_id": "vZxBpJ", "step_type": "conditional_split_step",
      "name": "VIP Check", "sort_order": 2, "state": "active",
      "step_type_settings": {
        "conditional_split_step": { "condition": { "filter_id": 40, "is_setup": true } }
      },
      "convergence_step_id": 1172,
      "branches": [
        { "branch": "matched", "steps": [
          { "id": 1170, "public_id": "vMaELv", "step_type": "contact_tag_step",
            "name": "Tag VIP", "sort_order": 0, "state": "active",
            "step_type_settings": { "contact_tag_step": { "action": "Add Tags", "contacts_tag_ids": [1] } } }
        ] },
        { "branch": "unmatched", "steps": [] }
      ]
    },
    {
      "id": 1172, "public_id": "jrBZaj", "step_type": "split_test_step",
      "name": "Offer A/B", "sort_order": 0, "state": "active",
      "step_type_settings": { "split_test_step": { "variants": [ { "weight": 60 }, { "weight": 40 } ] } },
      "convergence_step_id": null,
      "variants": [
        { "weight": 60, "steps": [] },
        { "weight": 40, "steps": [] }
      ]
    }
  ]
}
```

> **`sort_order` is container-local, not a global counter.** Each sequence numbers its own
> steps from 0 - the trunk, each split branch/variant, and the convergence sequence that runs
> after a split's branches rejoin. Above, `Offer A/B` shows `sort_order: 0` even though it runs
> after `VIP Check` (`sort_order: 2`), because it lives in the convergence sequence after the
> split. So `sort_order` repeats/restarts across splits and is not globally unique - **follow the
> `steps` array order for execution flow, not `sort_order`.**

## Worked example - a welcome series, end to end

```http
# 0) verify address_complete; configure it with PUT if false (see Emails skill)
GET  /api/v2/workspaces/5/emails/settings
# 1) create (draft)                -> capture workflow public_id as WF
POST /api/v2/workspaces/5/workflows           { "workflow": { "name": "Welcome Series" } }
# 2) trigger on opt-in (page-scoped)
POST /api/v2/workflows/WF/triggers            { "workflows_trigger": { "event_type_key": "$optin", "page_id": "<page_pid>" } }
# 3) send the welcome email
POST /api/v2/workflows/WF/steps               { "workflows_step": { "step_type_settings": { "send_email_step": { "subject": "Welcome!", "template_id": "<template_pid>", "from_address_id": "<address_pid>" } } } }
# 4) tag the subscriber
POST /api/v2/workflows/WF/steps               { "workflows_step": { "step_type_settings": { "contact_tag_step": { "action": "Add Tags", "contacts_tag_ids": ["<tag_pid>"] } } } }
# 5) go live  (needs the active trigger from step 2)
POST /api/v2/workflows/WF/enable
# 6) verify the tree
GET  /api/v2/workflows/WF/structure
```

## Common errors

| Error | Cause | Fix |
|---|---|---|
| 422 "A workflow cannot be enabled without at least one active trigger." | Enabling a workflow with no active trigger | Add a trigger first, then enable. |
| 422 "event_type_key '...' is not a valid workflow trigger event." | Unknown event key | Use one from the catalog above. |
| 422 "A complete business mailing address is required..." | Creating a send-email step before email settings have a complete business name/address | Use `PUT /api/v2/workspaces/:workspace_id/emails/settings`, then retry. |
| 422 "send_email_step requires a subject or a template_id." | Empty send-email settings | Provide a `template_id` (with body content) to actually send. |
| Step shows `state: "notSetup"` | Required settings missing/invalid (e.g. subject-only email, bad inbox) | Fix the settings; only `active` steps run. |
| 422 "move_step workflow_id cannot reference its own workflow." | A move target loop | Point at a different workflow. |
| 422 "move_step workflow_id references an archived workflow." | Archived move target | Use a non-archived workflow. |
| 422 "branch must be 'matched' or 'unmatched'..." / "...a variant index between 0 and 1." | Wrong branch token for the split type | Use matched/unmatched (conditional) or 0/1 (split test). |
| 422 "`<field>` '...' was not found in this workspace." | FK id from another workspace / wrong type | Use a public id of the right type in this workspace. |
| `$optin`/`$view` trigger never fires | Scoped by `funnel_id` | Scope site events by `page_id` instead. |
| 403 | Writing (create/update/delete) without trusted platform access | Reads are open to authorized callers; creating or modifying workflows requires trusted platform access. |
| 404 "Not found" | Workflow/step/trigger/run id outside your accessible scope | Verify the id and token. |
