# Outstanding Tasks

This file tracks work that is planned but not yet implemented. Anyone picking up this project should start here.

---

## Backlog

### 1. Move off localhost
**Priority:** High (required before sharing with others or running reliably 24/7)

The n8n instance currently runs locally via Docker. If the machine sleeps or restarts, workflows stop.

**Options:**
- [n8n Cloud](https://app.n8n.cloud) — easiest, managed hosting
- Railway — cheap self-hosted option (~$5/month)
- DigitalOcean Droplet — more control, more setup

**Steps:**
1. Export all workflows from n8n UI (Settings → Export)
2. Deploy n8n to chosen platform
3. Re-import workflows and re-link credentials (Gmail OAuth2, Google Calendar OAuth2, Slack Bot Token)
4. Activate workflows and verify they fire correctly

---

### 2. Filter by sender
**Priority:** Medium

The Gmail → Slack Notifier currently filters only on the `urgent` keyword in the subject line. Extend the IF node to also match emails from specific senders (e.g. your manager, a key client).

**Where to change:** Workflow 1 (Gmail → Slack Notifier) → IF node → add an OR condition on the `From` field.

**Example condition to add:**
```
{{ $json.From }} contains "boss@company.com"
```

---

### 3. GitHub integration in morning digest
**Priority:** Medium

Add a GitHub node to the Morning Digest workflow that pulls in open PRs or issues assigned to you, and includes them in the 9am Slack DM.

**Where to change:** Workflow 2 (Morning Digest) — add a GitHub node after Get Weather, before Format Digest.

**n8n node:** `n8n-nodes-base.github`
- Resource: Pull Request or Issue
- Operation: Get Many
- Filter: assigned to authenticated user, state: open

**Format Digest Code node** will need a new section:
```javascript
const prs = $('Get GitHub PRs').all();
const prSection = prs.length > 0
  ? prs.map(p => `• [#${p.json.number}] ${p.json.title}`).join('\n')
  : "No open PRs";
```

---

### 4. Weekly summary workflow
**Priority:** Low

A new Workflow 3 that fires once a week (e.g. Friday 5pm) and sends a Slack DM summarising the past 7 days of emails and calendar events.

**Nodes needed:**
- Schedule Trigger — weekly recurrence (Friday 17:00, Europe/London)
- Gmail Search — query: `newer_than:7d` (no urgent filter — broader sweep)
- Google Calendar — date range: last 7 days
- Code node — format weekly summary
- Slack — DM

---

## Completed

See the **Completed Enhancements** section in [CLAUDE.md](CLAUDE.md) for everything already built and working.
