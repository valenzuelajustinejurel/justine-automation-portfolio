# Architecture — AI-Powered Hiring Pipeline

## Node Map

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Gmail Trigger | Trigger | Polls inbox every minute for new emails. Each new email starts one execution. |
| 2 | Classify Candidate | Basic LLM Chain | Sends subject + sender + email body to GPT-4.1-mini. Extracts name, email, position, cover letter, score (1–10), tier, and reply in one step. |
| 3 | OpenRouter Chat Model | LLM sub-node | GPT-4.1-mini via OpenRouter. Attached to Classify Candidate. Cost-optimised for high-volume screening. |
| 4 | Structured Output Parser | Parser sub-node | Enforces JSON schema on the AI output — prevents free-text responses from breaking downstream nodes. Schema: `{ name, email, position, coverLetter, score, tier, reply }`. |
| 5 | Validate Required Fields | IF | Guards the flow: all 6 AI output fields must be non-empty. Incomplete AI output → Stop: Missing Fields. |
| 6 | Stop: Missing Fields | NoOp | Terminates silently if AI output is incomplete. |
| 7 | Check for Duplicate Email | Google Sheets (read) | Looks up the applicant's extracted email in the candidate tracker. `alwaysOutputData: true` ensures an empty row is returned when no match is found. |
| 8 | Is Duplicate? | IF | If the returned email field is not empty → duplicate → stop. If empty → new applicant → proceed. |
| 9 | Stop: Duplicate | NoOp | Terminates silently. Prevents double-processing if the same email arrives twice. |
| 10 | Route by Candidate Tier | Switch | Reads `tier` from AI output. Routes strong → output 0, maybe → output 1, pass → output 2. |
| 11 | DM Hiring Manager: Strong Candidate (Slack) | Slack | Sends a DM to the hiring manager with name, score, cover letter summary, and suggested reply. Strong candidates only. |
| 12 | Strong Reply to Candidate (Gmail) | Gmail | Sends the AI-drafted enthusiastic reply to the strong candidate's email address. |
| 13 | Holding Reply to Candidate (Gmail) | Gmail | Sends a warm holding reply to the maybe candidate. No Slack alert. |
| 14 | Rejection Reply to Candidate (Gmail) | Gmail | Sends a polite rejection to the pass candidate. No Slack alert. |
| 15 | Log Candidate to Sheets | Google Sheets (appendOrUpdate) | Logs all 7 fields (name, email, position, coverLetter, score, tier, reply) to the candidate tracker. Runs after every successful tier path. Matches on email. |

---

## Guardrails

### AI output validation
`Validate Required Fields` checks all 6 output fields from `Classify Candidate` before routing. Catches cases where the AI returns an incomplete JSON (missing name, score, etc.).

### Duplicate check
`Check for Duplicate Email` queries Google Sheets by the AI-extracted email before any notifications fire. `alwaysOutputData: true` ensures the node always outputs one item — empty if no match, populated if already seen. The `Is Duplicate?` IF branches on whether the email field is empty.

### Calls OUT — retry on fail
All nodes that call external services have `retryOnFail: true`:
- Classify Candidate (AI)
- Check for Duplicate Email (Sheets read)
- DM Hiring Manager: Strong Candidate (Slack)
- Strong Reply to Candidate (Gmail)
- Holding Reply to Candidate (Gmail)
- Rejection Reply to Candidate (Gmail)
- Log Candidate to Sheets (Sheets write)

### Whole flow — error workflow
A separate error workflow is connected in workflow settings. If any node fails at runtime, the error workflow fires an alert with the workflow name, failed node, error message, and execution ID.

---

## Data Flow

```
Gmail email received
  └─ Classify Candidate   → { output: { name, email, position, coverLetter, score, tier, reply } }
      └─ Validate Required Fields  (guard: all 6 fields non-empty)
          └─ Check for Duplicate Email  (guard: email not already in Sheets)
              └─ Is Duplicate?
                  └─ Route by Candidate Tier
                      ├─ strong → DM Hiring Manager (Slack) → Strong Reply (Gmail) → Log to Sheets
                      ├─ maybe  → Holding Reply (Gmail) → Log to Sheets
                      └─ pass   → Rejection Reply (Gmail) → Log to Sheets
```

---

## Credential Requirements

| Node(s) | Credential | Type |
|---|---|---|
| Gmail Trigger, Strong/Holding/Rejection Reply | Gmail | OAuth2 |
| OpenRouter Chat Model | OpenRouter API | API Key |
| DM Hiring Manager | Slack bot | OAuth2 |
| Check for Duplicate Email, Log Candidate to Sheets | Google Sheets | OAuth2 |

---

## Key Differences from WF01

| | WF01 — Lead Intake | WF02 — Hiring Pipeline |
|---|---|---|
| Trigger | Webhook (form POST) | Gmail Trigger (email polling) |
| AI task | Classify lead quality | Extract fields + score candidate |
| Output schema | tier, reply | name, email, position, coverLetter, score, tier, reply |
| Score | Qualitative (hot/warm/cold) | Numeric 1–10 + qualitative tier |
| Strong alert | Slack DM | Slack DM with score + cover letter |
| Model | GPT-4.1 | GPT-4.1-mini (cost-optimised for volume) |
