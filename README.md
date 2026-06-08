# AI Automation Portfolio — Justine Jurel Valenzuela

Production-grade n8n workflows enhanced with Claude AI. Each workflow is built as a reusable pattern: swap the branding and copy, keep the logic.

---

## Workflows

| # | Workflow | Pattern | Stack |
|---|---|---|---|
| [01](01-lead-intake-ai-qualification/) | Lead Intake + AI Qualification | Lead-to-text | n8n · OpenRouter · Slack · Gmail · Google Sheets |
| 02 | AI-Powered Hiring Pipeline | Candidate screening | n8n · Claude · Gmail |
| 03 | Slack IT Support + AI Router | Ticket triage | n8n · Claude · Slack |
| 04 | AI Agent: Employee Onboarding | Agentic loop | n8n · Claude AI Agent · Sub-workflows |

---

## What Makes These Production-Grade

Every workflow includes:
- **Input validation** — guards against empty, null, and whitespace-only fields
- **Duplicate protection** — prevents double-processing on webhook retries
- **Retry on fail** — all outbound calls (AI, Slack, Gmail, Sheets) retry automatically
- **Error workflow** — separate monitoring workflow alerts on any runtime failure
- **Webhook auth** — header secret blocks unauthorized requests

---

## How to Use

Each workflow folder contains:
- `*.json` — import directly into n8n (Workflows → Import from file)
- `README.md` — problem, solution, tier routing, credentials needed
- `architecture.md` — node-by-node breakdown with data flow
- `process-map.png` — visual flow diagram

---

## Contact

Built by Justine Jurel Valenzuela — AI automation specialist.
Available for freelance and contract work.
