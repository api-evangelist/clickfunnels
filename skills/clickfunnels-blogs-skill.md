---
name: clickfunnels-blogs
description: >
  Manage ClickFunnels 2.0 blog posts programmatically - create, list, update,
  and delete posts; manage authors, categories, and tags. The post body is
  authored in PML (Page Markup Language); the DSL itself lives in the Page
  Markup Skill. Post writes require trusted platform access only when a third-party
  platform acts on another team's workspaces; acting on your own account does not.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/page-markup/skill.md
    description: PML reference - elements, attributes, design system, examples. Read this before authoring any `markup` field.
  - path: /.well-known/pages/skill.md
    description: Companion API for funnel/site pages (same PML body, different surface).
---

# ClickFunnels Blogs Skill

The HTTP surface for managing CF2 blog posts and their supporting taxonomy
(authors, categories, tags). Post bodies are authored in
**PML (Page Markup Language)** - for the DSL itself, see the
[Page Markup Skill](/.well-known/page-markup/skill.md). This document covers
discovery, the post lifecycle, and the gotchas worth knowing.

## When to use this skill

- Publishing a new blog post (article) from a brief or prompt
- Listing or reading existing posts and their PML
- Updating a post's body, metadata, or publish state
- Creating tags and assigning them to posts
- Discovering authors and categories to attach to posts

## A note on blogs vs. blog posts

Every CF2 site ships with a default blog ("Primary Blog"). **You cannot create
a new blog itself via the public API** - blogs are provisioned automatically.
What you can create via the API is a **blog post** (an article) inside an
existing blog. This skill is almost entirely about that post lifecycle.

If you need a new blog container in a workspace, do it through the admin UI
("Sites & Blogs" -> "Blogs") before calling the API.

## Authentication

OAuth2 password grant using OAuth application credentials. See
[Pages Skill -> Authentication](/.well-known/pages/skill.md#authentication) for
the full handshake - the same access token works for the blogs surface.

> **When is trusted platform access required?** Only when a **third-party** platform
> acts on workspaces owned by **other** teams. Acting on your own account (your own API
> key, or an OAuth app within its own team's workspaces) never needs it; those writes
> always pass, and reads are never gated.

## Discovering your blog

Every blog post is nested under a blog; you need a `blog_id` to create posts.
List blogs in a workspace:

```
GET /api/v2/workspaces/:workspace_id/blogs
Authorization: Bearer <access_token>
```

Returns the array of blogs in the workspace. Most workspaces have exactly one
(the "Primary Blog" provisioned with the site). Capture `id` and use it as
`:blog_id` in every endpoint below.

Show a single blog:
```
GET /api/v2/blogs/:id
```

### Update blog metadata

```
PATCH /api/v2/blogs/:id
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "blog": {
    "name": "Engineering Notes",
    "seo_title": "Engineering Notes - ClickFunnels",
    "seo_description": "What we're building, what we're learning.",
    "seo_image_id": 42,
    "theme_id": 5
  }
}
```

Returns **200 OK** with the updated blog. Permitted fields: `name`,
`seo_title`, `seo_description`, `seo_image_id`, `theme_id`.

### Configure blog appearance with existing Theme and Page APIs

There is no separate blog-appearance resource. A Blog selects a Theme with
`theme_id`, and that Theme supplies the Pages used for the blog index, posts,
and categories. `GET /api/v2/blogs/:id` returns the effective Page IDs directly:

- `resolved_template_page_id` - blog index
- `resolved_post_template_page_id` - individual posts
- `resolved_category_template_page_id` - category archives

List the workspace's Themes and choose one whose `template_pages` response has
a non-null `blogs_post_template_page`. Then set that Blog Theme and read or
update its effective Pages with the Pages API. If the Blog already resolves the
Pages you want, skip the Theme change and update those resolved Pages directly.

```http
GET /api/v2/workspaces/:workspace_id/themes

PATCH /api/v2/blogs/:id
Content-Type: application/json

{"blog":{"theme_id":5}}

GET /api/v2/blogs/:id
GET /api/v2/pages/:resolved_post_template_page_id?expand[]=custom_css

PATCH /api/v2/pages/:resolved_post_template_page_id
Content-Type: application/json

{
  "page": {
    "custom_css": "article { max-width: 72ch; margin-inline: auto; }",
    "custom_css_mode": "append"
  }
}
```

The Blog model already rejects Themes outside the workspace and Themes without
Blog templates. A resolved template Page can be shared by more than one Blog or
Theme, so confirm its consumers before changing its CSS. Read the current CSS
first. Use `replace` only when sending the complete replacement stylesheet;
use `append` for a new scoped rule.

## Supporting taxonomy

These return ids you can pass to `author_ids`, `category_ids`, and `tag_ids`
when creating or updating posts.

### Authors (read-only)

```
GET /api/v2/blogs/:blog_id/authors          # list
GET /api/v2/blogs/authors/:id               # show
```

Authors are managed through the admin UI; this surface is read-only.

### Categories (read-only)

```
GET /api/v2/blogs/:blog_id/categories       # list
GET /api/v2/blogs/categories/:id            # show
```

Categories are managed through the admin UI; this surface is read-only.

### Tags (list / show / create)

```
GET  /api/v2/blogs/:blog_id/tags            # list
GET  /api/v2/blogs/tags/:id                 # show
POST /api/v2/blogs/:blog_id/tags            # create
```

Create a tag:

```http
POST /api/v2/blogs/:blog_id/tags
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "blogs_tag": {
    "name": "engineering"
  }
}
```

Returns **201 Created** with the new tag (capture its `id` for `tag_ids` on posts).

## Creating a blog post

This is the core operation. The post body lives in the `markup` field as PML.

```http
POST /api/v2/blogs/:blog_id/posts
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "blogs_post": {
    "title": "Becoming an Agentic Engineer in 2026",
    "visibility": "public",
    "published_at": "2026-05-12T14:00:00Z",
    "excerpt_content": "Notes from a year of letting agents take the keyboard.",
    "seo_title": "Becoming an Agentic Engineer in 2026: A Field Guide",
    "seo_description": "A field guide to the skills that compound in 2026.",
    "image_id": 17,
    "author_ids": [3],
    "category_ids": [1],
    "tag_ids": [5, 8],
    "markup": "<section bg=\"#ffffff\" padding=\"96 24\"><row><column><headline>Hello</headline></column></row></section>"
  }
}
```

On success: **201 Created** with the post resource. Pass `?expand[]=markup` to
also receive the round-tripped PML in the response. On invalid markup: **400 Bad
Request** with the PML parser error - no post is created.

### Permitted fields

| Field | Type | Notes |
|-------|------|-------|
| `title` | string | The post title (required for usable URLs - slug is derived from it). |
| `visibility` | enum | `"draft"` or `"public"` - see the gotcha below. Default `"draft"`. |
| `published_at` | ISO8601 | When the post becomes visible publicly. See the gotcha below. |
| `excerpt_content` | string | Short summary used in indexes and previews. |
| `seo_title` | string | `<title>` tag override. Defaults to `title` if omitted. |
| `seo_description` | string | Meta description. |
| `image_id` | id | Featured image - id of a workspace Image resource. |
| `author_ids` | array of ids | Authors attached to the post (see Authors). |
| `category_ids` | array of ids | Categories (see Categories). |
| `tag_ids` | array of ids | Tags (see Tags). |
| `markup` | PML string | Post body. See [Page Markup Skill](/.well-known/page-markup/skill.md). |
| `post_content` | Markdown | **Deprecated.** Older Markdown-based authoring path kept for back-compat with the live MCP `create_blog_post` tool. New consumers should use `markup`. When both are passed, `markup` wins. |

### Rehost source images during a migration

The Images API already accepts a remote source URL. Rehost each source image
before creating the post, then use the returned ClickFunnels URL in the body:

```http
POST /api/v2/workspaces/:workspace_id/images
Content-Type: application/json

{
  "image": {
    "upload_source_url": "https://source.example/articles/diagram.png",
    "name": "Architecture diagram",
    "alt_text": "Services connected through an event bus"
  }
}
```

The response contains both `id` and `url`. Use `id` as the post's `image_id`
only when this is the featured image. For an image in the article body, replace
the source URL with the returned `url`, for example
`<image src="https://images.example/hosted.png" alt="Architecture diagram"/>`.
Keep a source-URL-to-hosted-URL map so repeated images are uploaded once, and do
not create the post until all required uploads succeed.

! **FORBIDDEN - never encode a nested path into a post's title or slug.** A blog
post's slug is derived **automatically** from its `title`, and its full public
URL is composed by ClickFunnels from the blog/post hierarchy. Never prefix the
blog's path (or any other parent path) onto the post - the post's slug must stand
on its own, and you must **never** guess or build the composed URL yourself.
e.g. for a blog served at path `recipes`, a post titled "Catering" must **not** be
given a title/slug like `recipes/catering`; keep it as its own standalone slug
(`catering`) and the platform composes the full nested URL for you.

### Two gotchas worth knowing

**1. `visibility: "public"`, not `"published"`.** The model uses `"public"` as
the canonical value for a visible post (`Blogs::Post::PUBLIC_STATUS = "public"`).
`"published"` is the name of an ActiveRecord scope, not a column value - sending
it as a string is accepted by the API but produces a half-published row whose
admin index shows a grey badge with no date. Always use `"public"`.

**2. `published_at` is `before_create` only.** The model auto-stamps
`published_at = Time.zone.now` on **create** when `visibility == "public"`. On
**update** the callback does not fire, so flipping a draft to published via
`PATCH` requires sending `published_at` explicitly:

```http
PATCH /api/v2/blogs/posts/:id
Content-Type: application/json

{
  "blogs_post": {
    "visibility": "public",
    "published_at": "2026-05-12T14:34:40Z"
  }
}
```

## Reading a blog post

```
GET /api/v2/blogs/posts/:id
GET /api/v2/blogs/posts/:id?expand[]=markup
```

`markup` is opt-in to keep show/list responses small. Without `expand[]=markup`,
the response excludes the field entirely.

### Listing posts for a blog

```
GET /api/v2/blogs/:blog_id/posts
GET /api/v2/blogs/:blog_id/posts?expand[]=markup
```

Returns the array of posts in the blog. Pagination, filtering, and sorting
follow the standard V2 conventions.

## Updating a blog post

**STOP before you send `markup` here.** A `markup` write replaces the post's
entire body, published version and editor draft alike, and PML covers only a
subset of what the ClickFunnels editor can build. If this post could have been
written or edited in the editor, ask the person for explicit approval first and
tell them editor-only content will not survive. "Update the post" is not that
approval. The full rule, including the cases where you do not need to ask, is in
[STOP: get approval before overwriting an existing page](/.well-known/page-markup/skill.md#stop-get-approval-before-overwriting-an-existing-page).
Updating only metadata (title, SEO fields, taxonomy, visibility) is not affected:
leave `markup` out of the payload and the body is untouched.

```http
PUT /api/v2/blogs/posts/:id
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "blogs_post": {
    "title": "Updated title",
    "markup": "<section><row><column><headline>Updated body</headline></column></row></section>"
  }
}
```

Returns **200 OK**. All permitted fields above are accepted on update. Remember
the `published_at` gotcha above - explicit value required to publish a draft via
update.

## Deleting a blog post

```
DELETE /api/v2/blogs/posts/:id
Authorization: Bearer <access_token>
```

Returns **204 No Content** on success.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| 400 "Markup is not valid PML: ..." | Invalid DSL | See [Page Markup Skill -> Error Reference](/.well-known/page-markup/skill.md#18-error-reference) |
| 401 "API key missing or invalid" | Missing, revoked, or wrong token | Re-authenticate |
| 403 Forbidden | A third-party platform writing on another team's workspace without trusted platform access (first-party / own-account writes are never blocked) | Request trusted platform access via your CF integration owner |
| 404 "Not found" | Blog or post does not exist, or token does not have access | Check ids and authorization |
| 422 Unprocessable Entity | Validation failure on permitted attributes | Read the `error` body - usually a missing title or invalid id reference |
