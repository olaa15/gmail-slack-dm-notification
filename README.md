# Gmail → Slack DM Notification

An automated workflow built with [n8n](https://n8n.io) that monitors a Gmail inbox and instantly notifies you via Slack direct message whenever a new unread email arrives.

---

## Overview

This workflow eliminates the need to constantly check your inbox. The moment a new email lands, you receive a structured Slack DM containing everything you need to decide whether to act — sender, subject, date, and a preview of the message body.

**Trigger:** Gmail inbox (polls every 10 minutes)  
**Action:** Slack direct message to a specified user  
**Platform:** n8n (self-hosted)

---

## How It Works

```
Gmail Inbox  →  n8n Polling Trigger  →  Slack DM
```

1. Every 10 minutes, n8n checks the connected Gmail account for new unread emails
2. When one is found, it extracts the key details from the email
3. A formatted direct message is sent to the configured Slack user immediately

---

## Slack Message Format

Each notification arrives as a clean, readable DM:

```
📧 New email received!

From: Sender Name <sender@example.com>
Subject: Meeting follow-up
Date: 12/05/2026, 09:30:00

Hi Ayo, just following up on our conversation yesterday...

Automated with this n8n workflow
```

---

## Tech Stack

| Component | Technology |
|---|---|
| Workflow automation | [n8n](https://n8n.io) (self-hosted via Docker) |
| Email source | Gmail API (OAuth2) |
| Notification delivery | Slack API (OAuth2) |
| Runtime | Docker |

---

## Prerequisites

- n8n instance running (self-hosted or cloud)
- A Gmail account with API access enabled
- A Slack workspace with permission to create apps
- OAuth2 credentials for both Gmail and Slack (see setup below)

---

## Setup Guide

### 1. Import the Workflow

In your n8n instance:

1. Go to **Workflows → Import**
2. Upload `workflow.json` from this repository
3. The workflow will be imported in an **inactive** state

### 2. Create Slack OAuth2 Credentials

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and click **Create New App → From scratch**
2. Navigate to **OAuth & Permissions**
3. Under **Redirect URLs**, add:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
4. Under **Bot Token Scopes**, add: `chat:write`, `users:read`, `im:write`
5. Click **Install to Workspace** and authorise
6. In n8n, go to **Credentials → New → Slack OAuth2 API**
7. Enter your app's **Client ID** and **Client Secret** (from Basic Information in your Slack app)
8. Click **Connect my account** and complete the OAuth flow
9. Name the credential `Slack OAuth2` and save

### 3. Create Gmail OAuth2 Credentials

1. Go to [Google Cloud Console](https://console.cloud.google.com) and create a new project
2. Enable the **Gmail API**
3. Create an **OAuth 2.0 Client ID** (Application type: Web application)
4. Add the redirect URI:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
5. In n8n, go to **Credentials → New → Gmail OAuth2 API**
6. Enter your **Client ID** and **Client Secret**
7. Click **Connect my account** and authorise access
8. Name the credential `Gmail OAuth2` and save

### 4. Configure the Workflow

1. Open the imported workflow in n8n
2. Click the **Slack** node
3. Link the `Slack OAuth2` credential
4. In the **User** field, select the Slack user who should receive the DMs
5. Click the **Gmail Trigger** node and link the `Gmail OAuth2` credential
6. Save the workflow

### 5. Activate

Toggle the workflow from **Inactive** to **Active** in the top-right of the canvas. The workflow will begin polling immediately.

---

## Workflow Configuration

| Setting | Value |
|---|---|
| Poll interval | Every 10 minutes |
| Email filter | Unread only |
| Timezone | Europe/London |
| Execution order | v1 |

---

## Customisation

**Change the poll interval**  
Open the Gmail Trigger node and adjust the polling interval (e.g. every 5 minutes, every hour).

**Filter by sender or subject**  
Add an n8n Filter node between the Gmail Trigger and Slack nodes to only notify on emails matching specific criteria.

**Post to a Slack channel instead of a DM**  
In the Slack node, change **Send Message To** from `User` to `Channel` and select your target channel.

**Add an AI summary**  
Insert a Claude or OpenAI node between Gmail and Slack to summarise long emails before the DM is sent.

---

## Project Structure

```
├── workflow.json   # Exportable n8n workflow definition
├── CLAUDE.md       # Developer reference (node specs, expressions, troubleshooting)
└── README.md       # This file
```

---

## Author

Built by **Ayo** — Senior DevOps & Cloud Engineer transitioning into AI automation consulting.  
Part of a series of workflow automation projects demonstrating practical AI and integration patterns.
