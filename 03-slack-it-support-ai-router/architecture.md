# Architecture — Slack IT Support + AI Router

## Node Map

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Slack Trigger | Trigger | Polls #it-support for new messages. Each new message starts one execution. |
| 2 | Filter: Not a Bot | IF | Checks `$json.bot_id` is empty. Stops bot messages before they enter the flow — prevents the workflow from triggering on its own replies. |
| 3 | Check for Duplicate Message | Google Sheets (read) | Looks up `client_msg_id` in the ticket tracker. `alwaysOutputData: true` returns an empty row when no match is found. |
| 4 | Is Duplicate? | IF | If the returned `messageId` field is not empty → duplicate → stop. If empty → new message → proceed. |
| 5 | Stop: Duplicate | NoOp | Terminates silently if message was already processed. |
| 6 | Classify Ticket | Basic LLM Chain | Sends message text to GPT-4.1-mini. Extracts priority (p1/p2/p3), writes a one-line summary, and drafts a thread reply in one step. |
| 7 | OpenRouter: GPT-4.1-mini | LLM sub-node | GPT-4.1-mini via OpenRouter. Attached to Classify Ticket. Cost-optimised for high-volume ticket triage. |
| 8 | Structured Output Parser | Parser sub-node | Enforces JSON schema on AI output — prevents free-text responses from breaking downstream nodes. Schema: `{ priority, summary, reply }`. |
| 9 | Validate Required Fields | IF | Guards the flow: all 3 AI output fields must be non-empty. Incomplete output → Stop: Missing Fields. |
| 10 | Stop: Missing Fields | NoOp | Terminates silently if AI output is incomplete. |
| 11 | Route by Priority | Switch | Reads `priority` from AI output. Routes p1 → output 0, p2 → output 1, p3 → output 2. |
| 12 | DM: On-Call Engineer | Slack | Sends a DM to the on-call engineer with priority, summary, reporter, and original message. P1 only. |
| 13 | Post to #incidents | Slack | Posts a P1 incident alert to #incidents with full ticket context. P1 only. |
| 14 | Post to #it-queue channel | Slack | Posts a P2 ticket notice to #it-queue for the IT team to pick up. P2 only. |
| 15 | Reply in Thread | Slack | Posts the AI-drafted reply back to the original #it-support thread. All three priority paths. |
| 16 | Log Ticket to Sheets | Google Sheets (appendOrUpdate) | Logs all 7 fields to the ticket tracker. Matches on `messageId` to prevent duplicate rows. |

---

## Guardrails

### Bot filter
`Filter: Not a Bot` checks `$json.bot_id` before anything else. The Slack Trigger fires on every message — including messages the bot itself posts. Without this guard, the workflow triggers on its own thread replies, creating an infinite loop.

### Duplicate check
`Check for Duplicate Message` queries Google Sheets by `client_msg_id` before calling the AI. `alwaysOutputData: true` ensures the node always outputs one item — empty if no match, populated if already seen. The `Is Duplicate?` IF branches on whether `$json.messageId` is empty.

### AI output validation
`Validate Required Fields` checks all 3 output fields from `Classify Ticket` before routing. Catches cases where the AI returns an incomplete JSON.

### Calls OUT — retry on fail
All nodes that call external services have `retryOnFail: true`:
- Classify Ticket (AI)
- Check for Duplicate Message (Sheets read)
- DM: On-Call Engineer (Slack)
- Post to #incidents (Slack)
- Post to #it-queue channel (Slack)
- Reply in Thread (Slack)
- Log Ticket to Sheets (Sheets write)

### Whole flow — error workflow
A separate error workflow is connected in workflow settings. If any node fails at runtime, the error workflow fires an alert with the workflow name, failed node, error message, and execution ID.

---

## Data Flow

```
Slack message received ($json.text, $json.user, $json.channel, $json.ts, $json.client_msg_id)
  └─ Filter: Not a Bot       (guard: bot_id must be empty)
      └─ Check for Duplicate  (guard: client_msg_id not already in Sheets)
          └─ Is Duplicate?
              └─ Classify Ticket  → { output: { priority, summary, reply } }
                  └─ Validate Required Fields  (guard: all 3 fields non-empty)
                      └─ Route by Priority
                          ├─ p1 → DM: On-Call Engineer → Post to #incidents → Reply in Thread → Log to Sheets
                          ├─ p2 → Post to #it-queue → Reply in Thread → Log to Sheets
                          └─ p3 → Reply in Thread → Log to Sheets
```

---

## Credential Requirements

| Node(s) | Credential | Type |
|---|---|---|
| Slack Trigger, all Slack message nodes | Slack bot | OAuth2 |
| OpenRouter: GPT-4.1-mini | OpenRouter API | API Key |
| Check for Duplicate Message, Log Ticket to Sheets | Google Sheets | OAuth2 |

---

## Key Differences from WF01 and WF02

| | WF01 — Lead Intake | WF02 — Hiring Pipeline | WF03 — IT Support |
|---|---|---|---|
| Trigger | Webhook (form POST) | Gmail Trigger (email polling) | Slack Trigger (channel message) |
| AI task | Classify lead quality | Extract fields + score candidate | Classify ticket priority |
| Output schema | tier, reply | name, email, position, coverLetter, score, tier, reply | priority, summary, reply |
| Duplicate key | Email address | Email address | Slack `client_msg_id` |
| Extra guard | — | — | Bot filter (prevents loop on own replies) |
| Alert channel | Slack DM | Slack DM | Slack DM + #incidents channel |
| Model | GPT-4.1 | GPT-4.1-mini | GPT-4.1-mini |
