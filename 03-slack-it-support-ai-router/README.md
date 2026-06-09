# Workflow 03 — Slack IT Support + AI Router

**Client:** Meridian Corp (fictional)
**Stack:** n8n · OpenRouter (GPT-4.1-mini) · Slack · Google Sheets
**Pattern:** AI ticket triage — Slack message → AI classifies priority → route to right team → reply in thread

---

## The Problem

Meridian Corp's IT team receives support requests in a Slack channel. Without automation, every message sits unread until someone spots it. Critical outages wait in the same queue as minor requests. There is no consistent response, no record, and no escalation path.

## The Solution

When a message is posted in #it-support, this workflow:

1. Filters out bot messages to prevent infinite loops
2. Checks for duplicate message IDs before processing
3. Sends the message text to GPT-4.1-mini to classify priority and draft a reply
4. Routes to the right response path based on P1 / P2 / P3
5. Replies in the original thread so the requester gets immediate acknowledgement
6. Logs every ticket to a Google Sheets tracker

---

## Priority Routing

| Priority | Signal | Actions |
|---|---|---|
| **P1** (Critical) | System-wide outage. Entire team or org cannot work. Production is down. | DM on-call engineer + post to #incidents + reply in thread |
| **P2** (Degraded) | Partial disruption. Multiple users affected. Work possible but impaired. | Post to #it-queue + reply in thread |
| **P3** (Minor) | Low urgency. Single user or small issue. Work not significantly disrupted. | Reply in thread only |

All three paths log to Sheets.

---

## Guardrails

| Guard | What it does |
|---|---|
| **Filter: Not a Bot** | Checks `$json.bot_id` is empty — stops if the message came from a bot |
| **Check for Duplicate Message** | Looks up `client_msg_id` in Google Sheets before processing |
| **Is Duplicate?** | Stops if the message ID already exists in the tracker |
| **Validate Required Fields** | Checks all 3 AI output fields (priority, summary, reply) are non-empty before routing |
| **Retry on fail** | All outbound nodes (AI, Slack, Sheets) retry automatically on transient failure |
| **Error workflow** | Separate error workflow alerts on any runtime failure |

---

## Flow Overview

```
Slack message → Filter: Not a Bot
  → Check for Duplicate Message → Is Duplicate?
  → Classify Ticket (AI: priority · summary · reply)
  → Validate Required Fields
  → Route by Priority
      P1 → DM: On-Call Engineer → Post to #incidents → Reply in Thread → Log to Sheets
      P2 → Post to #it-queue → Reply in Thread → Log to Sheets
      P3 → Reply in Thread → Log to Sheets
```

---

## Files

| File | What it is |
|---|---|
| `Slack IT Support + AI Router.json` | n8n workflow export — import directly into n8n |
| `process-map.png` | Visual flow diagram |
| `process-map.excalidraw` | Editable source for the diagram |
| `architecture.md` | Node-by-node technical breakdown |

---

## How to Import

1. Open n8n → **Workflows** → **Import from file**
2. Select `Slack IT Support + AI Router.json`
3. Set credentials: OpenRouter API, Slack OAuth2, Google Sheets OAuth2
4. Invite the Slack bot to: `#it-support`, `#incidents`, `#it-queue`
5. Update the Google Sheets document ID to your own ticket tracker sheet
6. Set the on-call engineer's Slack user ID in `DM: On-Call Engineer`
7. Activate the workflow

---

## Credentials Required

| Service | Credential type |
|---|---|
| OpenRouter (GPT-4.1-mini) | API key |
| Slack (trigger + all message nodes) | OAuth2 (bot token) |
| Google Sheets | OAuth2 |

## Slack Bot Token Scopes Required

| Scope | Why |
|---|---|
| `channels:history` | Read messages from public channels (trigger) |
| `groups:history` | Read messages from private channels |
| `chat:write` | Post messages, DMs, and thread replies |
| `chat:write.public` | Post to channels without joining them |
| `channels:read` | Look up channel IDs |
| `groups:read` | Look up private channel IDs |
| `users:read` | Look up on-call engineer's user ID for DM |
