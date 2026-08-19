---
name: clickfunnels-courses
description: >
  Build ClickFunnels 2.0 courses programmatically - create a course, add its
  sections (modules), and fill each section with lessons whose body is authored
  in Markdown, then publish the whole content tree in one step. Use this skill
  when you need to scaffold, populate, or publish course content via the public
  API. Requires an OAuth application token; course writes are not gated by
  trusted platform access.
version: "1.1"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/page-markup/skill.md
    description: PML reference - the richer page DSL, if you outgrow Markdown lesson bodies.
---

# ClickFunnels Courses Skill

The HTTP surface for creating CF2 courses and their content tree: a course holds
sections (the modules a learner sees), and each section holds lessons. Lesson
bodies are authored in Markdown. This document covers the content model, the
create/read/update lifecycle, and the gotchas worth knowing before you script it.

## When to use this skill

- Scaffolding a brand-new course (modules + lessons) from an outline or prompt.
- Adding sections (modules) to an existing course.
- Adding or updating lessons inside a section, with Markdown bodies.
- Reading back a course's structure (sections, then lessons per section).
- Publishing a completed course, all eligible modules, and their lessons in one step.

## How a course is structured

```
Course
  Root section          (auto-created, titled "Primary module")
    Section  (module)    <- what you create via the API
      Lesson
      Lesson
    Section  (module)
      Lesson
```

Every course is provisioned with one **root section** the moment it is created;
its id comes back as `root_section_id`. You never create it and it is not shown in
the course builder UI - it is the implicit parent of every top-level module. The
API sections list, however, DOES return it (the entry whose `parent_id` is `null`,
matching the course's `root_section_id`); most integrations can ignore that entry
and treat its children as the modules.

Two facts shape how you script content creation:

- **Sections you create via the API are top-level modules.** The create/update
  payloads do not accept a `parent_id` or a `sort_order`, so a new section is
  attached under the root and appended after the existing modules. Building
  multi-level nesting or re-ordering modules is done in the course builder UI,
  not through this API.
- **Lessons always belong to a section.** You create them under a specific
  `section_id`, never directly under the course.

## Authentication

OAuth2 password grant using a trusted OAuth application's credentials. The same
access token is used across the CF2 public API surfaces.

Course, section, and lesson **writes** (create/update) are available to any OAuth token
authorized for the workspace; they are **not** behind trusted platform access, so both
first-party and third-party OAuth apps may write once authorized for the workspace.
**Reads** are likewise available wherever the token has access to the workspace.

## Endpoint reference

| Action          | Method | Path                                              |
|-----------------|--------|---------------------------------------------------|
| List courses    | GET    | `/api/v2/workspaces/:workspace_id/courses`        |
| Create course   | POST   | `/api/v2/workspaces/:workspace_id/courses`        |
| Show course     | GET    | `/api/v2/courses/:id`                             |
| List sections   | GET    | `/api/v2/courses/:course_id/sections`             |
| Create section  | POST   | `/api/v2/courses/:course_id/sections`             |
| Show section    | GET    | `/api/v2/courses/sections/:id`                    |
| Update section  | PATCH  | `/api/v2/courses/sections/:id`                    |
| Publish section | POST   | `/api/v2/courses/sections/:id/publish`            |
| List lessons    | GET    | `/api/v2/courses/sections/:section_id/lessons`    |
| Create lesson   | POST   | `/api/v2/courses/sections/:section_id/lessons`    |
| Show lesson     | GET    | `/api/v2/courses/lessons/:id`                     |
| Update lesson   | PATCH  | `/api/v2/courses/lessons/:id`                     |
| Publish lesson  | POST   | `/api/v2/courses/lessons/:id/publish`             |
| Publish course or whole tree | POST | `/api/v2/courses/:id/publish`              |
| Publish sections| POST   | `/api/v2/courses/:course_id/sections/publish_actions` |
| List publish actions | GET | `/api/v2/courses/:course_id/sections/publish_actions` |
| Show publish action  | GET | `/api/v2/courses/sections/publish_actions/:id`   |

Note the two path shapes: collections that belong to a parent are nested under
it (`/courses/:course_id/sections`), while showing/updating a single section or
lesson uses the flat `/courses/sections/:id` and `/courses/lessons/:id` forms.

## Creating a course

```http
POST /api/v2/workspaces/365069/courses
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "course": {
    "title": "Agentic Engineering for Senior Software Engineers",
    "description": "A hands-on program on designing, building, and operating agentic AI systems."
  }
}
```

On success: **201 Created** with the course. Capture `id` (use it as
`:course_id`) and `root_section_id`.

```json
{
  "id": 86816,
  "public_id": "vERkqB",
  "title": "Agentic Engineering for Senior Software Engineers",
  "published_at": null,
  "root_section_id": 490402,
  "description": "A hands-on program on designing, building, and operating agentic AI systems.",
  "current_path": "/agentic-engineering-for-senior-software-engineers",
  "show_in_community": false,
  "show_to_non_members": false,
  "upgrade_url": null,
  "redirect_to_full_course": true,
  "unauthorized_redirect_url": null,
  "comments_enabled": false,
  "created_at": "2026-06-30T15:56:18.369Z",
  "updated_at": "2026-06-30T15:56:18.508Z",
  "image_url": null
}
```

### Permitted course fields

| Field | Type | Notes |
|-------|------|-------|
| `title` | string | Required. The slug (`current_path`) is derived from it. |
| `description` | string | Short course summary. |
| `show_in_community` | boolean | Surface the course in the community area. |
| `redirect_to_full_course` | boolean | Redirect behavior for members. |
| `image_id` | id | Cover image - id of a workspace Image resource. |

## Sections (modules)

Create a module under a course. Only `title` is required.

```http
POST /api/v2/courses/86816/sections
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "courses_section": {
    "title": "Course Orientation and Mental Models",
    "description": "Module for senior engineers."
  }
}
```

Returns **201 Created**:

```json
{
  "id": 490404,
  "public_id": "erqaPn",
  "course_id": 86816,
  "title": "Course Orientation and Mental Models",
  "published_at": null,
  "description": "Module for senior engineers.",
  "days_till_drip_access": null,
  "blocker_section_id": null,
  "blocker_lesson_id": null,
  "current_path": "/course-orientation-and-mental-models",
  "is_hidden_from_non_members": true,
  "upgrade_url": null,
  "parent_id": 490402,
  "sort_order": 1,
  "created_at": "2026-06-30T15:57:00.738Z",
  "updated_at": "2026-06-30T15:57:00.738Z"
}
```

`parent_id` comes back set to the course's `root_section_id` and `sort_order` is
assigned automatically (it appends after the existing modules). Neither field is
settable from this API.

### Permitted section fields (create and update)

| Field | Type | Notes |
|-------|------|-------|
| `title` | string | Required. |
| `description` | string | Short module summary. |
| `days_till_drip_access` | integer | Drip delay in days; must be > 0 when set. |
| `is_hidden_from_non_members` | boolean | Hide the module from non-members. |
| `blocker_section_id` | id | Gate this module behind another section. |
| `blocker_lesson_id` | id | Gate this module behind a specific lesson. |

Update a module, including its `title`, with
`PATCH /api/v2/courses/sections/:id` and the same `courses_section` body.
Returns **200 OK** with the updated section.

## Lessons

Create a lesson inside a section. The body is authored in Markdown via
`lesson_content`, which is rendered into the lesson's page.

```http
POST /api/v2/courses/sections/490404/lessons
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "courses_lesson": {
    "title": "Why agentic systems break in production",
    "lesson_content": "## The core failure modes\n\n- Context drift across turns\n- Tool calls that silently fail\n- Unbounded retries\n\nIn this lesson we map each failure mode to a concrete mitigation."
  }
}
```

Returns **201 Created**:

```json
{
  "id": 1407244,
  "public_id": "xGGOWg",
  "title": "Why agentic systems break in production",
  "body": null,
  "section_id": 490404,
  "published_at": null,
  "current_path": "/why-agentic-systems-break-in-production",
  "sort_order": 0,
  "created_at": "2026-06-30T16:09:52.299Z",
  "updated_at": "2026-06-30T16:09:52.434Z"
}
```

### Permitted lesson fields

| Field | Type | Notes |
|-------|------|-------|
| `title` | string | Required. |
| `lesson_content` | Markdown | Rendered into the lesson page on create. Headings, lists, and paragraphs are supported. |
| `body` | string | Plain text/HTML body field, kept separate from the rendered page content. |
| `image_id` | id | Lesson image - id of a workspace Image resource. |

Update a lesson, including its `title` or `body`, with
`PATCH /api/v2/courses/lessons/:id` and a `courses_lesson` body. Returns
**200 OK** with the updated lesson.

List a section's lessons with `GET /api/v2/courses/sections/:section_id/lessons`:

```json
[
  {
    "id": 1407244,
    "public_id": "xGGOWg",
    "title": "Why agentic systems break in production",
    "body": null,
    "section_id": 490404,
    "published_at": null,
    "current_path": "/why-agentic-systems-break-in-production",
    "sort_order": 0,
    "created_at": "2026-06-30T16:09:52.299Z",
    "updated_at": "2026-06-30T16:09:52.434Z"
  }
]
```

## Publishing

New courses, sections, and lessons start unpublished (`published_at: null`).
There are five useful publishing flows. On each direct publish endpoint, omit
`published_at` to publish now or pass a future ISO 8601 datetime to schedule it.
A past value returns **422** without changing the requested resource.

**Publish only the course record** - `POST /api/v2/courses/:id/publish` with no
body. This leaves its sections and lessons as drafts. Pass `published_at` to
schedule the course record instead of publishing it immediately. Returns **200
OK** with the course.

```http
POST /api/v2/courses/86816/publish
Authorization: Bearer <access_token>
Content-Type: application/json

{ "published_at": "2026-08-01T09:00:00Z" }
```

**Publish one lesson** - `POST /api/v2/courses/lessons/:id/publish`. This is the
only way to set a lesson's publish timestamp; `published_at` is intentionally not
writable through the lesson update endpoint.

```http
POST /api/v2/courses/lessons/1407244/publish
Authorization: Bearer <access_token>
Content-Type: application/json

{}
```

The response is the updated lesson. To schedule it instead, send a future time:

```json
{ "published_at": "2026-08-01T09:00:00Z" }
```

**Publish one section and its lessons** -
`POST /api/v2/courses/sections/:id/publish`. The section and each draft or
scheduled lesson receive the requested publish timestamp. This direct form does
not grant or notify existing enrollees; use a section publish action when you need
those options. The root section cannot be published directly.

```http
POST /api/v2/courses/sections/490404/publish
Authorization: Bearer <access_token>
Content-Type: application/json

{}
```

The **200 OK** response is the updated section:

```json
{
  "id": 490404,
  "public_id": "erqaPn",
  "title": "Course Orientation and Mental Models",
  "published_at": "2026-07-21T15:10:00.000Z"
}
```

If section or lesson validation fails, the request returns **422** and no publish
timestamps are changed.

**Recommended: publish the whole content tree in one step** - pass
`publish_sections_and_lessons: true`. The API sets the course, eligible section,
and lesson publish timestamps before returning **200 OK**. It also queues
per-section follow-up actions; those actions may still be running after the
response. The response is the course, not a progress receipt.

```http
POST /api/v2/courses/86816/publish
Authorization: Bearer <access_token>
Content-Type: application/json

{ "publish_sections_and_lessons": true }
```

To schedule the whole tree, include both a future `published_at` and
`publish_sections_and_lessons: true`. Eligible sections receive scheduled
actions and lessons receive the future course timestamp. Sections with an
intentional drip or blocker keep that access gate; their lessons are timestamped
so they are ready when the gate grants access.

**Publish selected sections (async)** - create a section publish action. Choose
exactly one selector: `target_ids` (section public ids) or `target_all: true`
(every unpublished non-root section). Passing the root section public id has the
same expansion behavior as `target_all`. Each selected section and its
unpublished or scheduled lessons are published by the background action.

```http
POST /api/v2/courses/86816/sections/publish_actions
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "courses_sections_publish_action": {
    "target_ids": ["erqaPn"],
    "auto_enroll": true,
    "send_email": false
  }
}
```

Optional fields are `auto_enroll` (grant existing enrollees access),
`send_email` (notify them, only meaningful with `auto_enroll`), and
`scheduled_for` (a future date-time). A valid request returns **201 Created**.
The response always emits resolved integer `target_ids` and `target_all: false`,
even if the request used public ids or `target_all: true`.

Fetch the receipt at
`GET /api/v2/courses/sections/publish_actions/:id`. Poll
`status`, `target_count`, `performed_count`, and `published_lessons_count`.
`status` is `queued`, `scheduled`, or `processing`
while work remains and `completed` when it is done. `published_lessons_count`
records lessons this action changed, so it excludes lessons that were already
live. List receipts at
`GET /api/v2/courses/:course_id/sections/publish_actions`.

If a root or `target_all` request finds nothing unpublished, it returns **200
OK** without creating an action:

```json
{
  "message": "no unpublished course content to publish",
  "target_count": 0,
  "performed_count": 0,
  "published_lessons_count": 0,
  "status": "completed"
}
```

## Worked example: scaffold a course with content

```http
# 1. Create the course
POST /api/v2/workspaces/365069/courses
{ "course": { "title": "Agentic Engineering for Senior Software Engineers" } }
# -> 201, course.id = 86816

# 2. Add the first module
POST /api/v2/courses/86816/sections
{ "courses_section": { "title": "Course Orientation and Mental Models" } }
# 3. Add a lesson to that module
POST /api/v2/courses/sections/490404/lessons
{ "courses_lesson": { "title": "Why agentic systems break in production",
                      "lesson_content": "## The core failure modes\n\n- Context drift\n- Silent tool failures" } }
# -> 201, lesson.id = 1407244

# Repeat 2-3 for each module and its lessons.

# 4. Publish the course, eligible modules, and lessons in one step
POST /api/v2/courses/86816/publish
{ "publish_sections_and_lessons": true }
# -> 200; course, eligible section, and lesson timestamps are now set
```

## Reading back a course

- Show the course (`GET /api/v2/courses/:id`) for its metadata and
  `root_section_id`.
- List the course's modules (`GET /api/v2/courses/:course_id/sections`). The list
  includes the root section (the `parent_id: null` entry, matching the course's
  `root_section_id`); most integrations ignore it and treat its children as the
  modules. Ordered by `id`
  ascending by default (add `?sort_order=desc` for descending); pass
  `?sort_property=sort_order` to get the builder's display order instead. Note
  `sort_order` is scoped per parent, so it is not globally unique across the list -
  the root and each parent's first child both start at 0 and are tie-broken by id.
  Every ordering is consistent with the pagination cursor, so the Pagination-Next
  walk returns each section exactly once. It paginates at 20 per page (see below).
- For each section id, list its lessons
  (`GET /api/v2/courses/sections/:section_id/lessons`). Querying lessons by a
  known `section_id` is the most reliable way to read a section's contents.
  Lessons default to `id` order; pass `?sort_property=sort_order` for the builder's
  display order within the section (`sort_order` starts at 0 per section).

### Pagination

List endpoints page at 20 records and follow the standard V2 cursor convention:
read the `Pagination-Next` response header and pass it back as the `after` query
parameter, repeating until the header is absent.

```http
GET /api/v2/courses/86816/sections
GET /api/v2/courses/86816/sections?after=490422
```

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| 404 "Not found" | The course/section/lesson id is not in the token's accessible workspace | Verify the id and the access token. |
| 400 Bad Request on a section publish action | The `courses_sections_publish_action` wrapper is missing or empty | Send the wrapper with exactly one of `target_ids` or `target_all: true`. |
| 422 Unprocessable Entity | Validation failure on permitted attributes | Read the `error` body - usually a missing `title` or invalid id reference. |
| 422 mentioning `published_at` | A publish endpoint received a malformed or past datetime | Omit it to publish now, or pass a valid future ISO 8601 datetime. |
| 422 on a section publish action | Both `target_all` and `target_ids` were sent, or a target is unknown or belongs to another course | Send exactly one selector and use ids returned for sections of this course. |
| 401 "API key missing or invalid" | Missing, revoked, or wrong token | Re-authenticate. |
