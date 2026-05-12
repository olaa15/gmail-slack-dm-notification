# Gmail → Slack DM Notification Workflow

**One-line description:** n8n polling workflow that monitors a Gmail inbox every 10 minutes and sends a Slack DM when a new unread email arrives.
**Owner:** Ayo (Senior DevOps/Cloud Engineer transitioning to AI automation consulting)
**Purpose:** Learning exercise — rebuilding a core AIS+ training workflow from memory to reinforce the Trigger → Action automation pattern and programmatic workflow creation via MCP tools.

---

## Tech Stack

| Technology | Role |
|---|---|
| **n8n** | Workflow automation platform (self-hosted via Docker) |
| **n8n MCP Server** | Programmatic workflow creation and management via Claude Code |
| **Gmail API** | Email polling via `n8n-nodes-base.gmailTrigger` (OAuth2) |
| **Slack API** | Direct message delivery via `n8n-nodes-base.slack` (OAuth2) |
| **Docker** | n8n runtime environment |

---

## Architecture

- Two-node linear flow: **Gmail Trigger → Slack**
- Gmail Trigger polls every 10 minutes for unread emails using OAuth2 credentials
- Slack node sends a formatted DM containing: sender, subject, date, and snippet
- Polling trigger pattern (not webhook) — n8n initiates checks on a schedule
- Workflow created programmatically via n8n MCP tools, not the visual editor

---

## Key Files & Locations

- `CLAUDE.md` — this file (project context for Claude Code)
- n8n workflow — ID: `V1jrrxRM3JYx3zgd` (created 2026-05-11, inactive pending credential linking)
- Credentials — Gmail OAuth2 and Slack OAuth2 configured in the n8n UI (never stored in code or version control)

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

## Extension Ideas

- Filter node between Gmail and Slack to notify only on emails from specific senders
- Conditional branch for emails with attachments — separate Slack message
- Replace Slack DM with channel post for team-wide visibility
- Add Claude API node to summarise long emails before sending the DM
- Multi-trigger dashboard consolidating Gmail + Calendar + GitHub into a single Slack digest

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Gmail Trigger not firing | Check OAuth2 credential is valid; confirm polling interval is set; ensure workflow is **ACTIVE** |
| Slack DM not arriving | Verify `select` parameter is `"user"` (not `"channel"`); verify target user is selected |
| Expression errors | Field names must match simplified Gmail output (`from`, `subject`, `date`, `snippet`) — if `simple: false`, structure changes significantly |
| Credential errors after import | Credentials don't transfer between n8n instances — must be re-linked manually in the UI |
