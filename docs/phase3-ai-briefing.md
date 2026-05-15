# Phase 3: AI Morning Briefing

## Goal

Upgrade the Morning Digest from a code-formatted bullet list into a natural-language briefing composed by Claude (Anthropic API). Instead of structured sections, the Slack DM reads like a concise human-written summary of your day.

---

## Why This Matters

The current Morning Digest concatenates data into fixed sections. Phase 3 adds an AI node that synthesises all data sources — emails, calendar events, weather — into a single coherent paragraph or short briefing. This is:

- More readable at a glance
- Directly reusable for clients (swap data sources, same AI skeleton)
- A meaningful step up in automation complexity

---

## Architecture

```
Schedule Trigger (9am)
    → Gmail Search
    → Get Calendar Events
    → Get Weather (Open-Meteo)
    → Compose Briefing (HTTP Request → Anthropic API)
    → Slack DM
```

Seven nodes total. The key addition is the **Compose Briefing** node.

---

## Prerequisites

Before building, have these ready:

| Credential | Where to get it |
|---|---|
| Anthropic API key | console.anthropic.com |
| (Optional) OpenWeatherMap key | openweathermap.org — free tier; or keep using Open-Meteo (no key needed) |

---

## New Node: Compose Briefing

**Type:** HTTP Request node (the Code node sandbox blocks `fetch`, so use HTTP Request to call the Anthropic API directly)

**Method:** POST  
**URL:** `https://api.anthropic.com/v1/messages`

**Headers:**
```
x-api-key: YOUR_ANTHROPIC_API_KEY
anthropic-version: 2023-06-01
content-type: application/json
```

**Body (JSON):**
```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 300,
  "messages": [
    {
      "role": "user",
      "content": "Write a concise morning briefing based on this data:\n\nWEATHER: {{ $('Get Weather').item.json.current.temperature_2m }}°C, weathercode {{ $('Get Weather').item.json.current.weathercode }}\n\nEMAILS: {{ $('Gmail Search').all().map(e => e.json.Subject).join(', ') || 'None' }}\n\nCALENDAR: {{ $('Get Calender Events').all().map(e => e.json.summary + ' at ' + (e.json.start?.dateTime || e.json.start?.date)).join(', ') || 'Nothing scheduled' }}\n\nKeep it to 3-4 sentences. Friendly, professional tone. Start with the weather, then key meetings, then any urgent emails."
    }
  ]
}
```

> Note: n8n expressions inside JSON body strings must be written carefully. Use the n8n expression editor to verify interpolation works before saving.

---

## Format Digest Code Node Changes

Replace the current Format Digest Code node with a simpler node that just extracts the AI response:

```javascript
const aiResponse = $input.item.json.content[0].text;

return [{
  json: {
    message: `*🌅 Morning Briefing — ${$now.toFormat('dd MMM yyyy')}*\n\n${aiResponse}`
  }
}];
```

---

## Key Pattern: Reaching Back to Earlier Nodes

The Compose Briefing node (and the Code node) can pull data from *any* upstream node using:

```javascript
$('Node Name').all()        // returns array of all items from that node
$('Node Name').item.json    // returns the first item's data
```

This is the core n8n multi-source pattern — once understood, any multi-input workflow becomes straightforward.

---

## Reusability for Clients

The architecture is identical whether briefing yourself or a client:

| Swap this | For this |
|---|---|
| Google Calendar | Client's booking system |
| Gmail | Client's supplier email |
| Weather | Stock levels, delivery status, sales figures |

Same trigger → fetch → AI compose → deliver skeleton, different data sources.

---

## Testing

1. Execute each node individually to verify data shapes
2. Check the Compose Briefing node output — look for `content[0].text` in the response
3. Run the full workflow and confirm the Slack DM reads as natural prose
4. If the AI response feels too long or short, adjust `max_tokens` and the prompt instruction

---

## Status

- [ ] Not yet started
