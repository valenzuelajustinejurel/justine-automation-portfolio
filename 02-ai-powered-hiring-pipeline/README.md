# Workflow 02 — AI-Powered Hiring Pipeline

**Client:** NovaCo (fictional company)
**Stack:** n8n · OpenRouter (GPT-4.1-mini) · Slack · Gmail · Google Sheets
**Pattern:** AI hiring screen — email application → AI scores candidate → notify hiring manager → reply to applicant

---

## The Problem

NovaCo receives job applications via email. Without automation, every application sits in an inbox until someone reads it, manually decides how strong the candidate is, and writes a reply. Strong candidates wait too long and lose interest. Weak candidates get the same attention as strong ones.

## The Solution

When a candidate emails their application, this workflow:

1. Reads the email (subject, sender, body) and sends it to GPT-4.1-mini via OpenRouter
2. The AI extracts the candidate's details, scores them 1–10, and drafts a tier-appropriate reply
3. Routes the candidate to the right response path based on their score
4. Logs every application to a Google Sheets candidate tracker

---

## Tier Routing

| Tier | Score | Signal | Hiring Manager Alert | Candidate Reply |
|---|---|---|---|---|
| **Strong** | 7–10 | Specific experience, measurable results, clear fit | Slack DM with score + cover letter summary | Enthusiastic — invite to schedule a call |
| **Maybe** | 4–6 | Some relevant experience, vague, no clear results | None | Warm — we'll be in touch |
| **Pass** | 1–3 | No relevant experience, very generic | None | Polite rejection |

---

## Guardrails

| Guard | What it does |
|---|---|
| **AI output validation** | Checks all 6 AI output fields (name, email, position, score, tier, reply) are non-empty before routing |
| **Duplicate check** | Looks up the applicant's email in Google Sheets before processing — stops if already seen |
| **Retry on fail** | All outbound nodes (AI, Slack, Gmail, Sheets) retry automatically on transient failure |
| **Error workflow** | Separate error workflow alerts on any runtime failure |

---

## Flow Overview

```
New email → Gmail Trigger
  → AI extracts + scores (strong / maybe / pass)
  → [validate fields] → [check duplicate]
  → strong: DM hiring manager (Slack) → reply to candidate (Gmail) → log to Sheets
  → maybe:  holding reply (Gmail) → log to Sheets
  → pass:   rejection reply (Gmail) → log to Sheets
```

---

## Files

| File | What it is |
|---|---|
| `AI-Powered Hiring Pipeline.json` | n8n workflow export — import directly into n8n |
| `process-map.png` | Visual flow diagram |
| `process-map.excalidraw` | Editable source for the diagram |
| `architecture.md` | Node-by-node technical breakdown |

---

## How to Import

1. Open n8n → **Workflows** → **Import from file**
2. Select `AI-Powered Hiring Pipeline.json`
3. Set credentials: OpenRouter API, Slack, Gmail OAuth2, Google Sheets OAuth2
4. Set the Gmail Trigger to watch the inbox where applications are received
5. Update the Google Sheets document ID to your own candidate tracker sheet
6. Activate the workflow

---

## Credentials Required

| Service | Credential type |
|---|---|
| OpenRouter (GPT-4.1-mini) | API key |
| Slack | OAuth2 (bot token) |
| Gmail (trigger + send) | OAuth2 |
| Google Sheets | OAuth2 |

---

## Note on Email Body

This workflow reads `$json.snippet` from the Gmail Trigger — a short preview of the email body. For short application emails this is sufficient. For longer emails, disable **Simplify** in the Gmail Trigger node settings to access the full body via `$json.text`.
