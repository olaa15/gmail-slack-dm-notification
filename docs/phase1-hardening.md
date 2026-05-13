# Phase 1 — Hardening: Making the Workflow Production-Safe

## Overview

A workflow that breaks silently is worse than no workflow. Phase 1 makes the Gmail → Slack notifier resilient, reliable, and ready to run 24/7 without manual supervision.

---

## Tasks

| Task | What You Build | What You Learn |
|---|---|---|
| Task 1 | Error handler branch | Branching logic, error states, defensive automation |
| Task 2 | Duplicate email filter | Conditional logic, stateful workflows |
| Task 3 | Move off localhost | Production deployment, real hosting |

---

## Task 1 — Add an Error Handler

### The Problem

Right now if the Slack API goes down, your Gmail OAuth token expires, or n8n hits any unexpected error — the workflow fails silently. You would never know. Emails would be missed with no alert.

### The Solution

Add a second output branch to the workflow that fires **only when something goes wrong**, sending a different Slack message:

```
⚠️ Gmail Notifier failed — check n8n
```

### How n8n Error Handling Works

Every node in n8n has two output paths:
- **Main output** — fires when the node succeeds
- **Error output** — fires when the node fails (must be enabled)

You connect the error output to a second action node. This is called an **error branch**.

---

### Step-by-Step Implementation

#### Step 1 — Open the workflow in n8n

1. Go to `http://localhost:5678`
2. Open the **Gmail → Slack DM Notification** workflow
3. You should see the two existing nodes: Gmail Trigger and Slack

#### Step 2 — Enable error output on the Slack node

1. Click the **Slack** node to select it
2. Click the **Settings** tab (top of the node panel)
3. Find **"Continue On Fail"** — leave this OFF
4. Close the panel
5. Hover over the Slack node — you will see a small **red circle with a cross** appear on the node's edge. This is the error output handle.

#### Step 3 — Add a second Slack node for the error alert

1. Click the **+** button to add a new node
2. Search for **Slack** and select it
3. Configure it as follows:

| Parameter | Value |
|---|---|
| Credential | Slack OAuth2 |
| Resource | Message |
| Operation | Send |
| Send Message To | User |
| User | (select yourself) |
| Message Type | Simple Text Message |
| Message Text | (see below) |

**Error alert message text:**
```
⚠️ *Gmail Notifier — Workflow Error*

Something went wrong in your Gmail → Slack notification workflow.

*Time:* {{ $now.toLocaleString('en-GB', {timeZone: 'Europe/London'}) }}
*Workflow:* Gmail → Slack DM Notification

Action required: Log in to n8n and check the execution log.
http://localhost:5678/executions
```

4. Name this node **"Slack — Error Alert"** (double-click the node title to rename it)
5. Position it below the main Slack node

#### Step 4 — Connect the error branch

1. Hover over the original **Slack** node
2. You will see the red error output handle on the right edge
3. Drag from that red handle to the **Slack — Error Alert** node
4. A red connection line will appear — this is your error branch

#### Step 5 — Test the error handler

You need to trigger a failure deliberately to confirm the error branch works.

1. Temporarily break the Slack node:
   - Click the original **Slack** node
   - Change the credential to a non-existent one, or clear the User field
2. Click **Execute Workflow**
3. The main Slack node should fail (shown in red)
4. The **Slack — Error Alert** node should fire and send the alert DM
5. Fix the original Slack node back to the correct settings
6. Execute again to confirm the happy path still works

#### Step 6 — Save the workflow

Click **Save** in the top right.

---

### What the Workflow Looks Like After Task 1

```
Gmail Trigger
     │
     ▼
   Slack ──────────────────── (error) ──▶ Slack — Error Alert
     │
     ▼
  (success: DM sent)                     (failure: alert sent)
```

---

### Common Mistakes

- **Connecting from the wrong handle** — the error handle is red/orange. The main output handle is grey. Make sure you drag from the correct one.
- **Forgetting to test the failure path** — always verify the error branch actually fires. Untested error handlers are worthless.
- **Using the same message text** — the error alert must look visually different from a normal email notification so you can tell them apart at a glance.

---

### Definition of Done

Task 1 is complete when:
- [x] A second Slack node exists named "Slack — Error Alert"
- [x] It is connected via the error output of the original Slack node (red line)
- [x] You have deliberately broken the workflow and confirmed the alert fires
- [x] You have fixed the workflow and confirmed the normal DM still works
- [x] The workflow is saved

---

## Task 2 — Add a Duplicate Email Filter

### The Problem

If n8n restarts or the Gmail poll fires twice in quick succession, the same email can trigger duplicate Slack DMs. The workflow needs memory of which emails it has already processed.

### The Solution

A Code node that stores processed email IDs in n8n's built-in static data. Before sending a Slack DM, it checks whether the email has already been processed. If yes — drop it silently. If no — add to the list and pass through.

### What the Workflow Looks Like After Task 2

```
Gmail Trigger
     │
     ▼
Dedupe Check  (Code node — new)
     │
     ▼
   Slack ──── (error) ──▶ Slack — Error Alert
```

### Step-by-Step Implementation

#### Step 1 — Add a Code node
1. Click **+** between the Gmail Trigger and Slack nodes
2. Search for **Code** and select it
3. Name it **"Dedupe Check"**

#### Step 2 — Paste this code

```javascript
const staticData = $getWorkflowStaticData('global');

if (!staticData.processedIds) {
  staticData.processedIds = [];
}

const emailId = $input.first().json.id;

if (staticData.processedIds.includes(emailId)) {
  return [];  // Already seen — stop execution
}

staticData.processedIds.push(emailId);

// Cap at 100 IDs to prevent unbounded growth
if (staticData.processedIds.length > 100) {
  staticData.processedIds = staticData.processedIds.slice(-100);
}

return $input.all();  // New email — pass through
```

#### Step 3 — Wire it up
- Connect: `Gmail Trigger → Dedupe Check → Slack`
- The error branch from Slack stays as-is

#### Step 4 — Test it
1. Activate the workflow and send a test email
2. The first poll fires — one DM arrives
3. The next poll fires for the same email — no second DM (dropped by Dedupe Check)

> **Note:** `$getWorkflowStaticData` does not persist during manual "Execute Workflow" runs. It only persists when the workflow is active and running via the trigger. Always test deduplication with the workflow active.

### Common Mistakes

- **Testing with manual Execute** — static data resets on manual runs. Only test deduplication with the workflow active.
- **Sending to channel instead of DM** — "Send Message To" must be **User**, not Channel.

### Definition of Done

- [x] "Dedupe Check" Code node sits between Gmail Trigger and Slack
- [x] First execution sends the DM
- [x] Second execution with the same email is silently dropped
- [x] Error branch from Slack still works
- [x] Workflow saved

---

## Task 3 — Move off localhost

### Status: Backlog

Moving the n8n instance off localhost to a production host so the workflow runs 24/7 without the local machine being on.

### Hosting Options

| Option | Cost | Effort |
|---|---|---|
| n8n Cloud | ~$20/month | Lowest — import workflow and go |
| Railway | Free tier available | Low — one-click Docker deploy |
| DigitalOcean Droplet | ~$6/month | Medium — Docker Compose on Linux VPS |

### Definition of Done

- [ ] n8n running on a remote host
- [ ] Both workflows imported and active
- [ ] Credentials re-linked on the new instance
- [ ] Workflow polling confirmed working without local machine
