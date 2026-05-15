# Gmail → Slack DM Notification Workflow

**One-line description:** n8n polling workflow that monitors a Gmail inbox every 10 minutes and sends a Slack DM when a new unread email arrives.
**Owner:** Ayo (Senior DevOps/Cloud Engineer transitioning to AI automation consulting)
**Purpose:** Learning exercise — rebuilding a core AIS+ training workflow from memory to reinforce the Trigger → Action automation pattern and programmatic workflow creation via MCP tools.

---

## n8n Workflow Development

- Always use HTTP Request node with raw content type for external API calls (Code node sandbox blocks fetch/require/$helpers)
- When configuring Respond to Webhook nodes, verify response body is populated and test with the production webhook URL (not test URL)
- Slack message field names from n8n Gmail trigger are case-sensitive: use From/Subject/Body (capitalized), not from/subject/body
- For Google Sheets nodes, verify the actual tab name after creation (CSV imports often name tabs after the file, not 'Sheet1')

---

## Deployment & Setup Defaults

- Trigger.dev: use `npx trigger.dev@latest init --yes` to avoid interactive TTY prompts
- Trigger.dev API polling: use /api/v3/ paths (not /api/v1/)
- When cloning repos, clone into ~/ (home directory) not inside the current project folder
- For Vercel deployments, always verify env vars are set in production after deploy and confirm `vercel link` points to the correct project before pushing
- For Stripe webhooks, use Unix timestamps (not ISO strings) for date fields

---

## File Operations on macOS

- Do not attempt direct file moves in protected folders (Documents, Downloads, Desktop) — write a shell script for the user to execute due to Full Disk Access restrictions
- Quote or escape '!' characters in shell scripts to avoid zsh history expansion errors
- When asked to summarize a path, verify it's a file first; if it's a directory, list contents and ask which file

---

## Tech Stack

| Technology | Role |
|---|---|
| **n8n** | Workflow automation platform (self-hosted via Docker) |
| **n8n MCP Server** | Programmatic workflow creation and management via Claude Code |
| **Gmail API** | Email polling via `n8n-nodes-base.gmailTrigger` (OAuth2) |
| **Slack API** | Direct message delivery via custom Slack app (Bot Token) |
| **Anthropic API** | Claude Haiku for email summarisation |
| **Google Calendar API** | Calendar event fetching for morning digest |
| **Docker** | n8n runtime environment |

---

## Architecture

### Workflow 1: Gmail → Slack Notifier
Full production-hardened pipeline:
```
Gmail Trigger → Dedupe Check → Filter → Summarise → Has Attachment?
                                                          │ TRUE          │ FALSE
                                                          ▼               ▼
                                                  Attachment Alert      Slack ── Error Alert
```

### Workflow 2: Morning Digest
Daily scheduled briefing:
```
Schedule Trigger (9am) → Gmail Search → Get Calendar Events → Format Digest → Slack
```

---

## Key Files & Locations

- `CLAUDE.md` — this file (project context for Claude Code)
- `docs/phase1-hardening.md` — Phase 1 implementation guide (error handling, deduplication, hosting)
- `docs/phase2-enhancements.md` — Phase 2 implementation guide (filter, AI summary, attachments, digest)
- `docs/phase3-ai-briefing.md` — Phase 3 implementation guide (AI-composed natural language morning briefing via Anthropic API)
- n8n Workflow 1 — Gmail → Slack Notifier (ID: `V1jrrxRM3JYx3zgd`, created 2026-05-11)
- n8n Workflow 2 — Morning Digest (created 2026-05-13)
- Credentials — Gmail OAuth2, Google Calendar OAuth2, Slack Bot Token — configured in n8n UI (never stored in code)

---

## Node Configuration Reference

### Gmail Trigger
```
Type:          n8n-nodes-base.gmailTrigger  v1.3
Auth:          oAuth2  (credential: "Gmail OAuth2")
Event:         messageReceived
Simplify:      true
Poll interval: every 10 minutes (pollTimes)
Filter:        unread only  (readStatus: "unread")
```

### Slack
```
Type:       n8n-nodes-base.slack  v2.4
Auth:       oAuth2  (credential: "Slack OAuth2")
Resource:   message
Operation:  post
Send to:    user  (DM, not channel)
Message:    text
```

Message template:
```
New email from {{ $json.from }}
Subject: {{ $json.subject }}
Date: {{ $json.date }}
---
{{ $json.snippet }}
```

---

## n8n Expression Syntax

```
{{ $json.fieldName }}              # Access field from previous node
{{ $json.payload.headers }}        # Nested field access
{{ $json.field ? $json.field : "fallback" }}  # Conditional with fallback
```

Expressions go in node parameter values, wrapped in `{{ }}`.

---

## Commands & Operations

| Task | MCP Tool |
|---|---|
| Create workflow | `n8n_create_workflow` (pass nodes, connections, name, settings) |
| Validate workflow | `n8n_validate_workflow` (pass workflow ID) |
| Full update | `n8n_update_full_workflow` |
| Partial update | `n8n_update_partial_workflow` |
| Test workflow | `n8n_test_workflow` (webhook/form/chat triggers only — polling triggers must be tested via UI) |
| Inspect node | `get_node` with `nodeType` parameter |
| Search nodes | `search_nodes` with query string |

---

## Conventions

- All workflows created **INACTIVE** — activate manually in the n8n UI after credential linking and testing
- Credentials never hardcoded — referenced by name, linked via UI
- Connections use node **NAMES** as keys, not node IDs
- Node positions: left-to-right — trigger at `x=250`, action nodes at `x=500`, `x=750`, etc.
- Timezone: `Europe/London`
- Execution order: `v1`

---

## Completed Enhancements

- [x] Error handler branch — Slack Error Alert on workflow failure
- [x] Duplicate email filter — static data deduplication via Code node
- [x] Smart keyword filter — IF node filters on `urgent` subject keyword
- [x] AI summarisation — Claude Haiku summarises emails before Slack DM
- [x] Attachment detection — separate Slack alert for emails with attachments
- [x] Morning digest — daily 9am Slack DM combining Gmail + Google Calendar
- [x] Weather in morning digest — Open-Meteo HTTP Request node added between Get Calendar Events and Format Digest; no API key required
- [x] AI morning briefing (Phase 3) — Compose Briefing HTTP Request node calls Anthropic API (claude-haiku-4-5-20251001); Format Digest simplified to extract natural language response

## Extension Ideas (Backlog)

- Move off localhost — deploy n8n to Railway, DigitalOcean, or n8n Cloud
- Filter by sender — extend the IF node to also match specific email addresses
- GitHub integration — add GitHub notifications to the morning digest
- Weekly summary — aggregate digest covering the past 7 days

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Gmail Trigger not firing | Check OAuth2 credential is valid; confirm polling interval is set; ensure workflow is **ACTIVE** |
| Slack DM not arriving | Verify `select` parameter is `"user"` (not `"channel"`); verify target user is selected |
| Expression errors | Field names must match simplified Gmail output (`from`, `subject`, `date`, `snippet`) — if `simple: false`, structure changes significantly |
| Credential errors after import | Credentials don't transfer between n8n instances — must be re-linked manually in the UI |
