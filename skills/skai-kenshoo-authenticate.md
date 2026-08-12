---
name: skai-authenticate
description: >-
  Obtain and manage a Skai (Kenshoo) API access token. Use before any other Skai skill —
  every Skai REST operation requires a bearer token, and the token is minted by exchanging a
  permanent refresh token rather than through OAuth.
api: Skai (Kenshoo) API
base_url: https://services.kenshoo.com
operations:
  - getAccessToken
generated: '2026-08-12'
method: generated
source: >-
  openapi/skai-kenshoo-api-openapi.yml and the Authentication section of
  https://developers.skai.io/. Every operationId was verified present in the saved spec.
---

# Authenticate against the Skai API

Skai does **not** use OAuth 2.0 for its REST API. It uses a bespoke two-step exchange: a
permanent refresh token, minted once by a human, is traded for a short-lived JWT access token.

## One-time human setup

A person must log in at `https://login.kenshoo.com/api/dev/refresh-token` and record the
**refresh token** and the **client ID**. This cannot be automated.

Two rules from Skai's own docs:

- The user who logs in determines the token's permissions. The account needs the **Standard**
  role or higher.
- Create a **dedicated service user** for API access. Rate limits are per-user, so sharing a
  human's user with an integration makes both compete for the same 60/minute budget.

The refresh token **does not expire** and **cannot be recovered** — store it in a secret
manager. If it is lost, generate a new one.

## Mint an access token — `getAccessToken`

`POST /api/v1/token` on `https://services.kenshoo.com`.

The body is **form-encoded**, not JSON. Skai explicitly rejects a refresh token sent as a URL
query parameter, because it is confidential.

```
curl -X POST \
  -d "refresh_token=<YOUR_REFRESH_TOKEN>&client_id=<YOUR_CLIENT_ID>" \
  https://services.kenshoo.com/api/v1/token
```

If the API user belongs to **multiple agencies**, you must pin the context or the request is
ambiguous — add `agency_id`:

```
curl -X POST \
  -d "refresh_token=<YOUR_REFRESH_TOKEN>&client_id=<YOUR_CLIENT_ID>&agency_id=<AGENCY_ID>" \
  https://services.kenshoo.com/api/v1/token
```

The response carries `access_token`, `expires_in` (seconds — typically `21600`, i.e. 6 hours)
and `email`.

## Use the token

Every subsequent call:

```
curl -H "Authorization: Bearer <ACCESS_TOKEN>" \
  https://services.kenshoo.com/api/v1/campaigns?ks=ks1234
```

## Cache the token — this is a rate-limit rule, not an optimization

**Reuse the access token for its full `expires_in`.** Skai warns that minting new tokens too
often trips the rate limit, and the limit is shared with your actual work: 60 requests/minute
and 2,000 requests/hour per user. An agent that re-authenticates per call burns its own budget.

Two ways to detect expiry:

1. Track `expires_in` and refresh proactively (refresh at ~90% of the lifetime).
2. Catch **HTTP 401** and re-mint once, then retry the original request.

Do not retry a 401 more than once. A second 401 after a fresh token means a permissions or
agency-context problem, not an expiry problem.

## The `ks` parameter is separate from the token

The token identifies **who** you are. The `ks` query parameter identifies **which Skai
account** you are operating on, and 51 operations require it. Find it in the Skai platform
under *Administration -> About Skai -> Server ID*. For Social, the literal value is
`ks=social`.

## Contract gotcha

The published OpenAPI declares `security: [{BearerAuth: []}]` at the root but **never defines
`BearerAuth` in `components.securitySchemes`**. Generated clients will not know the scheme is
HTTP bearer with a JWT and may emit no auth handling at all. Apply
`overlays/skai-kenshoo-api-overlay.yaml`, which supplies the missing definition, before
generating a client.

## MCP is different

If you are connecting an AI assistant rather than writing REST code, the hosted MCP servers at
`https://mcp.kenshoo.com` use real OAuth 2.0 (authorization_code + PKCE S256 against
`https://login.kenshoo.com`) or a separate **90-day Personal Access Token** sent as
`Authorization: Bearer <PAT>` with a `ks-name` header. The REST refresh token and the MCP PAT
are **not** interchangeable.
