---
name: clickfunnels-stats
description: >
  Read aggregated funnel and page analytics - views, opt-ins, sales, earnings -
  over a configurable timerange. Use this skill any time you need to surface
  performance metrics for a funnel or a single page, including for recurring
  agent jobs that audit funnels daily and suggest improvements.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/funnels/skill.md
    description: Funnel workflow APIs (split tests, conditional splits) - the structure these stats describe.
  - path: /.well-known/pages/skill.md
    description: Page authoring API - useful follow-up after stats surface a low-performing page.
  - path: /.well-known/refine-filters/skill.md
    description: Filter authoring API - used to retarget audiences when stats surface an audience-fit problem.
---

# ClickFunnels Stats Skill

Read aggregated analytics for funnels and individual pages. Two resources:

- **`Funnels::Stats`** - one funnel, one summary, optional per-step breakdown.
- **`Pages::Stats`** - one page, single-step analytics, with funnel context when the page is reached via a funnel step.

Both endpoints return the same metric vocabulary (`views_all`, `views_unique`, `optins`, `optin_rate`, `sales_count`, `sales_rate`, `sales_value`, `earnings_per_view`, etc.) so an agent can write one analyzer that consumes either payload.

## When to use this skill

- Daily / weekly automated audits that flag underperforming funnels or pages.
- Generating a report for a workspace owner ("which funnel made the most this week").
- Detecting regressions: a step's `optin_rate` dropping week-over-week.
- Picking which page to A/B test next (lowest `optin_rate` in a multi-step funnel).
- Picking which step to retarget with a conditional split (lowest `sales_rate` with high `views_all`).

This skill is **read-only**. If the analysis suggests a fix, hand off to the [Funnels Skill](/.well-known/funnels/skill.md) (split tests / conditional splits) or the [Pages Skill](/.well-known/pages/skill.md) (page rewrite via PML).

## Authentication

OAuth2 password grant. See the [Refine Filters Skill](/.well-known/refine-filters/skill.md#authentication) for the full flow - the same access token works here. No additional feature gates required for read access.

## Endpoint reference

| Action            | Method | Path                                |
|-------------------|--------|-------------------------------------|
| Funnel stats      | GET    | `/api/v2/funnels/:funnel_id/stats`  |
| Page stats        | GET    | `/api/v2/pages/:page_id/stats`      |

## Timerange parameters (both endpoints)

| Param              | Type    | Default                              | Notes                                                              |
|--------------------|---------|--------------------------------------|--------------------------------------------------------------------|
| `timerange_start`  | ISO8601 | now - 30 days                        | Clamped: if start > end, window resets to 30 days ending at `end`. |
| `timerange_end`    | ISO8601 | now                                  | Clamped: future end dates are clamped to now.                      |

The window is also clamped to a **90-day maximum**. Requests for longer windows are silently truncated to 90 days. For multi-quarter analysis, page through 90-day windows and aggregate client-side.

## Funnel stats

`GET /api/v2/funnels/:funnel_id/stats`

### Query parameters

| Param         | Type   | Notes                                                                          |
|---------------|--------|--------------------------------------------------------------------------------|
| `expand[]`    | string | Pass `steps` to receive a `steps` array with per-step metrics instead of the default `page_public_ids` array. |

### Response (default - without `expand[]=steps`)

```json
{
  "funnel": { "id": 1, "public_id": "gESyMv", "name": "My Sales Funnel" },
  "currency": "USD",
  "timerange": { "from": "2026-03-03T00:00:00Z", "to": "2026-04-02T23:59:59Z" },
  "summary": {
    "earnings_per_click": "1.24",
    "upfront_sales": "4250.00",
    "upfront_sales_count": 17,
    "recurring_sales": "850.00",
    "average_cart_value": "250.00",
    "pageviews": 3422
  },
  "page_public_ids": ["abc123", "def456", "ghi789"]
}
```

### Response (with `expand[]=steps`)

`page_public_ids` is replaced by `steps`. Each step has the same shape as the `Pages::Stats` `step` block:

```json
{
  "funnel": { ... },
  "currency": "USD",
  "timerange": { ... },
  "summary": { ... },
  "steps": [
    {
      "page_id": 100,
      "name": "Opt-in",
      "current_path": "/optin",
      "views_all": 1500,
      "views_unique": 1200,
      "optins": 360,
      "optin_rate": 0.3,
      "sales_count": 0,
      "sales_rate": 0,
      "sales_value": "0.00",
      "recurring_sales_count": 0,
      "recurring_sales_value": "0.00",
      "earnings_per_view": "0.00",
      "earnings_per_unique_view": "0.00"
    }
  ]
}
```

## Page stats

`GET /api/v2/pages/:page_id/stats`

Stats are computed from the page's associated funnel step. If the page is not part of a funnel (e.g. a standalone site page), `funnel` and `step` are both `null` - there is no other analytics surface for these pages yet.

### Response

```json
{
  "currency": "USD",
  "timerange": { "from": "2026-03-03T00:00:00Z", "to": "2026-04-02T23:59:59Z" },
  "page": {
    "id": 100,
    "public_id": "abc123",
    "name": "Order Page",
    "current_path": "/order-canonical",
    "type": "funnel_page"
  },
  "funnel": { "id": 3, "public_id": "fnlXyz", "name": "Q2 Promo" },
  "step": {
    "page_id": 100,
    "name": "Order",
    "current_path": "/order-canonical",
    "views_all": 800,
    "views_unique": 720,
    "optins": 0,
    "optin_rate": 0,
    "sales_count": 24,
    "sales_rate": 0.033,
    "sales_value": "5400.00",
    "recurring_sales_count": 4,
    "recurring_sales_value": "120.00",
    "earnings_per_view": "6.75",
    "earnings_per_unique_view": "7.50"
  }
}
```

## Metric vocabulary

| Field                          | Meaning                                                              |
|--------------------------------|----------------------------------------------------------------------|
| `views_all`                    | Total pageviews in the window (includes repeats).                    |
| `views_unique`                 | Distinct visitor sessions in the window.                             |
| `optins`                       | Lead form submissions on this step.                                  |
| `optin_rate`                   | `optins / views_unique` as a 0..1 float.                             |
| `sales_count`                  | Distinct purchases attributed to this step.                          |
| `sales_rate`                   | `sales_count / views_unique` as a 0..1 float.                        |
| `sales_value`                  | Upfront sales total (`Money.to_s`, e.g. `"5400.00"`).                |
| `recurring_sales_count`        | Subscriptions / installments started.                                |
| `recurring_sales_value`        | Recurring revenue captured in the window.                            |
| `earnings_per_view`            | `sales_value / views_all`.                                           |
| `earnings_per_unique_view`     | `sales_value / views_unique`.                                        |
| `summary.earnings_per_click`   | Funnel-level EPC across the whole workflow.                          |
| `summary.upfront_sales`        | Funnel-level one-time revenue.                                       |
| `summary.recurring_sales`      | Funnel-level recurring revenue.                                      |
| `summary.average_cart_value`   | `summary.upfront_sales / summary.upfront_sales_count`.               |

Money fields are strings (`"5400.00"`) to preserve precision. Rates are floats in `[0, 1]`.

## Example use case - recurring "funnel auditor" agent

A nightly agent job that pulls every funnel's stats for the trailing 7 days, scores them, and writes a digest to Slack with concrete next-step suggestions.

### Job shape

```
schedule:    cron, daily 06:00 UTC
inputs:      workspace_id, oauth access_token, list of funnel_ids (or "all")
outputs:     ranked findings + suggested actions, posted to Slack
```

### Pseudocode

```python
WINDOW_DAYS = 7
now = datetime.utcnow()
window = {
    "timerange_start": (now - timedelta(days=WINDOW_DAYS)).isoformat() + "Z",
    "timerange_end":   now.isoformat() + "Z",
}

findings = []
for funnel_id in funnel_ids:
    # 1. Pull funnel-level summary + per-step breakdown.
    funnel = GET(f"/api/v2/funnels/{funnel_id}/stats",
                 params={**window, "expand[]": "steps"})

    summary = funnel["summary"]
    steps   = funnel["steps"]

    # 2. Heuristic: low EPC overall.
    if float(summary["earnings_per_click"]) < 0.10 and summary["pageviews"] > 500:
        findings.append({
            "funnel_id":  funnel["funnel"]["public_id"],
            "severity":   "high",
            "kind":       "low_epc",
            "message":    f"EPC ${summary['earnings_per_click']} on "
                          f"{summary['pageviews']} views - funnel is monetising poorly.",
            "suggestion": "Add a conditional split on a high-intent filter "
                          "(see /.well-known/refine-filters/skill.md) or "
                          "A/B test the order page (see /.well-known/funnels/skill.md "
                          "#split-test-steps).",
        })

    # 3. Heuristic: optin step with low rate but plenty of traffic.
    for step in steps:
        if step["views_unique"] > 200 and 0 < step["optin_rate"] < 0.15:
            findings.append({
                "funnel_id":  funnel["funnel"]["public_id"],
                "step":       step["current_path"],
                "severity":   "medium",
                "kind":       "weak_optin",
                "message":    f"Step `{step['current_path']}`: optin rate "
                              f"{round(step['optin_rate']*100,1)}% on "
                              f"{step['views_unique']} unique views.",
                "suggestion": "Build a NEW page with a stronger hero + CTA in PML "
                              "(see /.well-known/page-markup), then wrap the "
                              "original page in a split test against it at weight "
                              "50/50. Do not overwrite the live page: that needs "
                              "explicit approval first, see the approval guardrail "
                              "in the Page Markup skill.",
            })

        # 4. Heuristic: order step with traffic but no sales.
        if step["views_unique"] > 200 and step["sales_count"] == 0 \
           and step["name"].lower().startswith(("order", "checkout")):
            findings.append({
                "funnel_id":  funnel["funnel"]["public_id"],
                "step":       step["current_path"],
                "severity":   "high",
                "kind":       "zero_sales",
                "message":    f"Order step `{step['current_path']}` had "
                              f"{step['views_unique']} unique views and zero sales.",
                "suggestion": "Confirm pricing, payment processor connection, and "
                              "fulfillment readiness before iterating on copy.",
            })

# 5. Rank and post.
findings.sort(key=lambda f: ({"high":0,"medium":1,"low":2}[f["severity"]],
                              -float(f.get("views", 0))))
slack_post(format_digest(findings))
```

### Notes for the agent author

- **Cache the auth token.** It is long-lived; refresh once per run, not per call.
- **One funnel <-> one HTTP call.** Always pass `expand[]=steps` so you can score steps without a second roundtrip.
- **Money is a string.** Cast with `float()` before comparing thresholds.
- **`optin_rate` and `sales_rate` are 0..1 floats**, not percentages. Multiply by 100 only when rendering.
- **Skip empty windows.** If `summary.pageviews` < 50, the noise floor is too high - emit "insufficient traffic" instead of a finding.
- **Don't repeat suggestions.** Persist the previous run's findings keyed by `(funnel_id, kind, step)` and skip findings that are still open from the prior run unless severity escalates.
- **Page stats are useful for landing-page audits.** When the agent is given a `page_id` instead of a funnel (e.g. for a standalone landing page hooked into ads), fetch `/api/v2/pages/:page_id/stats` directly - the response carries the same per-step block under `step`.
- **Suggest, do not overwrite.** An auditor proposes changes; it does not rewrite live pages on its own. Writing `markup` to an existing page replaces the whole page with a PML subset of what the editor can build, so it requires explicit approval from the person first: see [the approval guardrail](/.well-known/page-markup/skill.md#stop-get-approval-before-overwriting-an-existing-page). Prefer the additive route the heuristics above suggest: build a new page and split-test it against the original.

### Failure modes to handle

- **404** - the funnel/page id is unknown to this workspace, or the page is not a `user_page`.
- **`funnel: null` / `step: null` on a page** - the page isn't wired into a funnel; emit a "no analytics available" finding rather than a numeric one.
- **Clamped window** - if you ask for >90 days, the response silently uses 90. Re-read `timerange.from` / `timerange.to` from the response, not from your request, before quoting the window in the digest.

## Related skills

- [Funnels Skill](/.well-known/funnels/skill.md) - the structure these stats describe; primary destination for "fix it" actions.
- [Pages Skill](/.well-known/pages/skill.md) - build a replacement for a low-performing page in PML (overwriting an existing page needs approval first).
- [Refine Filters Skill](/.well-known/refine-filters/skill.md) - author a stored filter for retargeting via a conditional split.
