---
name: skai-bulk-update-entities
description: >-
  Change campaigns, ad groups, keywords, bids, budgets, targeting and other attributes at scale
  in Skai (Kenshoo) using the Bulk Update one-file endpoint, then track the resulting job to
  completion. Use this for anything beyond a handful of entities, or for any attribute the
  typed CRUD endpoints do not cover.
api: Skai (Kenshoo) API
base_url: https://services.kenshoo.com
operations:
  - bulkUpdate
  - getJobStatus
  - getJobResults
  - getCampaigns
  - updateCampaigns
  - getAdGroups
  - updateAdGroups
generated: '2026-08-12'
method: generated
source: >-
  openapi/skai-kenshoo-api-openapi.yml (tags Bulk Update, Jobs, Campaigns, Ad Groups) and the
  "Choosing the Right API" table at https://developers.skai.io/. Every operationId was verified
  present in the saved spec.
---

# Change entities at scale in Skai

Authenticate first — see `skai-kenshoo-authenticate.md`.

## Pick the right write path

Skai publishes an explicit decision table, and getting this wrong is the most common failure.

| Situation | Use | Why |
|---|---|---|
| Millions of rows, hundreds of attributes, all publishers **except Meta** | `bulkUpdate` — `POST /api/v1/bulk_update` | Full attribute coverage |
| Thousands of entities, **common attributes only** | `createCampaigns` / `updateCampaigns` / `createAdGroups` / `updateAdGroups` / `createAds` | Simpler, typed |
| Meta (Facebook/Instagram) entities | `/api/v2/campaigns`, `/api/v2/ad_groups`, `/api/v2/ads` and the tag operations | Separate schema branch |

The critical point: **the typed CRUD endpoints only cover common attributes.** Skai states
that campaign endpoints handle name, budget, bid, status, dates and campaign type, and that
"for the full range of campaign settings (targeting, extensions, dimension labels, and
publisher-specific fields) use Bulk Update." If the attribute you need is not in `CampaignDTO`
or `AdGroupDTO`, it is not missing from the product — it lives in Bulk Update.

Bulk Update does **not** cover Meta. Meta entities have their own `/api/v2` surface and their
own schemas that share nothing with `CampaignDTO`.

## Read before you write

Fetch current state so your change is a delta, not a guess:

- `getCampaigns` — `GET /api/v1/campaigns` — filter with `ids` or `channel_campaign_ids`
  (both capped at **500 items**), `statuses`, `channel_types`.
- `getAdGroups` — `GET /api/v1/ad_groups` — same filter family.

Every entity carries **two identities**: Skai's own integer `id` and the publisher's
`channel_campaign_id` / `channel_ad_group_id` / `channel_account_id`. Know which one your
source system holds before you build the file, and stay consistent — mixing them is a silent
source of unmatched rows.

## Submit the bulk update

`POST /api/v1/bulk_update` (operationId `bulkUpdate`), described by Skai as "Bulk Update (One
File)". It returns a **job**, not a result.

## Track the job

Skai defines any operation that can take more than a few seconds as a job:

1. `getJobStatus` — `GET /api/v1/jobs/{job_id}/status`
2. `getJobResults` — `GET /api/v1/jobs/{job_id}/results/file`

There is no webhook. Poll `getJobStatus` with backoff until it completes, then fetch the
results file. Remember the budget: 60 requests/minute and 2,000/hour per user, shared with
everything else that user is doing.

Job outcomes include a partial state — the envelope `status` field is one of `SUCCESS`,
`FAILED` or **`PARTIAL_SUCCESS`**. Always inspect the results file rather than trusting the
HTTP status alone.

## STOP — retries are not safe here

**The Skai API publishes no idempotency mechanism.** There is no idempotency key, no request
deduplication, no `ETag`/`If-Match` support. This matters more on this endpoint than anywhere
else in the API, because `bulkUpdate` accepts millions of rows of bid, budget and status
changes.

Consequences you must design around:

- If a submission times out at the network layer, **you do not know whether it was applied.**
  Do not blind-retry. Poll for a job that may already exist, or reconcile by reading state
  back with `getCampaigns` / `getAdGroups` before resubmitting.
- Nine Amazon DSP endpoints return **HTTP 207 Multi-Status** for partial success. A whole-
  request retry after a 207 **re-applies the rows that already succeeded**. Extract the failed
  entities from `entities[]` where `success` is `false` and resubmit only those.
- Build a client-side change ledger — record what you sent, when, and what came back — because
  the API gives you no request ID or correlation header to reconcile against later.

## Read the per-entity result, not just the HTTP code

The Skai envelope reports success **per entity**:

```json
{
  "status": "PARTIAL_SUCCESS",
  "entities": [
    {"id": 25000, "success": true,  "errors": []},
    {"id": null,  "success": false, "errors": [{"field_name":"name","error":"ILLEGAL_NAME"}]}
  ]
}
```

A `200` with `status: PARTIAL_SUCCESS` is a normal outcome, not an exception. Treat
`entities[].success` as the authoritative result for every row.

## Errors

- **400** — validation failure. `entities[].errors[]` names the field and an error token.
- **401** — expired access token; re-mint once and retry (safe, the call never reached the
  write path).
- **429** — rate limit. Back off; no `Retry-After` is published.
- **500** — server error. **Not automatically retry-safe** for a write, given no idempotency.
  Reconcile state first.

## Agent alternative

For conversational bid/budget/status changes, Skai runs an **Operations MCP** server at
`https://mcp.kenshoo.com/operations-mcp` that applies changes across every publisher and
account with a preview-and-approve step before anything lands. Skai publishes no tool list for
it, so it cannot be bound to specific operations here — but for an agent making a small number
of supervised changes, it is the safer surface than a raw `bulkUpdate` with no idempotency.
