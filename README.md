# AI-Powered Email Automation System with WhatsApp Approval

An enterprise-grade, asynchronous email automation engine for multi-tenant environments. The system ingests incoming emails, classifies intent with an LLM, extracts data, and drafts a reply. Before anything is sent, a human approver reviews the draft on **WhatsApp** and approves, rejects, or edits it (human-in-the-loop).

Built with FastAPI, Redis, Celery, PostgreSQL, and LiteLLM.

## Architecture & Flow

```
Incoming Email
      |
      v
Webhook (SendGrid / Mailgun)  or  IMAP polling
      |
      v
FastAPI ingestion API  --->  Redis / Celery queue
                                    |
                                    v
                          Background worker
                  (tenant context from PostgreSQL,
                   prompt build, LLM via LiteLLM:
                   intent classification + draft)
                                    |
                                    v
                       Draft saved: PENDING_APPROVAL
                                    |
                                    v
                    WhatsApp message to approver
                    (Meta Graph API, with buttons)
                                    |
                    +---------------+----------------+
                    |               |                |
                 Approve          Edit            Reject
                    |               |                |
                    |        Approver replies        |
                    |        with revised text       |
                    |               |                |
                    |        Updated draft sent      |
                    |        for re-approval         |
                    v                                v
        Send via SMTP / email API              Mark REJECTED
                    |
                    v
                  SENT
```

1. **Ingestion:** Receives incoming emails via webhooks (for example SendGrid or Mailgun) or IMAP polling.
2. **Authentication:** Secures API endpoints with stateless JWT authentication and a Redis-backed token blacklist.
3. **Queueing:** Offloads heavy processing to background workers (Redis/Celery) so the ingestion API stays responsive.
4. **AI Processing:** Retrieves tenant context from PostgreSQL, builds the prompt, and queries the LLM (via LiteLLM) to classify intent, extract data, and draft a response.
5. **Human Approval:** Sends the draft to the tenant's approver on WhatsApp. Nothing is sent to the customer until the approver responds.
6. **Execution:** On approval, dispatches the reply via SMTP or a third-party email API.

## WhatsApp Human-in-the-Loop Approval

### How it works

After the LLM generates a draft, the worker stores it with status `PENDING_APPROVAL` and sends the approver a WhatsApp message through the Meta Graph API. The message includes:

- The original sender and subject
- A short summary of the incoming email and the detected intent
- The drafted reply
- Reply buttons: **Approve**, **Edit**, and **Reject**

Each button carries a payload containing the approval ID, so the reply can be matched to the correct draft.

### Approver actions

| Action | Result |
|---|---|
| **Approve** | The draft is queued for sending. Status moves to `APPROVED`, then `SENT`. |
| **Edit** | The approver replies with revised text. The draft is updated and sent back for confirmation. |
| **Reject** | The draft is discarded. Status moves to `REJECTED`, and the email can be flagged for manual handling. |

### Approval states

```
RECEIVED -> PROCESSING -> PENDING_APPROVAL -> APPROVED -> SENT
                                  |
                                  +--> REJECTED
                                  +--> EXPIRED  (no response within the timeout)
                                  +--> FAILED   (send or processing error)
```

### Design notes

- **Message templates:** WhatsApp only allows free-form and interactive messages within 24 hours of the user's last message. Approval requests initiated by the system should use a pre-approved WhatsApp message template (with quick-reply buttons) created in Meta Business Manager. Once the approver replies, the 24-hour window opens and free-form edits can be exchanged.
- **Per-tenant approvers:** Each tenant has one or more configured approver phone numbers. Only replies from those numbers are accepted for that tenant's drafts.
- **Timeouts and reminders:** A Celery beat task sends a reminder after a configurable period and marks unanswered drafts as `EXPIRED`. Expired drafts are never sent automatically.
- **Idempotency:** Each approval request can be acted on once. Duplicate webhook deliveries or repeated button taps are ignored.
- **Audit trail:** Every approval event (sent, approved, edited, rejected, expired) is stored with a timestamp and approver number.
- **Webhook security:** WhatsApp webhook requests are verified using the `X-Hub-Signature-256` header and your Meta app secret before any action is taken.
- **Optional auto-send rules:** You can configure per-tenant rules (for example, auto-send for low-risk intents above a confidence threshold) while keeping approval mandatory for everything else.

## Tech Stack

| Area | Technology |
|---|---|
| Backend | Python 3.12, FastAPI |
| Database | PostgreSQL (pgvector supported for RAG) |
| Caching & Queues | Redis, Celery |
| AI / LLM | LiteLLM / LangChain (routing and LLM agnosticism) |
| Approval channel | WhatsApp Business Platform, Meta Graph API |
| Infrastructure | Docker, Docker Compose |

## Prerequisites

- Python 3.12
- Docker & Docker Compose
- Redis server (if running locally without Docker)
- PostgreSQL
- OpenAI / Gemini / Anthropic API key
- A WhatsApp Business Platform account with a phone number ID, access token, and an approved message template
- A public HTTPS URL for webhooks (Cloudflare Tunnel or ngrok for local development)

## Environment Variables

Create a `.env` file in the root directory:

| Variable | Description | Example |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@localhost:5432/email_db` |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379/0` |
| `JWT_SECRET_KEY` | Secret for signing tokens | `your-highly-secure-secret-key` |
| `OPENAI_API_KEY` | LLM provider API key | `sk-proj-...` |
| `SMTP_HOST` | Outbound email server | `smtp.sendgrid.net` |
| `SMTP_USER` | Email server username | `apikey` |
| `SMTP_PASS` | Email server password | `your-smtp-password` |
| `WHATSAPP_ACCESS_TOKEN` | Meta Graph API access token | `EAAG...` |
| `WHATSAPP_PHONE_NUMBER_ID` | WhatsApp sender phone number ID | `1234567890` |
| `WHATSAPP_API_VERSION` | Graph API version | `v21.0` |
| `WHATSAPP_VERIFY_TOKEN` | Token used to verify the webhook subscription | `your-verify-token` |
| `WHATSAPP_APP_SECRET` | Meta app secret for webhook signature checks | `your-app-secret` |
| `WHATSAPP_APPROVAL_TEMPLATE` | Name of the approved approval-request template | `email_approval_request` |
| `APPROVAL_TIMEOUT_MINUTES` | Time before a pending draft expires | `120` |
| `APPROVAL_REMINDER_MINUTES` | Time before a reminder is sent | `30` |

Use the exact variable names expected by your application code. Never commit `.env` to source control.

## Local Installation

1. Clone and activate the virtual environment:

```bash
git clone https://github.com/your-org/email-automation.git
cd email-automation
python -m venv venv
source venv/bin/activate  # On Windows: .\venv\Scripts\activate
```

2. Install dependencies:

```bash
pip install -r requirements.txt
```

3. Run database migrations (Alembic):

```bash
alembic upgrade head
```

4. Start the FastAPI server:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

5. Start the background worker:

```bash
celery -A worker.celery_app worker --loglevel=info
```

6. Start the scheduler (approval reminders and expiry):

```bash
celery -A worker.celery_app beat --loglevel=info
```

7. Expose the server for webhooks (local development):

```bash
cloudflared tunnel --url http://localhost:8000
# or
ngrok http 8000
```

## Docker Deployment

To start the entire stack (FastAPI, Redis, PostgreSQL, Celery worker, and Celery beat) with Docker Compose:

```bash
docker-compose up --build -d
```

## WhatsApp Setup

1. In Meta Business Manager, set up the WhatsApp Business Platform and obtain your phone number ID and access token.
2. Create and submit a message template for approval requests, with quick-reply buttons for Approve, Edit, and Reject, and variables for sender, subject, and draft text.
3. In your Meta app's WhatsApp configuration, set the webhook callback URL to `https://<your-domain>/api/v1/webhooks/whatsapp` and the verify token to the value of `WHATSAPP_VERIFY_TOKEN`.
4. Subscribe to the `messages` webhook field.
5. Register each tenant's approver phone number(s) in the application (international format).

## Core API Endpoints

| Method | Endpoint | Auth Required | Description |
|---|---|---|---|
| POST | `/api/v1/auth/login` | No | Authenticates tenant and issues JWT |
| POST | `/api/v1/auth/logout` | Yes | Blacklists current JWT in Redis |
| POST | `/api/v1/webhooks/email` | API Key | Ingests raw email payloads from providers |
| GET | `/api/v1/webhooks/whatsapp` | Verify token | Meta webhook subscription verification |
| POST | `/api/v1/webhooks/whatsapp` | Signature | Receives approver replies and button taps |
| GET | `/api/v1/emails/status/{id}` | Yes | Checks processing and approval status of an email |
| GET | `/api/v1/approvals/{id}` | Yes | Retrieves approval details, draft, and audit trail |
| POST | `/api/v1/approvals/{id}/resend` | Yes | Re-sends the approval request to the approver |
| POST | `/api/v1/templates/create` | Yes | Creates an LLM prompt template for a specific tenant |

Adjust paths to match your implementation.

## Webhook Payload Structure

When configuring your email provider (for example SendGrid Inbound Parse), route the webhook to `/api/v1/webhooks/email`. The system expects a standard JSON payload:

```json
{
  "sender": "client@example.com",
  "recipient": "support@van-kaal.com",
  "subject": "System Outage Update",
  "body_text": "Hello, when will the server be back online?",
  "tenant_id": "tenant_abc123"
}
```

## Example Approval Flow

1. A customer emails: "Hello, when will the server be back online?"
2. The system classifies the intent as `status_inquiry` and drafts a reply.
3. The approver receives a WhatsApp message:

   > **Approval needed** (tenant_abc123)
   > From: client@example.com
   > Subject: System Outage Update
   > Draft reply: "Hello, our team is working on the outage and we expect service to be restored by 3 PM. We'll update you as soon as it's resolved."
   > [Approve] [Edit] [Reject]

4. The approver taps **Approve**. The email is sent and the status becomes `SENT`.

## Security

- **Multi-Tenancy:** Strict data isolation enforced via the JWT `sub` claim and explicit `tenant_id` database queries.
- **Stateless Auth:** Short-lived access tokens (15 minutes) with a Redis-managed token blacklist for immediate revocation on logout.
- **Approver Verification:** Only replies from registered approver numbers for the matching tenant are accepted.
- **Webhook Verification:** WhatsApp webhooks are validated with the `X-Hub-Signature-256` signature; email webhooks use an API key.
- **Safe by Default:** Drafts are never sent without approval unless an explicit auto-send rule is configured for that tenant and intent.
- **Auditability:** All approval actions are logged with timestamps and approver identity.
