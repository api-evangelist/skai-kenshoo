---
name: skai-pull-performance-report
description: >-
  Pull advertising performance data out of Skai (Kenshoo) across search, social and retail
  media publishers — discover which columns exist, then run either a synchronous report or the
  asynchronous analysis flow and download the result. This is the primary data-access path and
  the main reason to call the Skai API.
api: Skai (Kenshoo) API
base_url: https://services.kenshoo.com
operations:
  - getAvailableColumns
  - getRelevantColumns
  - fetchReport
  - asyncAnalysisReport
  - getAsyncReportStatus
  - downloadAsyncReport
generated: '2026-08-12'
method: generated
source: >-
  openapi/skai-kenshoo-api-openapi.yml and the Overview / Reporting Best Practices / Group by
  and Segmentation sections of https://developers.skai.io/. Every operationId was verified
  present in the saved spec.
---

# Pull a performance report from Skai

Authenticate first — see `skai-kenshoo-authenticate.md`. Every call below needs
`Authorization: Bearer <access_token>` and, on most operations, the `ks` account parameter.

## Step 1 — decide sync or async before you build the request

This choice is not cosmetic; the two paths have different shapes.

| Expected rows | Use | Operation |
|---|---|---|
| Up to a few thousand | Synchronous | `fetchReport` — `POST /api/v1/reports` |
| More than a few thousand | Asynchronous | `asyncAnalysisReport` — `POST /api/v1/reports/async/analysis` |

Skai's own guidance: *"If your report may return more than a few thousand rows, use Async
Analysis Reports and poll for results rather than the synchronous endpoint."*

Do **not** use `runAsyncReport` (`POST /api/v1/reports/async`) for new work — Skai's own
summary marks it *(deprecated)* and marks `asyncAnalysisReport` *(recommended)*.

## Step 2 — discover the columns, do not guess them

Column names are account-specific: alongside standard performance metrics, each account has
its own dimensions (custom tagging labels), conversion events (publisher, pixel and
third-party) and custom metrics (formulas the customer's team defined). **Never hardcode a
column name.**

- `getAvailableColumns` — `GET /api/v1/reports/{entity}/available_columns` — the complete
  column list for an entity type, with descriptions and types.
- `getRelevantColumns` — `POST /api/v1/relevant-columns` — narrows to the columns applicable
  to a specific query.

Valid `{entity}` values: `CAMPAIGN`, `ADGROUP`, `KEYWORD`, `AD`, `PRODUCT_ASSET`,
`PRODUCT_TARGETING`, `PORTFOLIO`.

Note that `KEYWORD`, `PRODUCT_ASSET` and `PRODUCT_TARGETING` are **reportable but not CRUD
resources** — you can report on them and bulk-update them, but there are no create/read/update
operations for them.

## Step 3 — choose the breakdown

`breakdown_type` controls the shape of the result:

- **`FLAT`** — no grouping, unsegmented rows.
- **`GROUP`** — segment by the columns in `group_bys`. Entity detail is **dropped**. Grouping
  only by `{ "name": "Day", "group": "TimeSegment" }` gives you one row per date with no
  campaign identity, mirroring a grid export.
- **`SEGMENT`** — segment by date *and* another column. Put the date column in `group_bys` and
  the extra column in `fields`:

```json
{
  "breakdown_type": "SEGMENT",
  "group_bys": [ { "name": "Day", "group": "TimeSegment" } ],
  "fields":    [ { "name": "CampaignId", "group": "ATTRIBUTES" } ]
}
```

The most common mistake is using `GROUP` and then wondering where the campaign IDs went. If
you need both a time series and entity identity, you want `SEGMENT`.

## Step 4 — narrow the request before you run it

Skai's published best practices, in order of impact:

1. **Filter for non-zero data.** For performance reports, filter to rows where a key metric is
   greater than zero (e.g. impressions > 0). Advertising data is overwhelmingly sparse; this
   is usually the single biggest reduction available.
2. **Scope structure reports by recency.** Apply a "last updated > X days ago" filter to
   retrieve only entities that actually changed.
3. **Ask for the columns you need**, not every column `getAvailableColumns` returned.

## Step 5a — synchronous: `fetchReport`

`POST /api/v1/reports` returns the report body directly. Use it for small, interactive pulls.

## Step 5b — asynchronous: submit, poll, download

1. `asyncAnalysisReport` — `POST /api/v1/reports/async/analysis`. Returns an `execution_id`.
2. `getAsyncReportStatus` — `GET /api/v1/reports/async/{execution_id}/status`. Poll this.
3. `downloadAsyncReport` — `GET /api/v1/reports/async/{execution_id}`. Returns
   `application/x-zip-compressed` — the payload is a **zip**, not JSON. Unzip before parsing.

There is **no webhook and no callback**. Polling is the only completion signal Skai offers.

Poll with backoff, and budget your polls: at 60 requests/minute per user, a tight poll loop is
self-defeating. Poll every few seconds at most, and remember the 2,000/hour ceiling.

To re-run a report already configured in the Skai UI, use `runAsyncReportById` —
`POST /api/v1/reports/async/existing_report/{report_id}`. Setting
`useOriginalDeliveryMethod=true` delivers it to the destination configured in Skai (an email
address or an FTP location) instead of returning it to you.

## Error handling

- **401** — the access token expired. Re-mint once and retry.
- **429** — `API rate limit exceeded`. Back off to the next window. There is **no
  `Retry-After` header**, and 429 is not declared on any operation in the spec even though the
  limit is documented, so handle it defensively.
- **400 / 500** — read `entities[].errors[]` in the Skai envelope:

```json
{"status":"FAILED","entities":[{"id":null,"success":false,
 "errors":[{"field_name":"name","error":"ILLEGAL_NAME"}]}]}
```

Skai publishes no error-code registry, so treat unknown `error` tokens as opaque and surface
them verbatim rather than trying to map them.

## Cheaper alternative for conversational use

If the caller is an AI assistant rather than a program, the Reporting MCP server at
`https://mcp.kenshoo.com/reports-mcp` exposes `fetch_report`, `relevant_columns`, `get_today`,
`get_change_log` and `get_competitive_context` and hides the async polling loop. Skai claims it
takes roughly one-tenth the calls of a publisher-native interface. Note that `get_change_log`
and `get_competitive_context` have **no REST equivalent** — they exist only on MCP.
