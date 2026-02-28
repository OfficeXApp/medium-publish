---
name: medium-publish
description: |
  AI-powered blog content generation, approval, scheduling, and volunteer publishing via REST API. Use when: (1) Generating SEO-optimized blog posts in bulk, (2) Approving/rejecting AI-generated content, (3) Scheduling posts and coordinating volunteer publishers, (4) Receiving webhooks for job completion, schedule firing, and proof submission, (5) Building automations around the Job > Results > Approve > Schedule > Fire > Proof lifecycle, (6) Querying calendar views of scheduled content.
---

# Blog Publish

AI-powered blog content generation and volunteer publishing pipeline. Generate SEO-optimized posts in bulk, approve/reject, schedule for publication, and coordinate volunteers — all via REST API.

> **Get started on OfficeX:** Subscribe at [officex.app](https://officex.app) and install this app from the store: [officex.app/store/en/app/medium-publish](https://officex.app/store/en/app/medium-publish)

## Prerequisites

After installing the app on OfficeX, you'll receive credentials via the install webhook. Set these in your `.env`:

```bash
# Required — from OfficeX app install
OFFICEX_INSTALL_ID="your_install_id"        # Provided on install
OFFICEX_INSTALL_SECRET="your_install_secret" # Provided on install

# Required — API base URL
BLOG_PUBLISH_API_URL="https://blog-publisher.cloud.zoomgtm.com"
```

The `OFFICEX_INSTALL_ID` and `OFFICEX_INSTALL_SECRET` are provided automatically when you install the app on OfficeX. Use them to authenticate:

```bash
# Login to get an API key
curl -X POST $BLOG_PUBLISH_API_URL/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"install_id": "'$OFFICEX_INSTALL_ID'", "install_secret": "'$OFFICEX_INSTALL_SECRET'"}'
```

## Content Lifecycle

```
1. CREATE JOB     POST /jobs { content_prompt, output_quantity }
                  -> Returns immediately, processes async
                  -> Credits reserved upfront, settled progressively
                  -> Fires JOB_COMPLETED webhook when done

2. REVIEW         GET /jobs/:jid/results -> list generated posts
                  PATCH /results/:rid?job_id= { approved: true }

3. SCHEDULE       POST /results/:rid/schedule?job_id= { scheduled_datetime }
                  OR POST /results/:rid/auto-schedule?job_id= (AI picks time)

4. FIRE           Automatic hourly (posts due within 60 min)
                  OR manual: POST /scheduled/:sid/fire
                  -> Fires SCHEDULE_FIRED webhook with content + volunteer magic link
                  -> Sends email to volunteer with publishing instructions

5. PROOF          Volunteer publishes content, then:
                  POST /volunteer/:sid/proof { proof_url }
                  -> Fires PROOF_SUBMITTED webhook
                  -> Status -> COMPLETED
```

## Authentication

**Login:**
```
POST /auth/login
Body: { "install_id": "<officex_install_id>", "install_secret": "<officex_install_secret>" }
Response: { "success": true, "user": { "user_id", "email", ... }, "api_key": "..." }
```

**Auth methods (any one):**
1. `Authorization: Bearer <api_key>` (primary)
2. Headers: `x-officex-install-id` + `x-officex-install-secret`
3. Query params: `officex_install_id` + `officex_install_secret`

**Get current user:**
```
GET /auth/me -> { success, user: { user_id, email, internal_credits, ... } }
```

## API Reference

### Jobs

**POST /jobs** — Create a content generation job (async)
```json
{
  "content_prompt": "Write about sustainable farming practices",
  "output_quantity": 5,
  "seo_prompt": "organic farming, sustainability, eco-friendly",
  "tone_prompt": "Professional but approachable",
  "publish_prompt": "Publish on Medium under the Agriculture tag",
  "schedule_prompt": "Weekday mornings, space posts 2-3 days apart",
  "instruction_prompt": "Instructions shown to the volunteer publisher",
  "destination_url": "https://medium.com/@yourblog",
  "metadata_schema": { "category": "string", "region": "string" },
  "on_job_done_webhook": "https://your-server.com/hooks/job-done",
  "on_job_done_email": "you@email.com",
  "on_scheduled_webhook": "https://your-server.com/hooks/scheduled",
  "on_scheduled_email": "volunteer@email.com",
  "on_proof_webhook": "https://your-server.com/hooks/proof",
  "on_proof_email": "you@email.com",
  "tracer": "campaign-123",
  "inbox_tracer": "batch-abc",
  "notes": "Q1 content batch"
}
```
Required: `content_prompt`. Everything else optional. `output_quantity` defaults to 5.

Returns `201` with job in `PENDING` status. Job transitions: `PENDING -> PROCESSING -> COMPLETED` (or `FAILED`).

**GET /jobs** `?limit=25&cursor=<base64>`
**GET /jobs/:jid** — Full job detail
**PATCH /jobs/:jid** — `{ title?, notes?, bookmarked? }`
**DELETE /jobs/:jid** — Only if PENDING or FAILED
**GET /jobs/:jid/results** `?limit=25&cursor=<base64>`

### Results

All result endpoints require `?job_id=<jid>` query param.

**GET /results/:rid?job_id=**

**PATCH /results/:rid?job_id=** — Approve, reject, or edit
```json
{
  "approved": true,
  "rejected": false,
  "user_notes": "Great post, minor edits needed",
  "bookmarked": true,
  "scheduled_datetime": "2026-03-15T09:00:00Z",
  "on_scheduled_email": "volunteer@email.com",
  "on_scheduled_webhook": "https://...",
  "on_proof_email": "you@email.com",
  "on_proof_webhook": "https://...",
  "destination_url": "https://medium.com/@yourblog",
  "tracer": "campaign-123",
  "inbox_tracer": "batch-abc"
}
```
Setting `approved: true` auto-clears `rejected`. Setting `rejected: true` auto-clears `approved`. Notification config changes propagate to the Scheduled record if already scheduled.

**POST /results/:rid/schedule?job_id=** — Schedule an approved result
```json
{
  "scheduled_datetime": "2026-03-15T09:00:00Z",
  "timezone": "America/New_York",
  "password_protected": "secret123"
}
```
Required: `scheduled_datetime`. Result must be approved first.

**POST /results/:rid/auto-schedule?job_id=** — AI suggests optimal time
Returns `{ suggested_datetime, reasoning }`. Uses the job's `schedule_prompt` and existing schedule to avoid conflicts.

### Scheduled

**GET /scheduled** `?limit=50&cursor=<base64>`
**GET /scheduled/:sid**

**PATCH /scheduled/:sid** — Reschedule or update config (cannot update if already fired)
```json
{
  "scheduled_datetime": "2026-03-16T10:00:00Z",
  "timezone": "America/New_York",
  "on_schedule_email": "volunteer@email.com",
  "on_schedule_webhook": "https://...",
  "on_proof_email": "you@email.com",
  "on_proof_webhook": "https://...",
  "destination_url": "https://...",
  "password_protected": "new-password",
  "tracer": "...",
  "inbox_tracer": "..."
}
```

**DELETE /scheduled/:sid** — Cancel (also unschedules the result)

**POST /scheduled/:sid/fire** — Fire immediately (sends webhook + email now)

**POST /scheduled/:sid/proof** — Submit proof (public endpoint, no auth)
```json
{
  "proof_url": "https://medium.com/@blog/my-published-post-abc123",
  "reported_by": "volunteer@email.com",
  "metadata": { "views": 150 }
}
```
If password-protected, pass `?password=<pwd>` query param.

### Calendar

**GET /calendar/month/:year/:month** — All scheduled posts for a month
**GET /calendar/week/:year/:week** — Weekly view
**GET /calendar/day/:year/:month/:day** — Daily view

All return `{ success, scheduled: [...], by_day: { "YYYY-MM-DD": [...] }, total }`.

### Volunteer (public, no auth)

**GET /volunteer/:sid** `?password=<pwd>` — Volunteer-safe view (title, body, instructions, destination)
**POST /volunteer/:sid/proof** `?password=<pwd>` — Submit proof URL

## Outbound Webhooks

Configure webhook URLs at job creation. They propagate: Job defaults -> Result overrides -> Scheduled record.

### JOB_COMPLETED
Sent when async job processing finishes (all posts generated).
```json
{
  "id": "uuid",
  "event": "JOB_COMPLETED",
  "payload": {
    "job_id": "...",
    "user_id": "...",
    "title": "AI-generated job title",
    "status": "COMPLETED",
    "results_created": 5,
    "results_approved": 0,
    "credits_spent": 2.06,
    "tracer": "campaign-123"
  },
  "timestamp": "2026-03-15T09:00:00Z"
}
```

### SCHEDULE_FIRED
Sent when the scheduler fires a post (hourly cron or manual `/fire`).
```json
{
  "id": "uuid",
  "event": "SCHEDULE_FIRED",
  "payload": {
    "scheduled_id": "...",
    "result_id": "...",
    "job_id": "...",
    "user_id": "...",
    "title": "Blog Post Title",
    "subheading": "...",
    "body": "Full markdown content...",
    "instruction_prompt": "Instructions for the volunteer",
    "volunteer_magic_link": "https://domain/volunteer/:sid?password=...",
    "submit_proof_endpoint": "https://domain/api/scheduled/:sid/proof?password=...",
    "tracer": "campaign-123",
    "inbox_tracer": "batch-abc"
  },
  "timestamp": "2026-03-15T09:00:00Z"
}
```

### PROOF_SUBMITTED
Sent when a volunteer submits proof of publication.
```json
{
  "id": "uuid",
  "event": "PROOF_SUBMITTED",
  "payload": {
    "scheduled_id": "...",
    "result_id": "...",
    "job_id": "...",
    "user_id": "...",
    "proof_url": "https://medium.com/...",
    "proof_timestamp": "2026-03-15T10:30:00Z",
    "metadata": {},
    "title": "Blog Post Title",
    "tracer": "campaign-123",
    "inbox_tracer": "batch-abc"
  },
  "timestamp": "2026-03-15T10:30:00Z"
}
```

## Credit Costs

| Operation         | Credits |
|-------------------|---------|
| Blog post         | 0.4     |
| AI job title      | 0.04    |
| Infrastructure    | 0.02    |

Typical 5-post job: ~2.06 credits (0.04 title + 5 x 0.4 posts + 0.02 infra). Credits are reserved upfront and settled progressively as posts generate.

## Notification Config Hierarchy

Webhook/email config flows downhill and can be overridden at each level:

```
Job (defaults)  ->  Result (per-post overrides)  ->  Scheduled (per-task overrides)
```

Fields at each level:
- **Job done**: `on_job_done_email`, `on_job_done_webhook` (Job only)
- **Schedule notifications**: `on_scheduled_email`, `on_scheduled_webhook` (Result) / `on_schedule_email`, `on_schedule_webhook` (Scheduled)
- **Proof notifications**: `on_proof_email`, `on_proof_webhook`
- **Routing**: `destination_url`, `tracer`, `inbox_tracer`

Updating a Result's notification config automatically propagates to its Scheduled record (if not yet fired).

## Response Format

All responses follow:
```json
{
  "success": true,
  "job": { ... },
  "cursor": "base64-encoded-pagination-token"
}
```

Errors:
```json
{
  "success": false,
  "error": { "code": "NOT_FOUND", "message": "Job not found" }
}
```

Common error codes: `INVALID_REQUEST`, `NOT_FOUND`, `NOT_APPROVED`, `ALREADY_SCHEDULED`, `ALREADY_FIRED`, `INVALID_STATE`, `BILLING_FAILED` (402), `UNAUTHORIZED` (401).

## Entity Statuses

| Entity    | Statuses                                          |
|-----------|---------------------------------------------------|
| Job       | `PENDING` -> `PROCESSING` -> `COMPLETED` / `FAILED` |
| Scheduled | `PENDING` -> `SENT` -> `COMPLETED`                |
