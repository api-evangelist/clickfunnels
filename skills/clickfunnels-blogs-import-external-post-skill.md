---
name: clickfunnels-import-external-blog-post
description: >
  Copy one or more selected articles from an RSS feed, Atom feed, or public web
  page into an existing ClickFunnels blog as verified drafts. Use when migrating
  external blog content while preserving source metadata and avoiding duplicate
  posts.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index for the ClickFunnels API.
  - path: /.well-known/blogs/skill.md
    description: Blog and blog-post endpoint reference, fields, and publication rules.
  - path: /.well-known/page-markup/skill.md
    description: PML reference for the markup field used by blog posts.
  - path: /.well-known/pages/skill.md
    description: Page API reference for template-level custom CSS and head code.
---

# Import an External Blog Post

This workflow copies one or more articles from an external site into an existing
ClickFunnels blog. It discovers the source content, converts HTML into valid
PML, creates drafts one at a time, and verifies each result before publication.

Import means copy. The ClickFunnels API does not modify, redirect, unpublish,
or delete the source article.

## When to use this skill

- Import the newest article from an RSS or Atom feed
- Copy a specific public article into a ClickFunnels blog
- Migrate the exact remaining entries from a fixed feed snapshot
- Preserve an original title, excerpt, and publication timestamp
- Run an idempotent migration without creating duplicate posts
- Review migrated content as a draft before publishing it

## Authentication

Send a ClickFunnels OAuth access token on every ClickFunnels API request:

```http
Authorization: Bearer <access_token>
```

The public source feed or article usually does not require ClickFunnels
authentication. If the source is private, obtain separate permission and
credentials for that source.

## Endpoint reference

| Action | Method | Path |
|--------|--------|------|
| List destination blogs | GET | `/api/v2/workspaces/:workspace_id/blogs` |
| List existing posts | GET | `/api/v2/blogs/:blog_id/posts` |
| Create a draft | POST | `/api/v2/blogs/:blog_id/posts` |
| Verify a post and its markup | GET | `/api/v2/blogs/posts/:id?expand[]=markup` |
| Publish a verified draft | PATCH | `/api/v2/blogs/posts/:id` |
| Delete a confirmed destination post | DELETE | `/api/v2/blogs/posts/:id` |
| Style a destination template page | PATCH | `/api/v2/pages/:id` |

## Safety rules

1. Create with `visibility: "draft"` unless the user explicitly requests
   immediate publication.
2. Do not send a `slug` field. ClickFunnels derives the post path from `title`.
3. Do not delete or redirect the source article as part of this workflow.
4. Do not publish until metadata, markup, links, and images have been verified.
5. Treat source HTML as untrusted input. Normalizing HTML is not sanitizing it;
   remove or explicitly review executable content before importing it.
6. Treat draft as an editorial state, not a confidentiality boundary. Do not
   put secrets in a draft post.
7. Treat a shared blog template change as a separate operation. Require explicit
   approval for the exact template Page and code change before updating it.
8. Import into a new post. This workflow only ever creates posts, which is safe:
   there is no existing body to lose. The moment you instead write `markup` to a
   post that already exists, you are replacing its whole body with a PML subset
   of what the editor can build, so require explicit approval first. See
   [STOP: get approval before overwriting an existing page](/.well-known/page-markup/skill.md#stop-get-approval-before-overwriting-an-existing-page).

## 1. Discover the source feed

When given a site or article URL, fetch its HTML and inspect the document head
for an alternate feed link:

```html
<link rel="alternate" type="application/rss+xml" href="/feed.xml"/>
<link rel="alternate" type="application/atom+xml" href="/atom.xml"/>
```

Resolve a relative `href` against the page URL. If no alternate link exists,
look for a feed URL advertised by the site. Common paths such as `/feed/`,
`/rss.xml`, and `/atom.xml` are useful probes, but do not assume that one is
authoritative without checking its contents.

If there is no usable feed, accept a specific article URL and extract the main
article body from that page. Avoid importing navigation, cookie banners,
related-post cards, or the site footer.

## 2. Select the latest article

Do not assume feed order is chronological. Parse every candidate item and sort
by its best available timestamp:

- RSS: `pubDate`, then a namespaced date field
- Atom: `published`, then `updated`
- Missing dates: use feed order and report that the date could not be verified

For the selected item, capture:

- Stable source key: canonical article URL, falling back to the feed GUID or ID
- Title
- Canonical article URL
- Publication timestamp
- Excerpt or summary
- Full article HTML

For RSS, prefer `content:encoded` over `description`. For Atom, prefer
`content` over `summary`. Decode the feed's CDATA or escaped HTML before the
next step. Do not nest the source CDATA wrapper inside the PML wrapper.

Some feeds contain only a summary. In that case, fetch the canonical article
URL and extract its article body. If a complete article cannot be identified
reliably, stop before creating a destination post.

## 3. Normalize external HTML for PML

The `markup` field accepts PML, not an arbitrary HTML document. For a faithful
one-post migration, place the normalized article body in a top-level
`<customhtml>` element:

```xml
<customhtml><![CDATA[<article><h2>A practical migration</h2><p>Verify the draft before publishing it.</p><img src="https://media.example/article.jpg" alt="Notebook"/></article>]]></customhtml>
```

Although CDATA protects the HTML while PML is parsed, the custom HTML is parsed
again when the page tree is built. The content must therefore be well-formed
XHTML. Use an HTML parser and serializer rather than regular expressions:

- Close every non-void element
- Self-close void elements such as `br`, `hr`, `img`, `input`, and `source`
- Quote every attribute value
- Serialize true HTML boolean attributes as repeated values, such as
  `async="async"` or `disabled="disabled"`
- Serialize a non-boolean valueless attribute as an empty value, such as
  `data-note=""`, or remove it
- Escape bare ampersands and invalid text characters
- Remove malformed closing tags
- Rewrite relative `href`, `src`, and `srcset` values as absolute URLs
- Remove scripts, inline event handlers, forms, and other executable content
  unless the user explicitly wants a reviewed embed preserved
- Replace each literal `]]>` with `]]]]><![CDATA[>` before wrapping it

`customhtml` preserves source links and can also preserve scripts and embed
code. The normalization step does not make that code safe or inert. Strip it by
default and keep only code whose behavior and external requests were reviewed.

Choose the representation based on the content:

- Prefer native PML elements for simple headings, paragraphs, lists, links, and
  images that should remain easy to edit in ClickFunnels.
- For fidelity-heavy HTML, inline links, or reviewed embeds, use one stable
  wrapper inside `customhtml` and scope the imported styles to that wrapper.

### Rendering fidelity

Preserving the complete HTML structure does not preserve the source site's
appearance. The source theme CSS is usually absent, while destination theme
styles and resets can change typography, spacing, links, media, and embeds.
Include explicit, wrapper-scoped CSS for the imported article when fidelity
matters:

```xml
<customhtml><![CDATA[<style>
.imported-article { font: 400 18px/1.7 sans-serif; }
.imported-article h2 { margin: 2rem 0 0.75rem; }
.imported-article p { margin: 0 0 1.25rem; }
.imported-article a { color: #2457d6; text-decoration: underline; }
.imported-article img { display: block; max-width: 100%; height: auto; }
.imported-article blockquote { margin: 1.5rem 0; padding-left: 1rem; border-left: 4px solid #cccccc; }
.imported-article iframe { display: block; width: 100%; max-width: 100%; }
</style><article class="imported-article">...</article>]]></customhtml>
```

Use a wrapper name that is stable and specific to the imported content. Style
headings, paragraphs, lists, links, images, blockquotes, tables, and embeds that
the article actually contains. Remove empty source placeholders during
normalization; if a runtime placeholder must remain, hide it with a stable,
scoped selector. Deduplicate reviewed external embed loaders so each script URL
is loaded at most once.

Scope responsive height or aspect-ratio rules to a reviewed media provider or an
explicit importer-owned class, not every iframe under a generic embed-card class.
Third-party loaders can inject frames that calculate their own height; wait for each
loader to settle during rendered verification and confirm its frame is not clipped.

An API round trip proves that markup parsed, not that it renders well. Inspect
the actual rendered post at desktop and narrow widths. Check typography,
spacing, link behavior, image sizing, blockquotes, and every retained embed.

### Blog template styling is separate

The post `markup` field controls the article body. Stock blog-template chrome,
including sidebars and template placeholders, lives outside that body on the
blog template Page. If those elements need styling or empty-state cleanup,
they can be updated through that template Page's `custom_css` or `head_code`
using the
[Update Page API](https://developers.myclickfunnels.com/reference/updatepages.md).

Template Pages are shared presentation. This mutation is not part of the normal
post import. Before making it:

1. Obtain separate, explicit approval for the proposed shared-template change.
2. Show and confirm the exact template Page `id` and name, and verify that it is
   the selected blog-post template for the destination. If its identity cannot
   be proven, stop and leave the template unchanged.
3. Read the existing `custom_css` and `head_code`; do not append a duplicate
   style block or external loader.
4. Use stable, narrowly scoped selectors and additive mode. Set
   `custom_css_mode: "append"` or `head_code_mode: "append"` explicitly.
5. Capture a rendered baseline and confirm that the imported post and an
   existing post using the template can be checked after the change.
6. Update only the approved code field. Never include the template's `markup`
   in this request.

Use the Page Editor, rather than generated CSS text or DOM-rewriting JavaScript,
for semantic template content such as headings, biographies, CTA copy, links,
and images.

`append` adds the new value after the current value and is the default for both
fields. `replace` overwrites the entire field, so do not use it in this import
workflow. `custom_css` is raw CSS without a `<style>` wrapper; `head_code` must
include any required HTML wrapper such as `<style>` or `<script>`.

After approval and identity confirmation, read the current code fields and send
only the additive change:

```http
GET /api/v2/pages/:template_page_id?expand[]=custom_css&expand[]=head_code
Authorization: Bearer <access_token>

PATCH /api/v2/pages/:template_page_id?expand[]=custom_css
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "page": {
    "custom_css": ".stable-template-selector { /* reviewed template styles */ }",
    "custom_css_mode": "append"
  }
}
```

Do not fetch a blog template Page's serialized `markup` and send it back merely
to preserve the template. Updating `markup` replaces the page's visual tree,
and serialized PML may not represent every dynamic blog element. When changing
only `custom_css` or `head_code`, omit `markup` from the update entirely.

Read the Page back with `expand[]=custom_css` and compare the stored result with
the intended code. An unexpanded Page response can omit that field, and API
read-back alone does not prove that the public template is using the latest
editor version. If storage is correct but the public template remains unchanged,
open that exact template Page in the supported Page Editor, review its code and
preview, and use the editor's Save flow. Use **Save Anyway** only after confirming
that any reported validation issues are pre-existing or unrelated to the reviewed
change and the dynamic blog elements remain intact.

After the update, inspect the imported post and at least one existing post that
uses the same template at desktop and narrow widths. Confirm that navigation,
sidebar content, placeholders, article layout, and embeds still render as
expected. If those regression checks cannot be performed, omit the shared
template mutation.

### Image caveat

An external `<img>` remains hosted by the source site. It can break when the
source changes, and it may create unwanted hotlinking. Re-host body images and
rewrite their URLs before retiring the source. The blog post's `image_id` field
sets the featured image only; it does not migrate images embedded in the body.

## 4. Discover the destination and check for duplicates

List blogs and choose the intended destination:

```http
GET /api/v2/workspaces/:workspace_id/blogs
Authorization: Bearer <access_token>
```

Then list all posts in the selected blog, following pagination until complete:

```http
GET /api/v2/blogs/:blog_id/posts
Authorization: Bearer <access_token>
```

Use the source canonical URL as the durable idempotency key in the importing
client's state, mapped to the returned destination post `id` and `public_id`.
If that state is unavailable, compare the exact normalized title and the final
segment of existing `current_path` values. Treat a match as a possible duplicate
and verify it instead of creating another post.

ClickFunnels derives `current_path` from `title` when the post is created.
There is no supported `slug` input on this endpoint, so do not send one and do
not build the final public URL yourself. Use the `current_path` and `url` values
returned by the API.

## 5. Create a draft

Preserve the original `published_at` when chronology matters, but keep the new
post in draft state. Omit author, category, tag, and featured-image IDs unless
they have been explicitly mapped to resources in the destination blog.

```http
POST /api/v2/blogs/:blog_id/posts?expand[]=markup
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "blogs_post": {
    "title": "A practical migration",
    "visibility": "draft",
    "published_at": "2026-06-14T09:30:00Z",
    "excerpt_content": "A short guide to safely migrating an existing article.",
    "seo_title": "A practical migration",
    "seo_description": "How to copy and verify an existing article before publishing it.",
    "markup": "<customhtml><![CDATA[<article><h2>A practical migration</h2><p>Verify the draft before publishing it.</p><img src=\"https://media.example/article.jpg\" alt=\"Notebook\"/></article>]]></customhtml>"
  }
}
```

A successful response has this shape:

```json
{
  "id": 123,
  "public_id": "AbCdEf",
  "title": "A practical migration",
  "blog_id": 45,
  "published_at": "2026-06-14T09:30:00.000Z",
  "current_path": "/a-practical-migration",
  "visibility": "draft",
  "excerpt_content": "A short guide to safely migrating an existing article.",
  "seo_title": "A practical migration",
  "seo_description": "How to copy and verify an existing article before publishing it.",
  "image_id": null,
  "created_at": "2026-07-01T12:00:00.000Z",
  "updated_at": "2026-07-01T12:00:00.000Z",
  "url": "https://workspace.example/blog/a-practical-migration",
  "image_url": null,
  "categories": [],
  "tags": [],
  "authors": [],
  "markup": "<customhtml>\n<article><h2>A practical migration</h2><p>Verify the draft before publishing it.</p><img src=\"https://media.example/article.jpg\" alt=\"Notebook\"/></article>\n</customhtml>"
}
```

The returned markup is normalized PML. The round trip can remove the CDATA
wrapper, add generated classes or element IDs, normalize void elements, and
change entity representation. Verify durable content rather than byte-for-byte
or XML-tree equality: key text, link targets, media URLs, and response fields.

## 6. Verify the draft

Fetch the created post using the returned `id`:

```http
GET /api/v2/blogs/posts/123?expand[]=markup
Authorization: Bearer <access_token>
```

Confirm all of the following:

- `visibility` is `draft`
- `title`, excerpt, SEO fields, and source timestamp are correct
- `current_path` and `url` were returned by ClickFunnels
- `markup` contains the complete article, not only its feed summary
- Relative links and image URLs are now absolute
- No unreviewed scripts, forms, or inline event handlers remain
- Important text, links, headings, lists, and images survived normalization
- The actual rendered post has readable typography, spacing, media, and embeds
- Desktop and narrow-width layouts both match the intended article hierarchy

A draft is normally omitted from the public blog index, but its exact generated
URL can still be readable without authentication in some site configurations.
Fetch the returned `url` in an unauthenticated session and record the result.
Do not rely on draft status to protect confidential content. Use the expanded
markup and the ClickFunnels editor preview for editorial review.

If verification fails, leave the item as a draft while correcting it. Delete
the destination draft only when the user explicitly requests rollback. Do not
change the source article.

## 7. Publish separately

Publishing is a separate, user-visible action. After explicit approval, send
both `visibility` and `published_at`. A draft-to-public update does not generate
a publication timestamp automatically. Publishing a blog post and publishing a
template version are independent actions; neither publishes the other.

```http
PATCH /api/v2/blogs/posts/123
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "blogs_post": {
    "visibility": "public",
    "published_at": "2026-06-14T09:30:00Z"
  }
}
```

Use the original timestamp to preserve chronology, or use the current time when
the destination should treat the article as newly published. Fetch the post
again and confirm `visibility: "public"` before sharing its returned `url`.

### Optional cleanup of confirmed example posts

Treat destination cleanup as a separate destructive operation. First list and
paginate every post in the selected blog. Present each candidate's exact returned
`id`, title, `current_path`, URL, and visibility. Delete only posts whose stock or
example status and exact identity the user has explicitly confirmed; do not infer
that status from age, ordering, draft state, or theme alone.

After approval for the exact IDs:

```http
DELETE /api/v2/blogs/posts/:id
Authorization: Bearer <access_token>
```

Success is `204 No Content`; an empty response body is expected. Deletion is
irreversible through the API. After deletion, list and paginate every post again
and confirm the deleted IDs are absent and every intended post remains. Then fetch
the blog's returned public URL without authentication and verify that intended
public posts appear, removed examples do not, and intended post URLs load. Stop the
batch if any verification fails; do not delete another post.

## Worked example: migrate a fixed feed snapshot

A representative batch starts with one captured feed containing six entries and a
destination containing one confirmed match. "Import all other posts" means the exact
five-entry complement at capture time, not a moving query that can include later feed
changes.

1. Fetch the feed once and record its final URL and content hash.
2. Paginate the destination, match the existing post, and confirm the exact five
   entries to create before writing.
3. Build an oldest-to-newest manifest with each source key, canonical URL, title,
   excerpt, timestamp, and media inventory.
4. Normalize each body with an HTML parser, resolve relative URLs, remove unreviewed
   executable content, and wrap fidelity-heavy HTML in scoped article styles.
5. Preserve reviewed callouts, bookmark links, external images, and YouTube frames.
   Keep one reviewed X loader and disclose externally hosted images.
6. Apply dimensions only through provider-specific or importer-owned selectors;
   preserve portrait video ratios without resizing dynamically injected X frames.
7. Create one draft, read it back with `expand[]=markup`, and compare durable text,
   URLs, media counts, timestamp, path, and visibility before advancing.
8. Render the returned URL, wait for loaders to settle, and verify typography, cards,
   images, video ratios, and injected-frame height at desktop and narrow widths.
9. Paginate again and confirm one original plus five new drafts with no duplicates.
   Report returned URLs, unchanged source content, and externally hosted media.
10. Treat publication as a second approved batch with explicit `published_at`, and
    deletion of confirmed example posts as a separate irreversible approval.

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| No feed discovered | The site does not advertise RSS or Atom | Use a confirmed feed URL or import a specific article page |
| Feed item contains only a summary | The publisher omits full content from feeds | Fetch and extract the canonical article body before creating the draft |
| 400 markup error | Raw HTML was sent as PML, or custom HTML is not well-formed XHTML | Normalize with an HTML parser and wrap it in top-level `customhtml` |
| Duplicate destination post | The source URL was imported before, or a matching title/path exists | Verify and update the existing post instead of creating another |
| 401 API key missing or invalid | Access token is missing, revoked, or for another workspace | Re-authenticate and retry with the intended workspace token |
| 403 Forbidden | The API denied the application or operation | Follow the returned 403 instructions and the [Blogs Skill authentication guidance](/.well-known/blogs/skill.md#authentication); request developer-platform access when directed |
| 404 Not found | The blog or post is outside the token's workspace | Re-discover the destination blog with the same token |
| 422 Unprocessable Entity | A required value or referenced resource ID is invalid | Read the error body and remove unmapped author, category, tag, or image IDs |
| Draft content is readable by exact URL | Draft status removed the post from normal indexes but did not make it private | Do not import confidential content; verify the exact URL before relying on its visibility |
| Complete markup renders poorly | Source-theme CSS is absent or destination styles override it | Add explicit wrapper-scoped article CSS and verify the rendered page at multiple widths |
| Empty sidebar or template placeholder remains | The placeholder belongs to the blog template, not the post body | Style or hide it through template Page code fields; do not replace the template markup |
| Embed runs more than once | The source included duplicate loader scripts | Keep one reviewed loader per external script URL |
| Images disappear later | Body images still point at the source host | Re-host them and update the body URLs before retiring the source |
