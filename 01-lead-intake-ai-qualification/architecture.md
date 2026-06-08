# Architecture — Lead Intake + AI Qualification

## Node Map

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Webhook | Trigger | Receives POST from the contact form. Header auth blocks unauthorized requests. |
| 2 | Parse Lead Fields | Set | Extracts name, email, phone, message. Applies `($json.field \|\| '').trim()` to each — handles missing fields and whitespace-only strings. |
| 3 | Validate Required Fields | IF | Guards the flow: all 4 fields must be non-empty. Missing fields → Stop: Missing Fields. |
| 4 | Stop: Missing Fields | NoOp | Terminates silently. No action taken on incomplete submissions. |
| 5 | Check for Duplicate Email | Google Sheets (read) | Looks up the incoming email in the leads log. `alwaysOutputData: true` ensures an empty row is returned (not 0 items) when no match is found. |
| 6 | Is Duplicate? | IF | If the returned Email field is not empty → duplicate → stop. If empty → new lead → proceed. |
| 7 | Stop: Duplicate Found | NoOp | Terminates silently. Prevents double-processing if the webhook fires twice. |
| 8 | Classify Lead | Basic LLM Chain | Sends lead data to Claude. Structured Output Parser forces the response into `{ messageFormatted: { tier, reply } }`. |
| 9 | OpenRouter Chat Model | LLM sub-node | OpenRouter model node (GPT-4.1). Attached to Classify Lead. |
| 10 | Structured Output Parser | Parser sub-node | Enforces JSON schema on Claude's output — prevents free-text responses from breaking downstream nodes. |
| 11 | Route by Lead Tier | Switch | Reads `tier` from Claude's output. Routes hot → output 0, warm → output 1, cold → output 2. |
| 12 | DM owner: Hot Lead (Slack) | Slack | Sends a direct message to the owner with the lead summary and Claude's draft reply. Hot leads only. |
| 13 | Channel Summary: Warm Lead (Slack) | Slack | Posts a lead summary to the team channel. Warm leads only. |
| 14 | Auto-Reply to Lead (Gmail) | Gmail | Sends Claude's enthusiastic reply to the hot lead's email address. |
| 15 | Holding Reply to Lead (Gmail) | Gmail | Sends Claude's welcoming reply to the warm lead's email address. |
| 16 | Polite Auto-Reply to Lead (Gmail) | Gmail | Sends Claude's low-pressure reply to the cold lead. No Slack alert for cold leads. |
| 17 | Log Lead to Sheets | Google Sheets (appendOrUpdate) | Logs name, email, phone, message, tier, and Claude's reply to the CRM sheet. Runs after every successful tier path. |

---

## Guardrails

### Data IN — field validation
`Validate Required Fields` checks all 4 fields before any processing. Combined with the `.trim()` pattern in `Parse Lead Fields`, this catches empty strings, null values, and whitespace-only submissions.

### Data IN — duplicate check
`Check for Duplicate Email` queries Google Sheets by email before the AI runs. `alwaysOutputData: true` ensures the node always outputs one item — empty if no match, populated if duplicate found. The `Is Duplicate?` IF node branches on whether the Email field is empty.

### Calls OUT — retry on fail
All nodes that call external services have `retryOnFail: true`:
- Classify Lead (AI)
- DM owner: Hot Lead (Slack)
- Channel Summary: Warm Lead (Slack)
- Auto-Reply to Lead (Gmail)
- Holding Reply to Lead (Gmail)
- Polite Auto-Reply to Lead (Gmail)
- Check for Duplicate Email (Sheets read)
- Log Lead to Sheets (Sheets write)

### Whole flow — error workflow
A separate error workflow is connected in workflow settings. If any node fails at runtime, the error workflow fires and posts an alert to Slack with the workflow name, failed node, error message, and execution ID.

### Front door — webhook auth
The Webhook node requires a `headerAuth` credential. Any POST without the correct header is rejected before the flow runs. In production, the header is added server-side by a Netlify function — the secret is never exposed in the browser.

---

## Data Flow

```
Webhook payload
  └─ Parse Lead Fields        → { name, email, phone, message }
      └─ Validate Required Fields  (guard: all non-empty)
          └─ Check for Duplicate Email  (guard: email not in Sheets)
              └─ Is Duplicate?
                  └─ Classify Lead  → { output.messageFormatted.tier, .reply }
                      └─ Route by Lead Tier
                          ├─ hot  → DM Owner (Slack) → Auto-Reply (Gmail) → Log to Sheets
                          ├─ warm → Channel Alert (Slack) → Holding Reply (Gmail) → Log to Sheets
                          └─ cold → Polite Reply (Gmail) → Log to Sheets
```

---

## Credential Requirements

| Node(s) | Credential | Type |
|---|---|---|
| Webhook | Custom Website Auth | Header Auth |
| OpenRouter Chat Model | OpenRouter API | API Key |
| DM owner, Channel Summary | Slack bot | OAuth2 |
| Auto-Reply, Holding Reply, Polite Reply | Gmail | OAuth2 |
| Check for Duplicate Email, Log Lead to Sheets | Google Sheets | OAuth2 |
