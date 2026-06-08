# Workflow 01 — Lead Intake + AI Qualification

**Client:** Horizon Consulting (fictional service business)
**Stack:** n8n · OpenRouter (GPT-4.1) · Slack · Gmail · Google Sheets
**Pattern:** Lead-to-text — inbound form → AI qualify → notify owner → reply to lead

---

## The Problem

Horizon Consulting receives inbound leads through their website contact form. Without automation, every lead — hot or cold — sits in an inbox waiting for a human to read it, decide how urgent it is, and write a reply. Hot leads lose interest waiting. Cold leads get the same attention as hot ones.

## The Solution

When a lead submits the contact form, this workflow:

1. Validates the submission (no empty fields, no duplicates)
2. Sends the lead data to GPT-4.1 via OpenRouter
3. The AI classifies the lead as **hot**, **warm**, or **cold** and drafts a personalized first reply
4. Routes the lead to the right response path based on tier
5. Logs every lead to a Google Sheets CRM

---

## Tier Routing

| Tier | Signal | Owner Alert | Lead Reply |
|---|---|---|---|
| **Hot** | Urgent language, budget mentioned, ready to start | Slack DM to owner | Enthusiastic reply, book a call |
| **Warm** | Interested, asking questions, no timeline | Slack channel summary | Welcoming reply, offer more info |
| **Cold** | Vague, just browsing | None | Polite, low-pressure reply |

---

## Guardrails

| Guard | What it does |
|---|---|
| **Field validation** | Rejects submissions with any empty or whitespace-only field before touching the AI |
| **Duplicate check** | Looks up the email in Google Sheets before processing — stops if already seen |
| **Webhook auth** | Header auth credential on the webhook — only authorized sources can trigger the flow |
| **Retry on fail** | All outbound nodes (AI, Slack, Gmail, Sheets) retry automatically on transient failure |
| **Error workflow** | Separate error workflow pings Slack if anything breaks at runtime |

---

## Flow Overview

```
Form POST → [validate fields] → [check duplicate]
  → AI classifies (hot / warm / cold)
  → hot:  DM owner (Slack) → reply to lead (Gmail) → log to Sheets
  → warm: channel alert (Slack) → holding reply (Gmail) → log to Sheets
  → cold: polite reply (Gmail) → log to Sheets
```

---

## Files

| File | What it is |
|---|---|
| `Lead Intake + AI Qualification.json` | n8n workflow export — import directly into n8n |
| `process-map.png` | Visual flow diagram |
| `process-map.excalidraw` | Editable source for the diagram |
| `architecture.md` | Node-by-node technical breakdown |

---

## How to Import

1. Open n8n → **Workflows** → **Import from file**
2. Select `Lead Intake + AI Qualification.json`
3. Set credentials: OpenRouter API, Slack, Gmail OAuth2, Google Sheets OAuth2
4. Set webhook header auth credential (`x-webhook-secret`)
5. Update the Google Sheets document ID to your own sheet
6. Activate the workflow

---

## Credentials Required

| Service | Credential type |
|---|---|
| OpenRouter (GPT-4.1) | API key |
| Slack | OAuth2 (bot token) |
| Gmail | OAuth2 |
| Google Sheets | OAuth2 |
| Webhook | Header auth (secret key) |
