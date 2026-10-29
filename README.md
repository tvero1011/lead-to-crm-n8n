# AI Lead-to-CRM Automation (n8n)

An automated lead intake pipeline: a web form lead comes in, gets validated and saved to HubSpot, is scored and answered by an AI agent, and then goes to a human for one-click approval in Slack before any email is sent.

Built with **n8n**, **OpenAI**, **HubSpot**, **Supabase** and **Slack**.

> Built with AI assistance. I designed the flow, tested it end to end, and can explain and modify every step.

---

## What it does

1. **Receives** a lead through a secured webhook (API key required).
2. **Validates and cleans** the data. Invalid leads get a `422` response and are logged.
3. **Creates or updates the contact in HubSpot**, so a repeat submission never makes a duplicate.
4. **AI agent** (GPT-4o-mini) reads the lead and returns structured output: score, priority, category, summary and a draft reply email. It uses a **RAG knowledge base** (Supabase vector store) so replies match real company information.
5. **Responds `200`** (with status, contactId, score and priority) as soon as scoring is done, so the sender never waits for the human approval.
6. **Routes by priority**
   - **High:** posts to Slack with the summary and draft email, with **Approve and Send** and **Reject** buttons.
     - Approve: sends the email and creates a HubSpot follow-up task.
     - Reject: logs the lead as rejected by a human.
   - **Other priorities:** stored only, no email.
7. **Logs every run** to a Supabase `lead_runs` table (email, company, status, execution id).

## Flow

```
Webhook (API key)
  -> Validate and Clean -> valid?
       no  -> Respond 422 -> Log Rejected
       yes -> HubSpot Upsert Contact (sub-workflow)
           -> AI Agent: score + draft email (OpenAI + Supabase RAG + structured output)
           -> Prepare Data -> Respond 200
           -> Route by Priority
                high  -> Slack Approval -> Approved?
                           yes -> Send Email + HubSpot follow-up task (sub-workflow) -> Log Run
                           no  -> Status: rejected by human -> Log Run
                other -> Status: stored only -> Log Run
```

Full canvas: [`docs/screenshots/10-canvas-full.png`](docs/screenshots/10-canvas-full.png)

Extra pieces on the canvas:
- **RAG ingestion** (run once): loads company documents into the Supabase vector store.
- **Error handler:** logs failures to Supabase and alerts Slack.
- **Duplicate check:** a scheduled job that scans HubSpot for duplicate emails.

## Tests

| # | Scenario | Result | Evidence |
|---|----------|--------|----------|
| 1 | Valid lead | `200`, `contactId`, score 85, priority high; Slack approval and HubSpot contact created | `05`, `06`, `07` |
| 2 | Same lead sent twice | Same `contactId` returned, no duplicate in HubSpot | `08`, `09` |
| 3 | Missing email | `422` "A valid email is required", logged as `rejected_invalid` | `02`, `03` |
| 4a | Wrong API key | `403 Forbidden` | `01` |
| 4b | No API key | `403 Forbidden` | `04` |
| 5 | Approve in Slack | Email received, run logged as `email_sent` | `11`, `12` |
| 6 | Reject in Slack | Run logged as `approval_rejected` | `12` |

Screenshots are in [`docs/screenshots/`](docs/screenshots/).

## Design decisions

- **Human in the loop:** the AI drafts, but a person approves before anything is sent to a customer.
- **Idempotent CRM write:** HubSpot upsert by email, so retries and double submissions are safe.
- **Fast webhook response:** the `200` is returned before the Slack approval, so the sender never times out.
- **Fail loudly:** invalid or unauthorized requests get clear HTTP codes, and every outcome is logged.
- **Modular:** the HubSpot and email steps are sub-workflows, so they can be reused or tested alone.

## Setup

1. Import the files in `workflows/` into n8n (import the sub-workflows first).
2. Create credentials in n8n: Header Auth (webhook), OpenAI, Supabase, HubSpot, Slack.
3. Supabase: create a `lead_runs` table (`created_at`, `email`, `company`, `status`, `execution_id`) and a `documents` table for the vector store.
4. Run the RAG ingestion branch once to load your company documents.
5. Set your Slack channel in the Slack nodes and activate the workflow.

## Try it

```bash
curl -X POST "$WEBHOOK_URL" \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Maria Santos",
    "email": "maria@acme-demo.com",
    "company": "Acme Demo",
    "message": "We want to integrate our web form leads into HubSpot this month."
  }'
```

Expected response:

```json
{"status":"received","contactId":"...","score":85,"priority":"high"}
```

## Tech stack

n8n - OpenAI (GPT-4o-mini) - Supabase (Postgres + pgvector) - HubSpot CRM - Slack

## Possible improvements

- Retry and backoff on API failures
- Rate limiting on the webhook
- Support for more lead sources (Typeform, Google Forms)
- A dashboard on top of the `lead_runs` table

## License

MIT
