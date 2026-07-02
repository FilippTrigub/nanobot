# Nanobot — Campaign Admin Bot State

## What this is

A nanobot instance configured as the admin Telegram bot for the Quincy campaign management platform (frank-ingest). It connects to frank-ingest's MCP server over HTTP and exposes campaign data-management capabilities to admins via Telegram.

Nanobot is a Python AI agent framework. This repo is a fork/deployment config layered on top of the upstream nanobot package — deployment-specific config lives in `config.deploy.json` (untracked, gitignored, contains API keys).

---

## Deployment Config (`config.deploy.json`)

```
providers.azure_openai.apiBase  = https://filip-mfmovghf-swedencentral.cognitiveservices.azure.com
providers.azure_openai.model    = gpt-5.4

channels.telegram.token         = ${TELEGRAM_TOKEN}
channels.telegram.allowFrom     = [388259993, 8773671352]   # admin Telegram user IDs

tools.mcpServers.frank-ingest.url     = https://quincy.bravesea-b11173b3.swedencentral.azurecontainerapps.io/api/mcp
tools.mcpServers.frank-ingest.headers = { x-api-key: ${INGEST_API_KEY} }
tools.mcpServers.frank-ingest.toolTimeout = 180   # precinct analytics can take 5-90s
```

Config file is intentionally untracked (`config.deploy.json` in `.gitignore`). Environment variables required at runtime:
- `AZURE_OPENAI_API_KEY`
- `TELEGRAM_TOKEN`
- `INGEST_API_KEY` — same key as frank-ingest's `INGEST_API_KEY` ACA secret

---

## MCP Tool Inventory

All tools are exposed by the frank-ingest MCP server at `/api/mcp`. The bot calls them automatically based on admin messages.

### Data pipeline tools (original)

| Tool | Description |
|------|-------------|
| `create_upload` | Initiate a voter file upload, returns SAS URL |
| `get_voters_schema` | Return the voters table column list |
| `get_source_status` | Return current voter source status |
| `query_voters` | Find campaign-scoped voters from structured name or address details; returns a full profile for one unique match or capped candidates for ambiguous matches; does not run SQL, counts, aggregates, trends, or broad list queries |
| `run_pipeline` | Trigger parse → validate → ingest pipeline |

### Admin operation tools (added 2026-06-26)

| Tool | Description |
|------|-------------|
| `add_diary_entry` | Upsert the campaign diary for a given day (defaults to today). Body appears in the Daily Command panel. |
| `add_contact` | Create a contact in the Rolodex/Supporters contact book. `contact_category` controls which view it appears in. |
| `get_contacts` | Search contacts by name/email/category/state. Supports `view=rolodex` or `view=supporters`. |
| `add_finance_record` | Add an income/expense/expected finance record to the campaign ledger. |
| `get_field_status` | Universe totals, walk/call/mail contact rates, score freshness, top 20 precincts by voter count. May take up to 90s. |
| `get_precinct` | Full analytics detail for precincts matching a code or name substring. May take up to 90s. |

All admin tools accept an optional `campaign_id` argument. When omitted they fall back to `DEFAULT_CAMPAIGN_ID` in the frank-ingest ACA environment — set this env var to avoid specifying a UUID on every message.

### Telegram volunteer tools (shared action wrappers, 2026-06-28)

The Telegram volunteer MCP tools still exist in frank-ingest and keep their Telegram-facing schemas:

| Tool | Description |
|------|-------------|
| `telegram_get_my_lists` | Return walk/call lists assigned to the resolved volunteer. |
| `telegram_get_list_voters` | Return paginated voters for an assigned walk/call list, including prior result data. |
| `telegram_submit_result` | Submit or update a walk/call contact result for a voter on an assigned list. |

Nanobot supplies the sender's numeric Telegram ID to these tools. Frank-ingest resolves that `telegram_id` to the app `users.id`, then calls shared trusted-user volunteer bot actions in `lib/volunteer-bot-actions.ts`. The shared actions own list lookup, voter paging access checks, and result submission; Telegram wrappers own Telegram identity resolution and MCP response wrapping. The web volunteer assistant is separate: it uses assistant-ui on `/volunteer`, Auth.js `session.user.id` for identity, and the same shared actions through `/api/volunteer/chat`. V1 supports volunteer list lookup, voter paging, and result submission only; no admin chat exists.

---

## Auth Flow

The Telegram bot → nanobot → MCP HTTP bridge at `/api/mcp` → frank-ingest route handlers.

- **Telegram auth**: allowFrom list in config gates which Telegram user IDs can send messages.
- **MCP auth**: every request carries `x-api-key: ${INGEST_API_KEY}` header. Frank-ingest's `proxy.ts` validates this key before forwarding to the MCP server. The MCP server also validates it independently.
- **Route auth**: admin operation routes use `requireCampaignAdminOrMachine` — sessionless requests (no Auth.js session cookie) that have passed the API key gate are trusted as machine clients. `*_by_user_id` DB columns are stored as `null` for machine-originated writes.
- **Volunteer Telegram tools**: frank-ingest resolves the supplied Telegram ID to an app user before invoking shared trusted-user volunteer bot actions. Web volunteer chat does not use Telegram IDs; it uses Auth.js `session.user.id`.

---

## Frank-ingest MCP Server

The MCP server runs as a separate process inside the same frank-ingest container (`scripts/start.sh` launches both `node server.js` and `mcp/server.js`). It listens on port 3001 internally but is not exposed publicly; the Next.js app proxies MCP requests through `/api/mcp`.

MCP server source: `frank-ingest/mcp/server.ts` and `frank-ingest/mcp/tools/`.

---

## Changelog

### 2026-06-26 — Admin bot tool extension
- Added 6 new MCP tools to frank-ingest MCP server (see Tool Inventory above)
- Added `requireCampaignAdminOrMachine` auth pattern to frank-ingest (`lib/campaign-access.ts`)
- Updated 4 frank-ingest route handlers to accept machine clients: `diary`, `contacts`, `finance`, `analytics/precincts`
- Created `mcp/tools/api-client.ts` in frank-ingest (shared fetch helpers)
- `DEFAULT_CAMPAIGN_ID` env var required on frank-ingest ACA for campaign ID fallback
- `toolTimeout: 180` already set in `config.deploy.json` (covers precinct analytics latency)

### 2026-06-28 — Volunteer tool sharing with web assistant
- Telegram volunteer MCP tools still expose Telegram schemas, resolve `telegram_id` to `users.id`, and now call shared trusted-user volunteer bot actions in frank-ingest.
- The new frank-ingest web volunteer assistant uses assistant-ui and Auth.js web identity through `/api/volunteer/chat`; it shares the same volunteer bot action logic for list lookup, voter paging, and result submission.
- V1 has no admin chat and does not treat Telegram IDs as web user identity.

### 2026-07-02 — Admin voter query narrowed, then switched to structured lookup
- frank-ingest `query_voters` is now a campaign-scoped structured voter lookup tool keyed by name/address details instead of requiring the admin to know `voter_id`; it no longer accepts arbitrary read-only SQL.
- A unique name/address match returns the full voter profile; ambiguous matches return capped candidate summaries so the admin can clarify.
- Admin bot memory now tells the model to refuse voter counts, aggregates, trends, broad lists, and arbitrary SQL through this tool.
- Use `get_voters_schema` only to explain available voter fields, not to build SQL for `query_voters`.

### Initial setup (date unknown)
- Nanobot instance configured with Azure OpenAI (gpt-5.4) and Telegram channel
- Connected to frank-ingest MCP server for data pipeline tools (voter upload, query, pipeline)
