# Phase 2 — Enhancements: Making the Workflow Smarter

## Overview

Phase 2 adds intelligence and breadth to the Gmail → Slack notifier. Each task makes the workflow more useful in daily practice — filtering noise, summarising content with AI, detecting attachments, and consolidating multiple data sources into a single morning briefing.

---

## Tasks

| Task | What You Build | What You Learn |
|---|---|---|
| Task 1 | Keyword filter | IF node logic, conditional routing |
| Task 2 | AI email summarisation | Anthropic API integration, Claude in n8n |
| Task 3 | Attachment detection | mimeType inspection, parallel Slack branches |
| Task 4 | Morning digest | Schedule trigger, Google Calendar, multi-source merge |

---

## Task 1 — Smart Email Filter

### The Problem

Every unread email triggers a Slack DM. Most emails don't need immediate attention. We want to notify only on emails that are genuinely urgent.

### The Solution

An IF node that checks the email subject for the keyword `urgent`. Only matching emails proceed to Slack — everything else is silently dropped.

### What the Workflow Looks Like After Task 1

```
Gmail Trigger
     │
     ▼
Dedupe Check
     │
     ▼
Filter (IF node — new)
     │ TRUE                    FALSE
     ▼                           ▼
   Slack ── (error) ──▶ Error Alert    (dropped — no action)
```

### Step-by-Step Implementation

#### Step 1 — Add an IF node
1. Click **+** between **Dedupe Check** and **Slack**
2. Search for **IF** and select it
3. Name it **"Filter"**

#### Step 2 — Configure the condition

| Setting | Value |
|---|---|
| Value 1 | `{{ $json.Subject }}` *(capital S — Simplify output)* |
| Operation | **Contains** |
| Value 2 | `urgent` |

> **Important:** Enable **Ignore Case** so `Urgent`, `URGENT`, and `urgent` all match.

#### Step 3 — Wire it up
- **TRUE output** → connect to **Slack** node
- **FALSE output** → leave unconnected (execution ends silently)

### Troubleshooting

| Symptom | Fix |
|---|---|
| Everything goes to FALSE | `$json.Subject` (capital S) — Simplify outputs capitalised field names |
| Condition backwards | Value 1 must be the expression, Value 2 must be the keyword |
| `multipart/alternative` payload showing | Gmail Trigger Simplify must be ON |

### Definition of Done

- [x] IF node named "Filter" sits between Dedupe Check and Slack
- [x] Emails with "urgent" in subject reach Slack
- [x] Emails without "urgent" are silently dropped
- [x] Ignore Case enabled

---

## Task 2 — AI Email Summarisation

### The Problem

The raw email snippet can be long, truncated, or filled with HTML formatting. A clean 2-3 sentence summary is more useful in a Slack DM.

### The Solution

An Anthropic node (Claude Haiku) that reads the email content and returns a concise summary before the Slack message is sent.

### What the Workflow Looks Like After Task 2

```
Gmail Trigger → Dedupe Check → Filter → Summarise → Slack → Error Alert
```

### Step-by-Step Implementation

#### Step 1 — Add an Anthropic node
1. Click **+** between **Filter** and **Slack**
2. Search for **Anthropic** and select it
3. Name it **"Summarise"**

#### Step 2 — Configure the node

| Parameter | Value |
|---|---|
| Credential | Anthropic API key |
| Resource | Text |
| Operation | Message a Model |
| Model | `claude-haiku-4-5-20251001` |
| Simplify Output | ON |

**Prompt (Role: User):**
```
Summarise this email in 2-3 concise sentences. Focus on what the sender wants or needs.

From: {{ $('Filter').item.json.From }}
Subject: {{ $('Filter').item.json.Subject }}
Email: {{ $('Filter').item.json.snippet }}
```

**System prompt (add via Options):**
```
You are an email assistant. Be direct and professional.
```

#### Step 3 — Update the Slack message template

```
New email from {{ $('Filter').item.json.From }}
Subject: {{ $('Filter').item.json.Subject }}

Summary:
{{ $('Summarise').item.json.content[0].text }}
```

### Key Notes

- Role must be **User** — not Assistant (API rejects assistant as the final message)
- With Simplify Output ON, the summary text is at `$json.content[0].text`
- Slack DMs arrive from the **n8n Notifier** bot, not a person

### Definition of Done

- [x] Anthropic node sits between Filter and Slack
- [x] Claude Haiku summarises the email in 2-3 sentences
- [x] Slack DM shows the summary instead of the raw snippet

---

## Task 3 — Attachment Detection

### The Problem

Emails with attachments need different handling — the recipient needs to know to log into Gmail and download the file, not just read the summary.

### The Solution

An IF node that inspects the email's `mimeType`. If it is `multipart/mixed` (Gmail's indicator for attachments), route to a separate Slack alert. Otherwise route to the normal DM.

### What the Workflow Looks Like After Task 3

```
Gmail Trigger → Dedupe Check → Filter → Summarise → Has Attachment? (IF)
                                                          │ TRUE          │ FALSE
                                                          ▼               ▼
                                                  Attachment Alert      Slack
                                                  (ends here)         (ends here)
```

### Step-by-Step Implementation

#### Step 1 — Add an IF node after Summarise
1. Click **+** after **Summarise**
2. Search for **IF** and select it
3. Name it **"Has Attachment?"**

#### Step 2 — Configure the condition

| Setting | Value |
|---|---|
| Value 1 | `{{ $('Filter').item.json.payload.mimeType }}` |
| Operation | **Equals** |
| Value 2 | `multipart/mixed` |

**mimeType reference:**
- `multipart/mixed` → email has an attachment
- `multipart/alternative` → plain text + HTML only, no attachment

#### Step 3 — Add Slack — Attachment Alert node
- Connect from **TRUE** output
- Same credential and User as main Slack node
- Message Text:

```
📎 New email WITH ATTACHMENT from {{ $('Filter').item.json.From }}
Subject: {{ $('Filter').item.json.Subject }}

Summary:
{{ $('Summarise').item.json.content[0].text }}

Action: Log in to Gmail to view the attachment.
```

#### Step 4 — Connect FALSE to existing Slack node
- Connect **FALSE** output → existing **Slack** node
- Do NOT chain Attachment Alert into Slack — they are two separate dead ends

### Definition of Done

- [x] "Has Attachment?" IF node sits after Summarise
- [x] TRUE → Attachment Alert Slack DM (with action note)
- [x] FALSE → normal Slack DM (summary only)
- [x] Both paths tested and confirmed working

---

## Task 4 — Morning Digest

### The Problem

The existing workflow is reactive — it fires when an email arrives. A morning digest is proactive — at 9am every day, get a single Slack DM summarising overnight urgent emails and today's calendar.

### The Solution

A separate n8n workflow triggered by a daily schedule. It fetches urgent unread emails from Gmail and events from Google Calendar, then formats and sends a single structured Slack DM.

### Workflow: Morning Digest (separate workflow)

```
Schedule Trigger (9am daily)
     │
     ▼
Gmail Search (getAll: unread + urgent)
     │
     ▼
Get Calendar Events (today's events)
     │
     ▼
Format Digest (Code node)
     │
     ▼
Send a message (Slack DM)
```

### Step-by-Step Implementation

#### Step 1 — Create a new workflow
- Name: **"Morning Digest"**

#### Step 2 — Schedule Trigger

| Setting | Value |
|---|---|
| Trigger interval | Days |
| Days between triggers | `1` |
| Trigger at hour | `9` |
| Trigger at minute | `0` |
| Timezone | `Europe/London` |

#### Step 3 — Gmail Search node

| Setting | Value |
|---|---|
| Credential | Gmail OAuth2 |
| Resource | Message |
| Operation | Get Many |
| Search query | `is:unread subject:urgent newer_than:1d` |
| Simplify | ON |
| Limit | `10` |

#### Step 4 — Google Calendar node

**Prerequisites:**
1. In Google Cloud Console, enable the **Google Calendar API** on your existing project
2. Add scope `https://www.googleapis.com/auth/calendar.readonly` to your OAuth consent screen
3. In n8n create a **Google Calendar OAuth2** credential using the same Client ID/Secret
4. In the node Settings tab, enable **Execute Once** (prevents running once per email input item)

| Setting | Value |
|---|---|
| Credential | Google Calendar OAuth2 |
| Resource | Event |
| Operation | Get Many |
| Calendar | `ayo.ai.automation@gmail.com` (primary) |
| After | `{{ $now.startOf('day').toISO() }}` |
| Before | `{{ $now.endOf('day').toISO() }}` |

#### Step 5 — Format Digest (Code node)

```javascript
// Get Gmail results
const emails = $('Gmail Search').all();

// Get Calendar results from input
const events = $input.all();

// Format emails section
let emailSection = '';
if (emails.length === 0) {
  emailSection = 'No urgent unread emails today.';
} else {
  emailSection = emails.map(e => 
    `• From: ${e.json.From}\n  Subject: ${e.json.Subject}`
  ).join('\n');
}

// Format calendar section
let calendarSection = '';
if (events.length === 0) {
  calendarSection = 'No events scheduled today.';
} else {
  calendarSection = events.map(e => 
    `• ${e.json.summary} — ${e.json.start?.dateTime || e.json.start?.date}`
  ).join('\n');
}

return [{
  json: {
    message: `*🌅 Morning Digest — ${$now.toFormat('dd MMM yyyy')}*\n\n*📧 Urgent Emails:*\n${emailSection}\n\n*📅 Today's Calendar:*\n${calendarSection}`
  }
}];
```

#### Step 6 — Slack node

| Setting | Value |
|---|---|
| Credential | Slack n8n Notifier |
| Send Message To | User |
| User | yourself |
| Message Text | `{{ $json.message }}` |

### Common Issues

| Symptom | Fix |
|---|---|
| `events is not defined` error | Ensure `const events = $input.all()` is declared before line 17 |
| Calendar node runs 10 times | Enable **Execute Once** in the Calendar node Settings tab |
| No calendar output | Create a test event in Google Calendar for today |
| Referenced node doesn't exist | Node name in code must exactly match the canvas node name |

### Definition of Done

- [x] Morning Digest is a separate workflow from the Gmail notifier
- [x] Schedule trigger fires at 9am Europe/London
- [x] Gmail Search fetches urgent unread emails from last 24 hours
- [x] Google Calendar fetches today's events
- [x] Format Digest combines both into one message
- [x] Slack DM arrives with both sections populated
