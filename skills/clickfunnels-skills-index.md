# ClickFunnels Skills Index

Skills are structured references that AI agents can use to interact with the ClickFunnels platform programmatically.

## Available Skills

### Page Markup (PML)

Reference for the XML-like DSL used to author the `markup` field on any CF2 surface (pages and blog posts). Elements, attributes, design system, examples. Writing `markup` replaces the whole page and PML covers only a subset of what the ClickFunnels editor can build, so read the approval guardrail before touching a page that already exists.

- [Page Markup Skill](/.well-known/page-markup/skill.md)
- [Approval guardrail - overwriting an existing page](/.well-known/page-markup/skill.md#stop-get-approval-before-overwriting-an-existing-page)

### Pages

Create and update CF2 funnel/site pages programmatically. Page body is authored in PML - from a single section to a full multi-page funnel.

- [Pages Skill](/.well-known/pages/skill.md)
- [API Reference - Create Page](https://developers.myclickfunnels.com/reference/createpage.md)
- [API Reference - Update Page](https://developers.myclickfunnels.com/reference/updatepages.md)

### Blogs

Create, list, and update CF2 blog posts. Post body is authored in PML.

- [Blogs Skill](/.well-known/blogs/skill.md)
- [Import External Blog Posts Skill](/.well-known/blogs/import-external-post/skill.md)
- [API Reference - List Blog Posts](https://developers.myclickfunnels.com/reference/listblogsposts.md)
- [API Reference - Create Blog Post](https://developers.myclickfunnels.com/reference/createblogpost.md)
- [API Reference - Update Blog Post](https://developers.myclickfunnels.com/reference/updateblogpost.md)

### Courses

Create CF2 courses and their content tree - a course, its sections (modules), and lessons whose body is authored in Markdown.

- [Courses Skill](/.well-known/courses/skill.md)

### Refine Filters

Define reusable contact filters and apply existing order-filter tokens when listing orders.

- [Refine Filters Skill](/.well-known/refine-filters/skill.md)
- [API Reference - Create Filter](https://developers.myclickfunnels.com/reference/createrefinefilter.md)
- [API Reference - List Filters](https://developers.myclickfunnels.com/reference/listrefinefilters.md)

### Products & Discounts

Create and manage sellable products, their prices, and discounts - the implicit default variant, one-time / subscription / payment-plan prices, compare-at pricing, safe repricing once orders exist, discount codes (percentage or fixed amount, workspace-wide or scoped), and attaching products to funnel pages.

- [Products & Discounts Skill](/.well-known/products/skill.md)
- [API Reference - Create Product](https://developers.myclickfunnels.com/reference/createproducts.md)
- [API Reference - Create Price](https://developers.myclickfunnels.com/reference/createproductsprices.md)
- [API Reference - Update Price](https://developers.myclickfunnels.com/reference/updateproductsprices.md)
- [API Reference - Create Discount](https://developers.myclickfunnels.com/reference/creatediscounts.md)

### Subscription Changes

Discover, preview, and commit variant/price changes on existing subscription line items.

- [Subscription Changes Skill](/.well-known/subscription-changes/skill.md)
- [API Reference - List Change Options](https://developers.myclickfunnels.com/reference/listproductspriceschangeoptions.md)
- [API Reference - Preview Change](https://developers.myclickfunnels.com/reference/previewordersLineitemschange.md)
- [API Reference - Commit Change](https://developers.myclickfunnels.com/reference/performordersLineitemschange.md)

### Orders

Manage the lifecycle of an existing order - cancel a Payments AI or Stripe subscription (immediately or at period end), refund a charge in full, and abandon an unpaid invoice to stop its automatic payment retries.

- [Orders Skill](/.well-known/orders/skill.md)
- [API Reference - Cancel Order](https://developers.myclickfunnels.com/reference/cancelorder.md)
- [API Reference - Refund Transaction](https://developers.myclickfunnels.com/reference/refundorderstransactions.md)
- [API Reference - Abandon Invoice](https://developers.myclickfunnels.com/reference/abandonordersinvoices.md)

### Funnels

Build and modify funnel workflows programmatically - split tests, conditional splits, branch positioning, multi-page branches.

- [Funnels Skill](/.well-known/funnels/skill.md)

### Workflows

Build marketing-automation workflows programmatically - triggers, ordered action steps (send email, tag, set attribute, create opportunity, grant course/community access, webhook, delay), conditional and A/B split branching, enabling, and manually enrolling contacts.

- [Workflows Skill](/.well-known/workflows/skill.md)

### Emails

Configure the workspace business mailing address and manage marketing email programmatically - email templates (create, update, delete from `html_body`/`text_body`), broadcasts, sending domains (provision and verify DKIM/SPF/DMARC), the sending addresses on them, and the inline-HTML body shortcut on a workflow send-email step.

- [Emails Skill](/.well-known/emails/skill.md)
- [API Reference - Create Email Template](https://developers.myclickfunnels.com/reference/createemailstemplate.md)
- [API Reference - Create Email Broadcast](https://developers.myclickfunnels.com/reference/createemailsbroadcast.md)

### SDK - External Pages

Add the ClickFunnels SDK to a page you host yourself (own domain, static host, Netlify/Vercel, or `file://`) so it tracks visits, captures opt-ins, and carries contact identity across funnel steps - using the `cf-page-token` meta tag, the `sdk.js` script, and `data-cf-element` form markup. The page is registered with ClickFunnels as an external page - placed in a funnel as a step, or standalone (registered for tracking, placed later, e.g. into a split branch). Checkout turns such a page into an order form: products, order bumps, one-click one-time offers, and gateway payment UI.

- [Create an External Page Skill](/.well-known/sdk/create-external-page/skill.md)
- [Migrate an External Page Domain Skill](/.well-known/sdk/migrate-external-page-domain/skill.md)
- [Add Checkout to an External Page Skill](/.well-known/sdk/add-checkout/skill.md)
- [API Reference - Funnel Structure](https://developers.myclickfunnels.com/reference/getfunnelstructure.md)
- [API Reference - Update Page](https://developers.myclickfunnels.com/reference/updatepages.md)

### Stats

Read aggregated funnel and page analytics - views, opt-ins, sales, earnings - over a configurable timerange. Designed for recurring audit agents that score funnels daily and suggest concrete next steps.

- [Stats Skill](/.well-known/stats/skill.md)
- [API Reference - Funnel Stats](https://developers.myclickfunnels.com/reference/getfunnelstats.md)
- [API Reference - Page Stats](https://developers.myclickfunnels.com/reference/getpagestats.md)

## Resources

- [ClickFunnels Product Overview](/llms.txt)
- [Developer Hub](https://developers.myclickfunnels.com): Full API documentation, guides, and reference
- [Support Center](https://support.myclickfunnels.com): Help docs, guides, and tutorials
