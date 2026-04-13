# Recruitment Pipeline — Setup Guide

Complete setup guide for importing and activating the 7-workflow recruitment automation pipeline on n8n Cloud.

---

## Prerequisites

- n8n Cloud account (cloud.n8n.io) or self-hosted n8n v1.x
- Access to all integrated services: Typeform, Anthropic API, HubSpot, Slack, Gmail, ClickUp, Google Sheets, Supabase (Postgres)
- Admin/owner access to your n8n workspace (required to create credentials)

---

## Step 1 — Create Credentials in n8n

Create all credentials **before** importing any workflows. Credentials must exist for workflow imports to link correctly.

Navigate to: **n8n → Settings → Credentials → + Add credential**

### 1.1 Slack (Bot Token)
- Type: **Slack API**
- Token: Your Slack Bot User OAuth Token (`xoxb-...`)
- Required bot scopes: `chat:write`, `channels:read`, `users:read`, `chat:write.public`
- Create the Slack app at: api.slack.com/apps → Create New App → From scratch → OAuth & Permissions → add scopes → Install to Workspace → copy Bot User OAuth Token
- Note the n8n credential ID — use it for `[SLACK_CREDENTIAL_ID]`

### 1.2 Gmail (OAuth2)
- Type: **Gmail OAuth2**
- Complete the Google OAuth flow — authenticate with the Gmail account that will send candidate emails
- Note the n8n credential ID — use it for `[GMAIL_CREDENTIAL_ID]`

### 1.3 Google Sheets (OAuth2)
- Type: **Google Sheets OAuth2**
- Can use the same Google account as Gmail or a different one
- Note the n8n credential ID — use it for `[GOOGLE_SHEETS_CREDENTIAL_ID]`

### 1.4 Anthropic API (Header Auth)
- Type: **HTTP Header Auth**
- Name field: `x-api-key`
- Value field: your Anthropic API key (`sk-ant-...`)
- Get key at: console.anthropic.com → API Keys
- Note the n8n credential ID — use it for `[ANTHROPIC_CREDENTIAL_ID]`

### 1.5 HubSpot (Private App Token)
- Create a HubSpot Private App first:
  - HubSpot → Settings → Integrations → Private Apps → Create a private app
  - App name: "Recruitment Pipeline"
  - Scopes required: `crm.objects.contacts.read`, `crm.objects.contacts.write`, `crm.objects.deals.read`, `crm.objects.deals.write`
  - Create app → copy the Access Token
- In n8n: Type: **HubSpot API** → paste Access Token
- Note the n8n credential ID — use it for `[HUBSPOT_CREDENTIAL_ID]`

### 1.6 Typeform (OAuth2)
- Type: **Typeform OAuth2 API**
- Complete the Typeform OAuth flow
- Note the n8n credential ID — use it for `[TYPEFORM_CREDENTIAL_ID]`

### 1.7 ClickUp (API Token)
- Type: **ClickUp API**
- Find your token: ClickUp → Profile (bottom-left) → Apps → API Token
- Note the n8n credential ID — use it for `[CLICKUP_CREDENTIAL_ID]`

### 1.8 Postgres / Supabase
- Type: **Postgres**
- Connection details from Supabase: Project → Settings → Database → Connection string
- Host: `db.[PROJECT_REF].supabase.co`
- Port: `5432`
- Database: `postgres`
- User: `postgres`
- Password: your database password
- SSL: enabled
- Note the n8n credential ID — use it for `[POSTGRES_CREDENTIAL_ID]`

---

## Step 2 — Set Up External Services

### 2.1 HubSpot — Create Custom Properties

Before activating, create these custom contact properties in HubSpot:

Navigate to: **HubSpot → Settings → Properties → Contact properties → Create property**

| Property name | Type | Group |
|---|---|---|
| `ai_composite_score` | Number | Contact Information |
| `ai_disposition` | Single-line text | Contact Information |
| `ai_role_applied` | Single-line text | Contact Information |
| `ai_scored_at` | Date / time | Contact Information |

### 2.2 Slack — Create Ops Channel

Create a Slack channel for recruitment ops alerts (e.g. `#recruitment-ops` or `#ops-alerts`).

Find channel ID: Slack → right-click channel name → View channel details → scroll to bottom → Copy channel ID. Looks like `C0123456789`.

Use this for `[SLACK_OPS_CHANNEL_ID]`.

Invite your Slack bot to this channel: type `/invite @your-bot-name` in the channel.

### 2.3 Google Sheets — Create Audit Log Sheet

Create a Google Spreadsheet for the candidate audit log.

In Sheet Tab 1 (name it e.g. `Candidates`), add these headers in row 1:

```
candidate_name | candidate_email | client_name | role_applied | composite_score | disposition | overall_signal | role_fit_score | experience_score | response_qual_score | hubspot_contact_id | hubspot_deal_id | slack_notified | gmail_sent | email_type | clickup_task_created | scored_at | comms_completed_at
```

In Sheet Tab 2 (name it e.g. `Offers`), add these headers in row 1:

```
candidate_name | candidate_email | role_applied | client_name | offer_details | decision | offered_at | logged_at
```

Both tabs can be in the same spreadsheet (use the same `[AUDIT_SPREADSHEET_ID]` with different sheet names) or separate spreadsheets.

### 2.4 Supabase — Create Client Config Tables

Run this SQL in your Supabase SQL editor to create the three config tables:

```sql
-- Table 1: Clients
CREATE TABLE clients (
  id              SERIAL PRIMARY KEY,
  name            TEXT NOT NULL,
  slug            TEXT NOT NULL UNIQUE,
  score_threshold INTEGER NOT NULL DEFAULT 60,
  default_role    TEXT,
  active          BOOLEAN NOT NULL DEFAULT true,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Table 2: Scoring prompts (one active prompt per client)
CREATE TABLE scoring_prompts (
  id              SERIAL PRIMARY KEY,
  client_id       INTEGER NOT NULL REFERENCES clients(id),
  prompt_template TEXT,
  active          BOOLEAN NOT NULL DEFAULT true
);

-- Table 3: Email templates (one per type per client: 'success' and 'rejection')
CREATE TABLE email_templates (
  id              SERIAL PRIMARY KEY,
  client_id       INTEGER NOT NULL REFERENCES clients(id),
  template_type   TEXT NOT NULL CHECK (template_type IN ('success', 'rejection')),
  subject         TEXT,
  body            TEXT
);

-- Sample client (replace with your actual client)
INSERT INTO clients (name, slug, score_threshold) VALUES ('Acme Corp', 'acme-corp', 65);

-- Add a default scoring prompt for the sample client (NULL = use built-in rubric)
INSERT INTO scoring_prompts (client_id, prompt_template, active)
  SELECT id, NULL, true FROM clients WHERE slug = 'acme-corp';
```

The `slug` must match the `client_slug` hidden field value your Typeform sends. The pipeline falls back to sensible defaults if `prompt_template`, email subjects, or bodies are null.

### 2.5 Typeform — Configure Your Form

Your Typeform form must collect these fields (field reference names must match exactly):

| Field | Typeform reference |
|---|---|
| Full name | `full_name` |
| Email | `email` |
| Role applied for | `role` |
| Years of experience | `experience` |
| Cover note / message | `cover_note` |
| Client slug (hidden field) | `client_slug` |

**These reference names must match exactly** — the intake workflow reads `fieldMap['role']`, `fieldMap['experience']`, etc. Using `role_applied` or `years_experience` will produce empty values.

To add a hidden field: Typeform form editor → Logic → Hidden fields → add `client_slug`.

Set the hidden field value in your Typeform embed URL: `?client_slug=acme-corp`

#### Typeform Webhook Setup
1. Typeform Admin → Connect → Webhooks → + Add a webhook
2. Webhook URL: copy the production webhook URL from n8n after activating `01-candidate-intake`
3. Enable the webhook → note the **Signing Secret**
4. In n8n: **Settings → Variables → + Add Variable** → Name: `TYPEFORM_HMAC_SECRET`, Value: your signing secret

### 2.6 ClickUp — Find Your IDs

| ID | Where to find it |
|---|---|
| `[CLICKUP_TEAM_ID]` | ClickUp → Settings → Workspace Settings → Workspace ID |
| `[CLICKUP_SPACE_ID]` | Navigate to Space → URL: `...clickup.com/[TEAM_ID]/v/l/[SPACE_ID]` |
| `[CLICKUP_FOLDER_ID]` | Navigate to Folder → URL contains `/folder/[FOLDER_ID]` |
| `[CLICKUP_LIST_ID]` | Navigate to List → URL contains `/list/[LIST_ID]` |

If your list is not inside a folder, use `no_folder` for `[CLICKUP_FOLDER_ID]` and leave the folder field empty in the node.

---

## Step 3 — Import Workflows

Download all 7 JSON files from the `workflows/` directory.

In n8n: **Workflows → Import from file** (top-right menu).

Import in this order (sub-workflows must exist before parent workflows reference them):

1. `00-error-handler.json`
2. `05-audit-log.json`
3. `04-comms-routing.json`
4. `03-crm-routing.json`
5. `02-ai-scoring.json`
6. `01-candidate-intake.json`
7. `06-offer-gate.json`

After import, n8n assigns each workflow a new ID. The sub-workflow `Execute Workflow` nodes reference specific IDs — you will need to update these to your newly assigned IDs (see Step 4).

---

## Step 4 — Update Workflow ID References

After import, each workflow gets a new n8n ID (visible in the URL: `.../workflow/[ID]`).

Open each workflow below and update the `Execute Workflow` node to point to the correct new ID:

| Workflow to edit | Node to update | Points to |
|---|---|---|
| `01-candidate-intake` | `Execute Workflow — AI Scoring` | new ID of `02-ai-scoring` |
| `02-ai-scoring` | `Execute Workflow — CRM Routing` | new ID of `03-crm-routing` |
| `03-crm-routing` | `Execute Workflow — Comms Routing` | new ID of `04-comms-routing` |
| `04-comms-routing` | `Execute Workflow — Audit Log` | new ID of `05-audit-log` |

In each node: click the workflow ID field → paste the new ID.

Also update `errorWorkflow` in each workflow's Settings tab to the new ID of `00-error-handler`.

---

## Step 5 — Set Environment Variables for HMAC Secrets

Both HMAC secrets are read from n8n environment variables — **do not edit Code node jsCode directly**.

In n8n: **Settings → Variables → + Add Variable** for each:

| Variable name | Value | Used in |
|---|---|---|
| `TYPEFORM_HMAC_SECRET` | Your Typeform webhook signing secret | `01-candidate-intake` → Verify Typeform HMAC |
| `OFFER_WEBHOOK_SECRET` | Your generated offer gate secret (`openssl rand -hex 32`) | `06-offer-gate` → Verify HMAC Signature |

The HubSpot Deal Stage ID is set in the `stage` expression of the `HubSpot — Deal Create` node in `03-crm-routing`. Replace `[HUBSPOT_DEAL_STAGE_ID]` with your pipeline stage ID.

---

## Step 6 — Link Credentials in Each Workflow

Open each workflow → click each external API node → select the credential from the dropdown.

Quick reference:

| Workflow | Node | Credential |
|---|---|---|
| 01 | Postgres — Load Client Config | `[POSTGRES_CREDENTIAL_ID]` |
| 02 | HTTP Request — Claude API | `[ANTHROPIC_CREDENTIAL_ID]` |
| 03 | HubSpot — Contact Upsert | `[HUBSPOT_CREDENTIAL_ID]` |
| 03 | HubSpot — Deal Create | `[HUBSPOT_CREDENTIAL_ID]` |
| 04 | Slack — Ops Alert | `[SLACK_CREDENTIAL_ID]` |
| 04 | Gmail — Application Accepted | `[GMAIL_CREDENTIAL_ID]` |
| 04 | Gmail — Application Rejected | `[GMAIL_CREDENTIAL_ID]` |
| 04 | ClickUp — Follow-Up Task | `[CLICKUP_CREDENTIAL_ID]` |
| 05 | Google Sheets — Append Audit Row | `[GOOGLE_SHEETS_CREDENTIAL_ID]` |
| 06 | Slack — Approval Request | `[SLACK_CREDENTIAL_ID]` |
| 06 | Gmail — Send Offer Email | `[GMAIL_CREDENTIAL_ID]` |
| 06 | Gmail — Send Withdrawal Email | `[GMAIL_CREDENTIAL_ID]` |
| 06 | Google Sheets — Log Offer Sent | `[GOOGLE_SHEETS_CREDENTIAL_ID]` |
| 06 | Google Sheets — Log Offer Withdrawn | `[GOOGLE_SHEETS_CREDENTIAL_ID]` |

---

## Step 7 — Activate Workflows

Activate in this exact order (reverse dependency order):

```
1. 00-error-handler        ← must be active first
2. 05-audit-log            ← called by 04-comms-routing
3. 04-comms-routing        ← called by 03-crm-routing
4. 03-crm-routing          ← called by 02-ai-scoring
5. 02-ai-scoring           ← called by 01-candidate-intake
6. 01-candidate-intake     ← pipeline entry point, activate last
7. 06-offer-gate           ← standalone, activate independently
```

For each workflow: open → toggle Active (top-right) → confirm.

After activating `01-candidate-intake`, copy the webhook URL from the Typeform Trigger node and paste it into your Typeform webhook settings (Step 2.5).

After activating `06-offer-gate`, copy the webhook URL from the `Offer Gate Webhook` node. This is the URL your recruiter dispatch system POSTs to when initiating an offer.

---

## Step 8 — Test the Pipeline

### Test the main pipeline (Typeform → Audit Log)

1. Submit a test entry through your Typeform
2. Watch `01-candidate-intake` execution in n8n (Executions tab)
3. Each sub-workflow should fire in sequence: 01 → 02 → 03 → 04 → 05
4. Verify:
   - HubSpot: new contact created with `ai_composite_score` property
   - HubSpot: new deal linked to contact
   - Slack `#recruitment-ops`: alert card received
   - Gmail (sending account): acceptance or rejection email sent
   - ClickUp: follow-up task created (if score ≥ threshold)
   - Google Sheets audit log: row appended with all 18 fields

### Test the offer gate

Send a test POST request to the offer gate webhook URL:

```bash
# Generate signature (replace SECRET with your OFFER_WEBHOOK_SECRET env var value)
# requestedAt is required — the workflow rejects payloads older than 5 minutes
BODY="{\"candidateName\":\"Jane Smith\",\"candidateEmail\":\"jane@example.com\",\"roleApplied\":\"Senior Engineer\",\"clientName\":\"Acme Corp\",\"offerDetails\":\"Base: \$120k | Start: May 1 | Full benefits\",\"requestedAt\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}"
SIG="sha256=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "SECRET" | awk '{print $2}')"

curl -X POST https://your-n8n-instance.app.n8n.cloud/webhook/offer-gate \
  -H "Content-Type: application/json" \
  -H "x-offer-signature: $SIG" \
  -d "$BODY"
```

Then in Slack: click "Approve Offer" or "Reject Offer" on the message that arrives.

Verify:
- Approval → offer email sent to `jane@example.com`, `OFFER_SENT` row in Google Sheets
- Rejection → withdrawal email sent, `OFFER_WITHDRAWN` row in Google Sheets

---

## Security Notes

- **Execution data retention:** Failed executions log candidate PII (name, email) for debugging (`saveDataErrorExecution: "all"`). Review n8n's data retention settings to limit exposure window.
- **Offer gate secret:** `OFFER_WEBHOOK_SECRET` (n8n env var) is the sole auth gate on the offer webhook. Rotate it immediately if compromised — update the env var and notify your recruiter dispatch system.
- **Typeform HMAC:** The Typeform signing secret validates that submissions come from your form, not arbitrary POST requests.
- **Sub-workflow caller policy:** All sub-workflows have `callerPolicy: "workflowsFromSameOwner"` — they reject calls from workflows owned by other accounts.
- **API keys:** Never stored in workflow JSON files. All credentials are in n8n's encrypted credential store.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| 01-candidate-intake fails immediately | HMAC verification failed | Check `TYPEFORM_HMAC_SECRET` env var in n8n Settings → Variables matches the Typeform signing secret |
| No Postgres data found | Wrong client_slug in Typeform hidden field | Verify `client_slug` hidden field value matches a `slug` in the `clients` table |
| HubSpot node fails with 403 | Missing scopes on Private App | Re-create Private App with correct scopes |
| ClickUp node fails | Hierarchy IDs wrong | Re-check Team/Space/Folder/List IDs via ClickUp URLs |
| Google Sheets append fails | Wrong spreadsheet ID or sheet name | Verify ID from URL, verify tab name exact (case-sensitive) |
| Offer gate returns 200 but no Slack message | Credential not linked | Link Slack credential to the `Slack — Approval Request` node |
| Slack approval never resumes workflow | Bot not invited to channel | `/invite @your-bot-name` in the ops channel |
