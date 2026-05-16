# Outstanding Tasks

This file tracks work that is planned but not yet implemented. Anyone picking up this project should start here.

---

## Phase 3: AI Morning Briefing
**Status:** Complete (2026-05-15) — full implementation guide in [docs/phase3-ai-briefing.md](docs/phase3-ai-briefing.md)

Upgrade the Morning Digest from a code-formatted bullet list into a natural-language briefing composed by Claude (Anthropic API). Requires an Anthropic API key. Architecture: Schedule Trigger → Gmail Search → Get Calendar Events → Get Weather → Compose Briefing (HTTP Request to Anthropic) → Slack DM.

---

## Backlog

### ~~1. Move off localhost~~ — DONE (2026-05-15)
Both workflows migrated to **n8n Cloud** (free trial). Credentials re-linked, workflows activated and verified.

---

### ~~2. Filter by sender~~ — DONE (2026-05-16)
Filter node in Workflow 1 updated with three OR conditions: Subject contains `urgent`, OR From contains `seyebadmus@gmail.com`, OR From contains `ola.ogunseye@outlook.com`. Use "contains" (not "is equal to") because Gmail's From field includes the display name, e.g. `"Name <email@address.com>"`.

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
