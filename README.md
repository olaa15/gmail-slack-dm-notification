# Gmail → Slack Automation Suite

A production-hardened n8n automation suite built in two phases. It monitors a Gmail inbox, filters and summarises emails with AI, detects attachments, and delivers a daily morning briefing combining email and calendar data — all via Slack DM.

---

## Workflows

### 1. Gmail → Slack Notifier

Polls Gmail every 10 minutes. When an unread email with `urgent` in the subject arrives, it deduplicates, summarises with Claude AI, detects attachments, and sends a formatted Slack DM.

```
Gmail Trigger → Dedupe Check → Filter → Summarise (Claude) → Has Attachment?
                                                                  │ TRUE          │ FALSE
                                                                  ▼               ▼
                                                          Attachment Alert      Slack DM
                                                                        ↘      ↙
                                                                      Error Alert
```

**Slack DM format (normal email):**
```
New email from Ayo <ayo.ai.automation@gmail.com>
Subject: urgent: verify your account

Summary:
Ayo is notifying you that you need to complete a verification
step before you can send messages from your toll-free number.
The process is quick to complete.
```

**Slack DM format (email with attachment):**
```
📎 New email WITH ATTACHMENT from Ayo <ayo.ai.automation@gmail.com>
Subject: urgent: contract attached

Summary:
...

Action: Log in to Gmail to view the attachment.
```

---

### 2. Morning Digest

Fires at 9am every day. Fetches overnight urgent unread emails and today's Google Calendar events, then sends a single structured Slack DM.

```
Schedule Trigger (9am) → Gmail Search → Get Calendar Events → Format Digest → Slack DM
```

**Slack DM format:**
```
🌅 Morning Digest — 13 May 2026

📧 Urgent Emails:
• From: Ayo <ayo.ai.automation@gmail.com>
  Subject: urgent: action required

📅 Today's Calendar:
• Team standup — 2026-05-13T09:30:00+01:00
• Client call — 2026-05-13T14:00:00+01:00
```

---

## Tech Stack

| Component | Technology |
|---|---|
| Workflow automation | n8n (self-hosted via Docker) |
| Email source | Gmail API (OAuth2) |
| Notification delivery | Slack API (custom bot, Bot Token) |
| AI summarisation | Anthropic Claude Haiku |
| Calendar data | Google Calendar API (OAuth2) |
| Runtime | Docker |

---

## Project Phases

### Phase 1 — Hardening

Making the workflow production-safe.

| Task | Status | Description |
|---|---|---|
| Task 1 — Error handler | ✅ Done | Error branch sends Slack alert on failure |
| Task 2 — Deduplicate | ✅ Done | Static data prevents duplicate DMs |
| Task 3 — Move off localhost | ⏳ Backlog | Deploy to cloud host for 24/7 operation |

See [docs/phase1-hardening.md](docs/phase1-hardening.md) for full implementation steps.

### Phase 2 — Enhancements

Making the workflow smarter.

| Task | Status | Description |
|---|---|---|
| Task 1 — Smart filter | ✅ Done | IF node filters on `urgent` keyword |
| Task 2 — AI summary | ✅ Done | Claude Haiku summarises emails |
| Task 3 — Attachment detection | ✅ Done | Separate alert for emails with files |
| Task 4 — Morning digest | ✅ Done | Daily Gmail + Calendar briefing |

See [docs/phase2-enhancements.md](docs/phase2-enhancements.md) for full implementation steps.

---

## Setup Guide

### Prerequisites

- n8n instance running (self-hosted via Docker)
- Google Cloud project with Gmail API and Google Calendar API enabled
- Slack workspace with a custom bot app created

### Credentials Required

| Credential | Used By |
|---|---|
| Gmail OAuth2 | Gmail Trigger, Gmail Search |
| Google Calendar OAuth2 | Get Calendar Events |
| Slack Bot Token (`xoxb-`) | All Slack nodes |
| Anthropic API Key | Summarise node |

### Slack App Setup

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From scratch**
2. Name: `n8n Notifier`, select your workspace
3. Go to **OAuth & Permissions → Bot Token Scopes**, add:
   - `chat:write`, `im:write`, `im:read`, `users:read`, `channels:read`
4. Click **Install to Workspace** → copy the `xoxb-` Bot Token
5. In n8n: **Credentials → Add → Slack → Access Token** → paste the token

### Gmail + Google Calendar OAuth2 Setup

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a project → enable **Gmail API** and **Google Calendar API**
3. Create an OAuth 2.0 Client ID (Web application)
4. Add redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
5. In n8n: create **Gmail OAuth2** and **Google Calendar OAuth2** credentials using the Client ID and Secret

### Importing the Workflows

1. In n8n go to **Workflows → Import**
2. Import `workflow.json`
3. Link credentials to each node
4. Activate the workflow

---

## Key Lessons

- Gmail Trigger with **Simplify ON** outputs capitalised field names: `$json.Subject`, `$json.From`
- `$getWorkflowStaticData` only persists when the workflow runs via an active trigger, not during manual test runs
- Slack bots require `im:write` scope to send DMs — `chat:write` alone is not sufficient
- Google Calendar node should have **Execute Once** enabled when upstream nodes produce multiple items
- Anthropic node Role must be **User** — the API rejects Assistant as the final message

---

## Project Structure

```
├── workflow.json              # n8n workflow export (Gmail → Slack Notifier)
├── CLAUDE.md                  # Developer reference and project context
├── README.md                  # This file
└── docs/
    ├── phase1-hardening.md    # Phase 1 implementation guide
    └── phase2-enhancements.md # Phase 2 implementation guide
```

---

## Author

Built by **Ayo** — Senior DevOps & Cloud Engineer transitioning into AI automation consulting.
Part of a series of workflow automation projects demonstrating practical AI integration patterns.
