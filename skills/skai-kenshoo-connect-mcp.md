---
name: skai-connect-mcp
description: >-
  Connect an AI assistant (Claude, ChatGPT, Cursor, VS Code, Windsurf, Gemini) to Skai's
  first-party hosted MCP servers — the read-only Reporting MCP and the write-access Operations
  MCP — and understand what each can and cannot do.
api: Skai (Kenshoo) MCP Servers
base_url: https://mcp.kenshoo.com
operations: []
generated: '2026-08-12'
method: generated
source: >-
  The MCP tag of Skai's own OpenAPI at https://developers.skai.io/, Skai's setup guide at
  https://skai-mcp-guide.vercel.app/, live probes of https://mcp.kenshoo.com, and
  https://mcp.kenshoo.com/.well-known/oauth-protected-resource
---

# Connect an agent to Skai over MCP

Skai runs first-party hosted, remote MCP servers. There is no local install and no npx package
— these are HTTP endpoints you point a client at.

## Two servers, split by risk

Skai deliberately split read from write rather than shipping one server.

| Server | URL | Access | What it does |
|---|---|---|---|
| Reporting MCP | `https://mcp.kenshoo.com/reports-mcp` | read | Campaigns, ad groups, keywords, ads, products, performance, budgets, change log — unified across 100+ publishers |
| Operations MCP | `https://mcp.kenshoo.com/operations-mcp` | **write** | Bid, budget and status changes at scale, with preview and approval before anything is applied |

Skai says it built four MCP servers and that these two are live today; the other two are not
publicly named.

## You need your KS ID first

Every Skai session is scoped to an account identifier — the **KS ID**, formatted like
`ks1234`. Find it in the Skai platform under *Administration -> About Skai -> Server ID*.

It appears in two places depending on the auth mode:

- **OAuth clients** use a tenant URL: `https://mcp.kenshoo.com/reports-mcp/ks1234`
- **PAT clients** use the base URL plus a `ks-name` header.

## Authentication: two paths

### OAuth 2.0 (ChatGPT and Claude)

Standards-compliant. The authorization server is `https://login.kenshoo.com`, discoverable at
`https://login.kenshoo.com/.well-known/oauth-authorization-server`:

- `authorization_endpoint`: `https://auth0.kenshoo.com/authorize`
- `token_endpoint`: `https://login.kenshoo.com/api/oauth/token`
- grants: `authorization_code`, `refresh_token`; PKCE `S256`; response type `code`
- scopes: `openid`, `email`, `profile`, `offline_access`

The protected-resource metadata at
`https://mcp.kenshoo.com/.well-known/oauth-protected-resource` points back at that server, so
a client that implements RFC 9728 discovery can wire itself up from the 401 challenge alone.

Skai publishes a public OAuth **client ID** in its setup guide for connector configuration and
tells you to leave the secret blank.

### Personal Access Token (every other client)

Generate a PAT at `https://login.kenshoo.com/api/dev/refresh-token`. It is **valid 90 days**
and must be rotated. Send:

```
Authorization: Bearer <YOUR_PAT>
ks-name: <YOUR_KS_NAME>
```

ChatGPT does **not** support header-based tokens, so there is no PAT fallback there — OAuth is
the only path.

The MCP PAT and the REST API refresh token are **different credentials**. Do not substitute
one for the other.

## Claude Code setup

Skai publishes this configuration:

```json
{
  "mcpServers": {
    "skai": {
      "type": "http",
      "url": "https://mcp.kenshoo.com/reports-mcp",
      "headers": {
        "Authorization": "Bearer <YOUR_PAT_TOKEN>",
        "ks-name": "<YOUR_KS_NAME>"
      }
    }
  }
}
```

Swap the URL for `https://mcp.kenshoo.com/operations-mcp` to add write access. Add it as a
**second, separately named server** rather than replacing the first — the read/write split is
the point, and collapsing both into one entry throws away the safety boundary Skai built.

## Reporting MCP tools

Skai publishes five:

| Tool | What it does |
|---|---|
| `fetch_report` | Retrieve reporting data for campaigns, ad groups, keywords, ads or products |
| `relevant_columns` | Identify which columns apply to a given query |
| `get_today` | Current-day reporting metrics |
| `get_change_log` | Change log for campaigns and other entities |
| `get_competitive_context` | Competitive brand data and context |

Verify the connection by asking the assistant *"What Skai tools do you have access to?"* — you
should see those five names.

Two of them, **`get_change_log`** and **`get_competitive_context`**, have **no REST
equivalent**. They are MCP-only capabilities; you cannot fall back to the API for them.

## Operations MCP tools

Skai names the capability areas — bid, budget and status changes — but **publishes no tool
list**, and live `tools/list` requires authentication. Discover the actual tools by connecting
and asking the assistant to enumerate them. Do not assume names.

## Supported clients

Claude Code, Claude Desktop, Claude Web, Cursor, VS Code, Windsurf, ChatGPT, Google Gemini.

For ChatGPT specifically: you need a Plus, Pro, Business, Enterprise or Edu plan; on
Business/Enterprise/Edu a workspace admin must enable **Developer mode** first; and custom
apps can only be created in ChatGPT on the web, after which they sync to desktop and mobile.

## Operating notes for an agent with write access

- **Scopes do not separate read from write.** Both servers advertise the same four scopes
  (`openid`, `profile`, `email`, `offline_access`). The only boundary is which endpoint you
  connected. If you want a read-only agent, connect it to `reports-mcp` and *do not* give it
  `operations-mcp`.
- Permission actually comes from the Skai platform role of the authenticating user (Standard
  or higher) and the KS/agency context — not from OAuth scope. A PAT minted by an admin gives
  the agent an admin's reach.
- Operations MCP applies a preview-and-approval step before changes land. Keep it. The
  underlying REST API has **no idempotency mechanism**, so an unattended agent retrying a
  failed write has no safe replay semantics.
- Skai's blog states broader creation support and deeper account-level actions are still
  rolling out, "with new coverage shipping most weeks" — re-enumerate `tools/list` rather than
  caching a tool set indefinitely.

## Verify the endpoint is live

An unauthenticated `tools/list` returns `401` with a `WWW-Authenticate: Bearer` challenge
naming the realm and a `resource_metadata` URL. That 401 is the healthy response — it confirms
the server is up and tells a compliant client exactly where to authenticate.

Do **not** use `https://mcp.kenshoo.com/.well-known/oauth-protected-resource/<name>` to test
whether a given server exists: that path echoes any name you supply and returns 200 for
paths that do not exist.
