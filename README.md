# Impact Dashboard API

Public API documentation for automation systems that create and update client records, calls, payments, engagement events, onboarding milestones, placement stages, Discord IDs, track assignments, development documents, and 1-on-1 booking requests in Impact Dashboard.

## Contents

- [Overview](#overview)
- [Typical integration flow](#typical-integration-flow)
- [Quickstart](#quickstart)
- [Base URL](#base-url)
- [Authentication](#authentication)
- [Request Rules](#request-rules)
- [Responses And Errors](#responses-and-errors)
- [Endpoint Summary](#endpoint-summary)
- [Endpoint Reference](#endpoint-reference)
- [Curl Examples](#curl-examples)

## Overview

Use this API when an external automation system needs to write lifecycle data into Impact Dashboard. The API accepts JSON over HTTPS and returns JSON responses.

Typical integrations use the API to:

- Create a client after a sale or enrollment event.
- Store onboarding, Discord, track, payment, and engagement updates against that client.
- Create scheduled calls and update the call after it is completed, missed, cancelled, or rebooked.
- Attach generated development documents to a client and, optionally, to a specific call.

### Typical integration flow

Most automations follow this shape:

1. **Create the client** with `POST /api/webhooks/clients` and a stable CRM **`contactid`**. Use that same `contactid` as the `{contactId}` in every later contact-scoped URL.
2. **Parallel enrichment updates** as data arrives: onboarding milestones (`PATCH .../onboarding`), Discord ids (`PATCH .../discord`), track/group (`PATCH .../track`).
3. **Calls**: create with `POST .../calls`, then later finalize with `PATCH /api/webhooks/calls/{callId}`. Keep the returned **`call.id`** from step one if you need to patch by call id.
4. **Financial and activity events**: payments (`POST .../payments`) and engagement (`POST .../engagement`) should use idempotency keys or stable external ids when retries are possible.
5. **Weekly dev docs** (`POST .../dev-docs`) optionally reference an existing `call_id`.

You only need the endpoints your automation actually produces; there is no requirement to call every route.

## Quickstart

1. Get webhook credentials from the Impact Dashboard administrator.
2. Send every request with `Content-Type: application/json`, **`Accept: application/json`**, and your **`x-api-key`**.
3. Create a client with `POST /api/webhooks/clients`.
4. Send a CRM `contactid` when creating the client.
5. Use that same `contactid` as `{contactId}` in all contact-scoped endpoint paths.
6. Store returned `call.id` values when creating calls so they can be updated later.
7. Treat every non-2xx response as a failed write and log the response body for troubleshooting.

## Base URL

```text
https://dashboard.impactteam.us
```

All paths in this document are relative to this base URL. The same origin is defined in code as `PRODUCTION_APP_URL` in [`lib/app-url.ts`](lib/app-url.ts).

## Authentication

All endpoints require an API key. Send it as the `x-api-key` header on every request.

```http
x-api-key: API_KEY
Content-Type: application/json
Accept: application/json
```

Get your API key (`WEBHOOK_API_KEY`) from the Impact Dashboard administrator. Do not send API keys in query parameters or request bodies. Do not paste real credentials into documentation, screenshots, logs, or source code.

**Network allowlists**

Some deployments restrict webhook callers by IP. If your IP is not allowed, responses return HTTP **`403`** with:

```json
{
  "success": false,
  "error": "Request origin not permitted."
}
```

Authentication failures return HTTP `401`:

```json
{
  "success": false,
  "error": "Invalid webhook credentials."
}
```

If the server operator has **not** set **`WEBHOOK_API_KEY`**, all requests receive HTTP **`401`**. Server logs may record that **`WEBHOOK_API_KEY`** is unset when diagnosing failed deliveries.

## Request Rules

- Request bodies must be valid JSON **objects** (not bare arrays or primitives).
- **Extra JSON keys:** On `POST /api/webhooks/clients`, `POST /api/webhooks/b2b/clients`, and `POST /api/webhooks/b2b/eod`, unknown keys are **ignored** (silently dropped). On **every other** webhook endpoint in this document, unknown keys cause **`400`** validation errors because schemas are strict.
- Required strings must be non-empty after trimming.
- Optional string fields may be omitted, but if present must be non-empty after trimming.
- Dates must be ISO 8601 datetimes with a timezone offset, for example `2026-05-01T10:00:00.000Z` or `2026-05-01T18:00:00+08:00`.
- Money fields may be JSON numbers or numeric strings such as `2500`, `"2500"`, or `"2500.00"`.
- JSON metadata fields (where documented) may contain strings, numbers, booleans, null, arrays, or nested objects, but nested metadata is capped at **8 levels** deep and **500** scalar/array/object nodes total per field.
- Contact-scoped paths use the `{contactId}` path parameter, which accepts either the GHL CRM `contactid` or the dashboard's internal client id.
- Call-specific paths use the dashboard call id returned by successful create-call responses.

### Payload limits

- Maximum JSON body size is **256 KiB**.
- Oversized bodies fail validation like malformed JSON (typically **`400`** `Invalid request payload.`).

## Responses And Errors

### Success Responses

Create endpoints return HTTP `201`. Update endpoints return HTTP `200`.

Successful responses use this shape:

```json
{
  "success": true,
  "<resource>": {}
}
```

### Validation Errors

Validation error responses return HTTP `400`:

```json
{
  "success": false,
  "error": "Invalid request payload.",
  "details": {
    "formErrors": [],
    "fieldErrors": {
      "email": ["Invalid email address"]
    }
  }
}
```

### Contact ID

All contact-scoped routes use a `{contactId}` path parameter. This accepts **either** the GHL CRM `contactid` sent during client creation **or** the dashboard's internal client id (returned in the `id` field of the create-client response). Using the GHL `contactid` directly is the recommended approach for automations - there is no need to store or forward the internal id.

### Common Errors

| Status | Message | Meaning |
| --- | --- | --- |
| `400` | `Invalid request payload.` | JSON parsed but failed schema validation, or a path ID was blank/invalid. |
| `401` | `Invalid webhook credentials.` | Missing or incorrect **`x-api-key`**, or **`WEBHOOK_API_KEY`** is unset on the server (see [Authentication](#authentication)). |
| `403` | `Request origin not permitted.` | Caller IP not in the server's webhook IP allowlist (when configured). |
| `404` | `Client not found.` | The `{contactId}` path parameter does not match a client by internal id or `contactid`. |
| `404` | `Call not found.` | The `{callId}` path parameter or linked `call_id` does not match a call. |
| `409` | Resource-specific conflict message | A unique value already exists. |
| `429` | `Too many requests.` | Webhook rate limit exceeded for this IP; retry with backoff. |
| `500` | Resource-specific failure message | Unexpected server or database failure. Retry with backoff and alert if repeated. |

### Rate Limits

Webhook routes may return HTTP **`429`** with `Too many requests.` when a caller exceeds the configured per-IP limit for `/api/webhooks` (the reference implementation uses **120 requests per rolling minute** per IP). Treat **`429`** like other transient errors: back off and retry.

In **this repository**, those limits are enforced in **`proxy.ts`** at the project root (Next.js 16's proxy hook convention; **`middleware.ts`** is not used). Paths under **`/api/auth`** share a tighter per-IP limit for abuse protection.

For payment, engagement, and placement survey webhooks, send an `Idempotency-Key` header or a stable external event/payment/survey id in the request body where documented. Replays with the same key are rejected as duplicates instead of creating a second record.

## Endpoint Summary

| Name | Method | Path | Purpose |
| --- | --- | --- | --- |
| Create Client | `POST` | `/api/webhooks/clients` | Create a client from a sale or enrollment event. |
| Update Onboarding | `PATCH` | `/api/webhooks/contacts/{contactId}/onboarding` | Move a client to an onboarding milestone and record metadata as an engagement event. |
| Update Discord IDs | `PATCH` | `/api/webhooks/contacts/{contactId}/discord` | Save the client's Discord channel and user IDs. |
| Update Track | `PATCH` | `/api/webhooks/contacts/{contactId}/track` | Assign the client to a dashboard track and group. |
| Create Call | `POST` | `/api/webhooks/contacts/{contactId}/calls` | Create a scheduled call record. |
| Update Call | `PATCH` | `/api/webhooks/calls/{callId}` | Update a call after completion, no-show, cancellation, or rebooking. |
| Create Payment | `POST` | `/api/webhooks/contacts/{contactId}/payments` | Record a backend payment and recalculate remaining balance. |
| Create B2B Company | `POST` | `/api/webhooks/b2b/clients` | Create a B2B company from the onboarding form and optionally pre-register its sales reps. Idempotent on `contactid`. |
| Record B2B EOD | `POST` | `/api/webhooks/b2b/eod` | Record a B2B sales rep's end-of-day numbers for a day, resolved by the company's `contactid` plus the rep's email (upserted per rep per day). |
| Record Sales Tag Event | `POST` | `/api/webhooks/sales/tag-events` | Record a CRM tag applied to a sales-funnel contact, with the moment it was applied. Upserts the contact if it does not exist yet. |
| Create Sales Contact | `POST` | `/api/webhooks/sales/contacts` | Record a sales-funnel contact creation from the CRM. |
| Record Dialer Call | `POST` | `/api/webhooks/sales/dialer-calls` | Record one dialer call event (dials, pickups, duration, disposition), matched to contacts by phone. |
| Record Call Attendance | `POST` | `/api/webhooks/sales/attendance` | Record contact presence for one imp/strategy call; the Showed/No Show verdict is computed server-side. |
| Record Weekly Ad Spend | `POST` | `/api/webhooks/sales/ad-spend` | Record one weekly ad-spend figure (Monday-start weeks, upserted per week and source). |
| Record Cash Collected | `POST` | `/api/webhooks/sales/cash` | Record one cash-collection event (closed sale) from the A1 mastersheet, matched to a funnel contact when possible; unmatched sales are stored but excluded from dashboard cash totals until they link. |
| Record Slot Utilization | `POST` | `/api/webhooks/sales/slot-utilization` | Record one day's booked/available slot counts for one sales calendar. |
| Create Engagement Event | `POST` | `/api/webhooks/contacts/{contactId}/engagement` | Log client activity or learning engagement. |
| Update Placement | `POST` | `/api/webhooks/contacts/{contactId}/placement` | Sync a client's placement stage from a monthly survey. |
| Create Development Document | `POST` | `/api/webhooks/contacts/{contactId}/dev-docs` | Save an AI-generated or automation-generated development document. |
| Create 1-on-1 Booking Request | `POST` | `/api/webhooks/contacts/{contactId}/one-on-one-requests` | Store a client's 1-on-1 booking form submission for operator approval or denial. |
| Create Refund Request | `POST` | `/api/webhooks/contacts/{contactId}/refund-requests` | Store an internal refund request for the approver to approve or deny. |
| Enroll Accelerator Member | `POST` | `/api/webhooks/accelerator/enrollments` | Create or update an Impact Accelerator membership. |
| Update Accelerator Onboarding | `PATCH` | `/api/webhooks/accelerator/{contactId}/onboarding` | Record PandaDoc, access, onboarding, portal, and certification milestones. |
| Sync Accelerator RSVP Count | `PATCH` | `/api/webhooks/accelerator/{contactId}/rsvp-count` | Mirror the GHL "IA Coaching Calls RSVP Count" custom field onto the membership. |

## Endpoint Reference

### POST /api/webhooks/clients - Create or Update Client (sale event)

Creates a dashboard client from a sale or enrollment event. Extra JSON keys in the body are **ignored** on this route only.

**Repeat sales (upsells):** when a client with the same `contactid` or `email` already exists (the `contactid` match wins when they differ), the call is an update, not a conflict, returning HTTP `200` with `"updated": true` instead of `201`. Fields present in the payload (name, phone, program, lead source, closer/setter, sale date, revenue, payment plan, notes, doc URL, agreement URL, pod types, client type) replace the stored values; omitted fields are preserved. `contract_value` is **added** to the existing contract value (a second sale increases what the client owes; sending a cumulative figure would double-count). A missing `contactid` link is filled in; an existing different link is never overwritten. Email is never changed by this route.

**Redelivery safety:** send an `Idempotency-Key` header to have an exact replay rejected with `409 Duplicate sale webhook.` Without a key, a redelivered sale that carries `revenue` (plus optionally `cash_collected`) **and** `sale_date` is detected through the seed payment the first delivery created (same cash, identical `sale_date`), and neither accumulates the contract again nor re-seeds. This key-less detection has limits: it does not cover a sale without `revenue` and `sale_date`, and it lapses once the seed has been claimed by a payment delivery that carried an `external_payment_id` (the claim replaces the seed's identifying id). Senders that retry must supply the `Idempotency-Key` header; the seed-based detection is a fallback, not the contract. The new sale's payment is seeded **unless** a payment with exactly the same cash already exists within 7 days: the back-end form may record the same money in either order, so an unrelated identical-cash payment inside that window (for example equal weekly installments) suppresses the seed until the payment webhook records it.

```http
POST /api/webhooks/clients
```

Headers:

```http
x-api-key: <API_KEY>
Content-Type: application/json
Accept: application/json
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `contactid` | string | Yes | CRM contact ID. Stored and used to identify the client in all follow-up webhook calls, and to link the client to their funnel lead (see [Linking leads to clients](#linking-leads-to-clients)); send the same value here and as `contact_id` on `POST /api/webhooks/sales/contacts`. |
| `name` | string | Yes | Client full name. |
| `email` | string | Yes | Client email. Lowercased and validated as an email address. Must be unique. |
| `phone` | string | No | Client phone number. |
| `program` | string | No | Program name as sold (e.g. `Platinum`, `Platinum Upsell (The House)`, `VIP Plus+`, `B2B Package`). Free-form: new program names are accepted without an API change. |
| `lead_source` | enum | No | One of `Referral / Other`, `Outbound`, `VIP Onboarding`, `Application Lead`, `Discord DM's`, `Offer Placement Call`. |
| `closer` | enum | No | One of `Sam`, `Dan`, `Hunter`, `Phillip`. |
| `setter` | enum | No | One of `Sam`, `Dan`, `Hunter`, `Phillip`. |
| `sale_date` | ISO datetime | No | Sale timestamp with timezone. |
| `contract_value` | money | No | Total contract value. |
| `revenue` | money | No | Revenue amount to display. |
| `cash_collected` | money | No | Cash collected at sale. |
| `remaining_balance` | money | No | Accepted but not authoritative. The remaining balance is always computed as contract value minus cash collected, so a value sent here is ignored. |
| `payment_plan` | string | No | Payment plan description. |
| `context_notes` | string | No | Internal context notes. |
| `development_doc_url` | URL | No | Link to the client's living Development Doc. |
| `signed_agreement_url` | URL | No | Link to the client's signed PandaDoc agreement. Sent once the client has completed/signed the document. Like `development_doc_url`, omitting it on a later update preserves the stored value. |
| `pod_types` | string or array | No | Coaching track type(s). Accepts a JSON array (`["SALES","MINDSET"]`) or a comma-separated string (`"SALES,MINDSET"`). Each value must be one of `SALES`, `MINDSET`. Unknown values are silently ignored. Defaults to `[]`. |
| `client_type` | enum | No | `B2B` or `B2C`. Set `B2B` for companies whose sales reps report daily numbers (see the B2B EOD endpoint); the company then appears in the dashboard B2B section. Defaults to `B2C`. The dashboard displays `B2C` as "Individual". |

Default values set by the API:

| Field | Default |
| --- | --- |
| `paymentStatus` | `INCOMPLETE` |
| `track` | `BEGINNER` |
| `groupAssignment` | `BEGINNER` |
| `healthStatus` | `GREEN` |
| `onboardingStatus` | `PAID` |
| `clientType` | `B2C` |

Example request:

```json
{
  "contactid": "zMC7sAfinnBzqYy8n98V",
  "name": "Jamie Rivera",
  "email": "Jamie.Rivera@example.com",
  "phone": "+15555550123",
  "program": "Platinum",
  "lead_source": "Application Lead",
  "closer": "Sam",
  "setter": "Dan",
  "sale_date": "2026-05-01T10:00:00.000Z",
  "contract_value": "5000.00",
  "revenue": "5000.00",
  "cash_collected": "1000.00",
  "remaining_balance": "4000.00",
  "payment_plan": "4 monthly payments",
  "context_notes": "Prefers evening calls.",
  "development_doc_url": "https://docs.google.com/document/d/example",
  "signed_agreement_url": "https://app.pandadoc.com/document/abc123"
}
```

Example success response, HTTP `201`:

```json
{
  "success": true,
  "client": {
    "id": "clwclient123",
    "contactId": "zMC7sAfinnBzqYy8n98V",
    "name": "Jamie Rivera",
    "email": "jamie.rivera@example.com",
    "track": "BEGINNER",
    "groupAssignment": "BEGINNER",
    "onboardingStatus": "PAID",
    "healthStatus": "GREEN",
    "remainingBalance": "4000",
    "developmentDocUrl": "https://docs.google.com/document/d/example",
    "createdAt": "2026-05-01T10:00:02.000Z",
    "updated": false
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `contactid`, `name`, or `email`, invalid email, invalid URL, or invalid date/money format. Unknown extra fields are ignored, not rejected. |
| `409` | `Client already exists.` | Only reachable through a race between two concurrent creates; a repeat sale for an existing client is handled as an update (`200`), not a conflict. |
| `500` | `Failed to create client.` | Unexpected database/server failure. |

### PATCH /api/webhooks/contacts/{contactId}/onboarding - Update Onboarding

Updates a client's onboarding status and creates an `ONBOARDING_MILESTONE` engagement event with optional metadata.

**Call history side-effect:** Sending `call_booked` or `call_completed` also creates or updates a call record so the client appears in the Calls view - no separate call webhook required for the onboarding call.

- `call_booked` → creates an `ONBOARDING / SCHEDULED` call if one does not already exist.
- `call_completed` → promotes the most recent `ONBOARDING / SCHEDULED` call to `COMPLETED`, or creates a new `ONBOARDING / COMPLETED` call if no scheduled record is found. Idempotent: a second `call_completed` for the same client does nothing if a completed call already exists.

Pass optional call details inside `metadata` to enrich the record:

| Metadata field | Type | Description |
| --- | --- | --- |
| `coach` | string | Coach name stored on the call record. |
| `scheduled_at` | ISO datetime | Scheduled call time (used on `call_booked`). |
| `happened_at` | ISO datetime | Actual call time (used on `call_completed`; defaults to now). |

For richer call records - including Cal.com event IDs, recording URLs, transcripts, and AI summaries - use the dedicated `POST .../calls` and `PATCH /api/webhooks/calls/{callId}` endpoints alongside or instead of the milestone.

```http
PATCH /api/webhooks/contacts/{contactId}/onboarding
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `milestone` | enum | Yes | One of `contract_signed`, `questionnaire_completed`, `call_booked`, `call_completed`. |
| `metadata` | JSON | No | Any structured automation context to store with the milestone event. See call detail fields above. |

Milestone mapping:

| Request value | Stored dashboard value |
| --- | --- |
| `contract_signed` | `CONTRACT_SIGNED` |
| `questionnaire_completed` | `QUESTIONNAIRE_COMPLETED` |
| `call_booked` | `CALL_BOOKED` |
| `call_completed` | `CALL_COMPLETED` |

Example request - booking an onboarding call:

```json
{
  "milestone": "call_booked",
  "metadata": {
    "coach": "Taylor Smith",
    "scheduled_at": "2026-05-10T09:00:00.000Z"
  }
}
```

Example request - completing an onboarding call:

```json
{
  "milestone": "call_completed",
  "metadata": {
    "coach": "Taylor Smith",
    "happened_at": "2026-05-10T09:45:00.000Z"
  }
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "client": {
    "id": "clwclient123",
    "onboardingStatus": "QUESTIONNAIRE_COMPLETED",
    "updatedAt": "2026-05-01T10:15:00.000Z",
    "metadataEvent": {
      "id": "clwevent123",
      "eventType": "ONBOARDING_MILESTONE",
      "happenedAt": "2026-05-01T10:15:00.000Z"
    }
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid `milestone`, invalid `metadata`, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `500` | `Failed to update client onboarding.` | Unexpected database/server failure. |

### PATCH /api/webhooks/contacts/{contactId}/discord - Update Discord IDs

Stores Discord identifiers for a client.

```http
PATCH /api/webhooks/contacts/{contactId}/discord
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `discord_channel_id` | string | Yes | Discord channel ID associated with the client. Must be unique. |
| `discord_user_id` | string | Yes | Discord user ID associated with the client. Must be unique. |

Example request:

```json
{
  "discord_channel_id": "123456789012345678",
  "discord_user_id": "987654321098765432"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "client": {
    "id": "clwclient123",
    "discordChannelId": "123456789012345678",
    "discordUserId": "987654321098765432",
    "updatedAt": "2026-05-01T10:20:00.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/blank fields or unknown fields. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `409` | `Discord identifier already exists.` | The channel ID or user ID is already assigned to another client. |
| `500` | `Failed to update client Discord details.` | Unexpected database/server failure. |

### PATCH /api/webhooks/contacts/{contactId}/track - Update Track

Assigns a client to a track and group.

```http
PATCH /api/webhooks/contacts/{contactId}/track
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `track` | enum | Yes | One of `beginner`, `advanced`. |
| `group_assignment` | enum | Yes | One of `BEGINNER`, `ADVANCED`. |

Track mapping:

| Request value | Stored dashboard value |
| --- | --- |
| `beginner` | `BEGINNER` |
| `advanced` | `ADVANCED` |

Example request:

```json
{
  "track": "beginner",
  "group_assignment": "BEGINNER"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "client": {
    "id": "clwclient123",
    "track": "BEGINNER",
    "groupAssignment": "BEGINNER",
    "updatedAt": "2026-05-01T10:25:00.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid enum values or unknown fields. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `500` | `Failed to update client track.` | Unexpected database/server failure. |

### POST /api/webhooks/contacts/{contactId}/calls - Create Call

Creates a scheduled call record for a client. The call starts with status `SCHEDULED`.

```http
POST /api/webhooks/contacts/{contactId}/calls
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `call_type` | enum | Yes | One of `onboarding`, `one_on_one`, `pod`, `performance_review`, `skill_evaluation`, `data_review`. |
| `coach` | string | No | Coach name. |
| `scheduled_at` | ISO datetime | No | Scheduled call time with timezone. |
| `call_number` | integer | No | Positive call sequence number. |
| `cal_com_event_id` | string | No | External Cal.com event ID. Must be unique if supplied. |

Call type mapping:

| Request value | Stored dashboard value |
| --- | --- |
| `onboarding` | `ONBOARDING` |
| `one_on_one` | `ONE_ON_ONE` |
| `pod` | `POD` |
| `performance_review` | `PERFORMANCE_REVIEW` |
| `skill_evaluation` | `SKILL_EVALUATION` |
| `data_review` | `DATA_REVIEW` |

Example request:

```json
{
  "call_type": "onboarding",
  "coach": "Taylor Smith",
  "scheduled_at": "2026-05-03T09:00:00.000Z",
  "call_number": 1,
  "cal_com_event_id": "cal_evt_123"
}
```

Example success response, HTTP `201`:

```json
{
  "success": true,
  "call": {
    "id": "clwcall123",
    "clientId": "clwclient123",
    "callType": "ONBOARDING",
    "coachName": "Taylor Smith",
    "scheduledTime": "2026-05-03T09:00:00.000Z",
    "callNumber": 1,
    "calComEventId": "cal_evt_123",
    "status": "SCHEDULED",
    "createdAt": "2026-05-01T10:30:00.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid `call_type`, invalid date, non-positive `call_number`, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `409` | `Call already exists.` | Another call already uses `cal_com_event_id`. |
| `500` | `Failed to create call.` | Unexpected database/server failure. |

### PATCH /api/webhooks/calls/{callId} - Update Call

Updates an existing call after it happens or changes status.

```http
PATCH /api/webhooks/calls/{callId}
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `status` | enum | Yes | One of `completed`, `no_show`, `cancelled`, `rebooked`. |
| `happened_at` | ISO datetime | No | Actual call time with timezone. |
| `recording_url` | string | No | Recording URL. |
| `contact_id` | string | No | If set, the call must belong to this client (GHL `contactid` or internal client id). Omit to update by call id only. |

Status mapping:

| Request value | Stored dashboard value |
| --- | --- |
| `completed` | `COMPLETED` |
| `no_show` | `NO_SHOW` |
| `cancelled` | `CANCELLED` |
| `rebooked` | `REBOOKED` |

Example request:

```json
{
  "status": "completed",
  "happened_at": "2026-05-03T09:04:00.000Z",
  "recording_url": "https://example.com/recordings/call-123",
  "contact_id": "zMC7sAfinnBzqYy8n98V"
}
```

Omit `contact_id` when you only have the dashboard `call.id` from the create-call response. Include it when you want the server to reject the update if the call is not tied to that contact.

Example success response, HTTP `200`:

```json
{
  "success": true,
  "call": {
    "id": "clwcall123",
    "clientId": "clwclient123",
    "status": "COMPLETED",
    "actualTime": "2026-05-03T09:04:00.000Z",
    "recordingUrl": "https://example.com/recordings/call-123",
    "updatedAt": "2026-05-03T10:00:00.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid call id.` | `{callId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid `status`, invalid date, blank optional string, or unknown fields. |
| `404` | `Client not found.` | `contact_id` was sent but does not match any client. |
| `404` | `Call not found.` | No call exists for `{callId}` (when `contact_id` is omitted). |
| `404` | `Call not found or does not belong to the specified client.` | `contact_id` is set but no call matches both `{callId}` and that client. |
| `500` | `Failed to update call.` | Unexpected database/server failure. |

### POST /api/webhooks/contacts/{contactId}/payments - Create Payment

Records a backend payment and recalculates the client's remaining balance.

**Sale-seed merging:** a sale webhook (`POST /api/webhooks/clients`) with `cash_collected` seeds a ledger payment for that sale. When this endpoint later receives a payment with exactly the same cash within 7 days of an unclaimed seed, the delivery is treated as the back-end recording of that same money: the seed is updated in place (date, closer, notes, external id) instead of a duplicate being created, and the response is HTTP `200` (not `201`) with `"merged_into_sale_seed": true`. Each seed can be claimed at most once; a later payment with the same cash creates a new ledger row as normal. Deliveries that hit different sale seeds, or arrive outside the window, behave exactly as before. Concurrent delivery of the sale webhook and the matching payment webhook (within the same second) can bypass the merge and record the money twice; sequence the two calls in the sender when both are emitted.

```http
POST /api/webhooks/contacts/{contactId}/payments
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `external_payment_id` | string | No | Stable upstream payment id used to reject duplicate payment deliveries. Must not start with `sale:` (reserved for sale-seed payments); such values are rejected with `400`. |
| `amount` | money | Yes | Payment amount received. |
| `payment_date` | ISO datetime | Yes | Payment timestamp with timezone. Must not be in the future (a 24-hour grace window absorbs timezone differences); future dates are rejected with `400`. |
| `cash_collected` | money | No | Cash actually received on this payment (after fees). Drives the client's balance and payment status. |
| `closer` | enum | No | Closer credited for the payment. One of `Sam`, `Dan`, `Hunter`, `Phillip`. |
| `notes` | string | No | Payment notes. |

Balance behavior (recalculated from the full payment ledger on every delivery):

- The client's `cashCollected` becomes the sum of `cash_collected` across all of the client's payments.
- If the client has `contractValue`, remaining balance becomes `contractValue - sum(cash_collected)`, clamped to `0`. When no payment on the ledger has a `cash_collected` value, the summed `amount` is used instead.
- `paymentStatus` is derived from the same math: `OK` when the remaining balance reaches `0`, otherwise `INCOMPLETE`.
- If the client does not have `contractValue`, remaining balance becomes current `remainingBalance - (cash_collected or amount)`, clamped to `0`.

Example request:

```json
{
  "external_payment_id": "stripe_pi_123",
  "amount": "1000.00",
  "payment_date": "2026-05-05T12:00:00.000Z",
  "cash_collected": "2000.00",
  "closer": "Sam",
  "notes": "Second installment paid by card."
}
```

Example success response, HTTP `201`:

```json
{
  "success": true,
  "payment": {
    "payment": {
      "id": "clwpayment123",
      "clientId": "clwclient123",
      "amount": "1000",
      "paymentDate": "2026-05-05T12:00:00.000Z",
      "cashCollected": "2000",
      "closer": "Sam",
      "notes": "Second installment paid by card.",
      "createdAt": "2026-05-05T12:01:00.000Z"
    },
    "client": {
      "id": "clwclient123",
      "remainingBalance": "3000.00",
      "cashCollected": "2000"
    }
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing `amount` or `payment_date`, invalid money/date format, a `payment_date` in the future, blank optional string, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `409` | `Duplicate payment webhook.` | The same `Idempotency-Key` or `external_payment_id` was already processed. |
| `500` | `Failed to create payment.` | Unexpected database/server failure. |

### POST /api/webhooks/b2b/clients - Create B2B Company

Creates a B2B company from the onboarding form, optionally invites company-owner emails as company-scoped dashboard users, and optionally pre-registers its sales reps in the same call. Once a rep is registered, their daily submissions to the B2B EOD endpoint resolve immediately by this company's `contactid` plus their email. Unlike the B2C sale webhook (`POST /api/webhooks/clients`), this seeds no payment, runs no balance reconcile, and fires no coaching workflows: a B2B company's revenue is rep activity reporting, not the payment ledger.

```http
POST /api/webhooks/b2b/clients
```

Behavior:

- Idempotent on `contactid`: if a client already exists for that CRM contact id (or internal id) and is already B2B, it is refreshed (`200`); otherwise a new B2B client is created (`201`). A resubmission with the same reps refreshes the roster rather than erroring.
- The new client is created with `clientType = B2B`, `onboardingStatus = CONTRACT_SIGNED`, and neutral defaults for the coaching fields (`track`/`groupAssignment = UNASSIGNED`, `healthStatus = GREEN`, `paymentStatus = OK`).
- Each rep in `reps` is matched by email within this company: an email already registered to *this* company is reactivated (idempotent re-submit); any other email is created as a **new** rep row under this company, even if that email already belongs to a different company. Rep email is unique per company, not globally, so the same person can be a rep at more than one B2B company (one `SalesRep` row per company, same email); registering them elsewhere never moves or overwrites their other company's rep row or historical numbers. The company write and all rep writes happen in one transaction, so a conflict rolls the whole call back. `repsRegistered` in the response counts newly created reps.
- On **first creation** of a company, app invites ("set your password" links) are sent to the primary `email` and every address in `ownerEmails` / `owner_emails` automatically, never to a rep's email. Invite links expire after one hour. Repeat deliveries (same `contactid`) update the company and never re-invite, and if an owner email is already a login no second invite is sent. Invites are best-effort and do not change the response: creation still returns `201` even if an invite email fails. Disable globally with `B2B_AUTO_INVITE=off`.

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `contactid` | string | Yes | CRM contact id for the company. Used to de-duplicate and to link later EOD/rep data. |
| `name` | string | Yes | Company name. If your automation can only map the CRM contact's personal name here, send the business name in `company_name` and it takes precedence. |
| `company_name` / `business_name` | string | No | Explicit business name. When present, it overrides `name` as the stored company name (`company_name` wins if both are sent). Send this when the CRM maps a contact person into `name`. Repeat deliveries with this field also correct an existing company's name. If sent, it must be non-empty after trimming: a blank value is a `400`, not a fallback to `name`, so omit the key when the CRM field is empty. |
| `email` | string | No | Primary business owner / company email (lowercased). Stored on the company and must be unique across clients. Also receives an app invite on first creation. |
| `ownerEmails` / `owner_emails` | array or string | No | Additional company-owner login emails to invite on first creation. Up to 100. Accepts an array of emails or one comma/semicolon/newline-separated string. These emails are lowercased, deduped with `email`, and are not stored on the company row. |
| `phone` | string | No | Contact phone. |
| `notes` | string | No | Internal context notes. |
| `reps` | array | No | Sales reps to pre-register. Up to 100. Each item requires only `email` (lowercased; combined with this company's `contactid`, it is the key the EOD endpoint resolves on); `name` is optional and defaults to the email's local part when omitted. So `{ "email": string }` or `{ "name": string, "email": string }`. |

Example request:

```json
{
  "contactid": "zMC7sAfinnBzqYy8n98V",
  "name": "Jamie Rivera",
  "company_name": "Acme Corp",
  "email": "owner@acme.com",
  "ownerEmails": ["finance@acme.com", "ops@acme.com", "ceo@acme.com"],
  "reps": [
    { "name": "Jane Smith", "email": "jane@acme.com" },
    { "email": "raj@acme.com" }
  ]
}
```

Example success response, HTTP `201`:

```json
{
  "success": true,
  "client": {
    "id": "clwclient123",
    "contactId": "zMC7sAfinnBzqYy8n98V",
    "name": "Acme Corp",
    "email": "owner@acme.com",
    "clientType": "B2B",
    "createdAt": "2026-07-02T12:00:00.000Z",
    "repsRegistered": 2
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `contactid` or `name`, an invalid `email`, or a `reps` entry missing its email / with an invalid email / beyond 100 items. Unknown extra fields are ignored, not rejected. |
| `409` | `A non-B2B client already exists for this contact id. Convert it from the client page if this is intended.` | A client already exists for that `contactid` but is not B2B. It is not silently converted; use the in-dashboard client-type toggle instead. |
| `409` | `A client with this email already exists.` | The company `email` is already used by a different client. |
| `409` | `A client with this contact id already exists.` | A concurrent duplicate create raced for a new `contactid`. |
| `409` | `One of the rep emails is already registered under this company.` | A rep email hit this company's unique (`clientId`, `email`) index during a concurrent write. A rep email that already belongs to a *different* company is not a conflict; it creates a second rep row for this company. |
| `500` | `Failed to create B2B client.` | Unexpected database/server failure. |

### POST /api/webhooks/b2b/eod - Record B2B EOD

Records one B2B sales rep's end-of-day numbers for a single day. This is the target for the daily EOD form each B2B client's sales reps fill out (wire your form tool or automation to POST here). The rep is resolved by the company's `contactid` plus their email, so the form needs the company's `contactid`, the rep's email, and the day's numbers. It is activity reporting only: `revenue_generated` is **not** part of the client payment ledger and never affects `contractValue`, `cashCollected`, or `remainingBalance`.

**Breaking change:** `contactid` is now a **required** field on this endpoint. Rep email is unique per B2B company, not globally (the same person can be a rep at more than one company), so a submission that identifies the rep by email alone is no longer enough to resolve which company's rep to credit. Form tools and automations that only send `rep_email` must be updated to also send the company's `contactid`, or submissions will be rejected with `400`.

```http
POST /api/webhooks/b2b/eod
```

Behavior:

- The rep must be **pre-registered** under a B2B company in the dashboard (name + email). The company is resolved first by `contactid` (the CRM `contactid` or the dashboard's internal client id); then `rep_email` is matched to a rep under that company (case-insensitive). An unknown `contactid`, or an email with no matching (or deactivated) rep under that company, is rejected with `404` so numbers are never attributed to the wrong company or silently dropped.
- Submissions are upserted by (rep, date): re-posting the same rep and `date` overwrites that day's numbers rather than duplicating them, so retries and corrections are safe. No `Idempotency-Key` header is needed.

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `contactid` | string | Yes | CRM contact id of the company the rep is reporting for. Accepts the GHL contactid or the dashboard's internal client id. |
| `rep_email` | string | Yes | The rep's registered email, scoped to the company identified by `contactid` (email is unique per company, not globally). |
| `date` | date | Yes | The EOD calendar date, `YYYY-MM-DD`. Must not be a future date (rejected with `400`). |
| `calls_made` | integer | Yes | Calls made / outreach. Non-negative whole number. |
| `conversations` | integer | Yes | Conversations / pickups. Non-negative whole number. |
| `appointments_booked` | integer | Yes | Appointments booked. Non-negative whole number. |
| `shows` | integer | Yes | Number of shows. Non-negative whole number. |
| `closes` | integer | Yes | Number of closes. Non-negative whole number. |
| `revenue_generated` | money | Yes | Revenue generated (dollars). Non-negative. Activity reporting only, not the payment ledger. |
| `opportunities_pipeline` | integer | Yes | Opportunities in pipeline (count). Non-negative whole number. |

Example request:

```json
{
  "contactid": "zMC7sAfinnBzqYy8n98V",
  "rep_email": "jane@acme.com",
  "date": "2026-07-01",
  "calls_made": 40,
  "conversations": 12,
  "appointments_booked": 5,
  "shows": 3,
  "closes": 2,
  "revenue_generated": "4500.00",
  "opportunities_pipeline": 8
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "eodSubmission": {
    "id": "clweod123",
    "clientId": "clwclient123",
    "salesRepId": "clwrep123",
    "submissionDate": "2026-07-01T00:00:00.000Z",
    "callsMade": 40,
    "conversations": 12,
    "appointmentsBooked": 5,
    "shows": 3,
    "closes": 2,
    "revenueGenerated": "4500",
    "opportunitiesPipeline": 8
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `contactid`, a required metric, or `rep_email`, an invalid email, a negative or non-integer count, negative or out-of-range revenue, or a malformed / impossible `date`. Unknown extra fields are ignored, not rejected. |
| `400` | `EOD date cannot be in the future.` | `date` is a later calendar day (UTC) than today. |
| `404` | `Company not found for that contactid.` | No B2B client matches `contactid` (by internal id or CRM contact id). |
| `404` | `Rep not found. Register this rep under this B2B company first.` | The company was found, but no active rep under it matches `rep_email`. Since email is unique per company (not globally), the same email may exist as a rep at a different company without matching here. |
| `500` | `Failed to record EOD submission.` | Unexpected database/server failure. |

### POST /api/webhooks/sales/tag-events - Record Sales Tag Event

Records one application of a CRM tag to a sales-funnel contact, with the moment it was applied (the GHL API cannot report when a tag was applied, so this webhook, fired from a GHL workflow, is the only tag-history record). This is the raw material for the future sales funnel dashboard's metrics and 30-day first-touch attribution. A sales-funnel contact (`SalesContact`) is distinct from a `Client`: it represents a lead in the acquisition funnel, not a closed, onboarded customer.

If the contact identified by `contact_id` does not exist yet, it is created automatically (`first_seen_at` is set to `applied_at`); an existing contact is enriched with any newly-provided `email`, `phone`, or `name` fields, but fields already on file are never overwritten by a webhook value.

**Idempotency:** re-posting the same tag event (same contact, tag, and `applied_at`) returns HTTP `200` with `{ "tagEvent": { "duplicate": true } }`, not an error. GHL workflows re-fire on reschedules by design, and duplicate deliveries must never look like a failure to the automations side.

```http
POST /api/webhooks/sales/tag-events
```

Headers:

```http
x-api-key: <API_KEY>
Content-Type: application/json
Accept: application/json
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `contact_id` | string | Yes | CRM (GHL) contact id. |
| `tag` | string | Yes | The tag exactly as applied in the CRM, max 200 characters. Stored verbatim; also matched case-and-whitespace-insensitively for metrics. |
| `applied_at` | ISO datetime | Yes | The moment the tag was applied, with a timezone offset. Rejected with `400` if more than 24 hours in the future (clock-skew guard) or earlier than 2020-01-01 (implausibly-old guard; a real automation tag event cannot predate this system). |
| `email` | string | No | Contact email. Only used to enrich the contact if it does not already have one on file. Loosely validated (must contain `@`, max 320 characters); an invalid value is silently **dropped** (treated as absent) rather than rejecting the whole event, so a malformed email from an upstream tool never blocks the tag event or gets stuck on the contact. |
| `phone` | string | No | Contact phone. Only used to enrich the contact if it does not already have one on file. An empty or whitespace-only value is silently **dropped** (treated as absent) rather than rejecting the event; junk values too short to be a real number are also dropped during ingestion. |
| `name` | string | No | Contact name, max 300 characters. Only used to enrich the contact if it does not already have one on file. |

Unknown extra fields are rejected with `400` (this route has no lenient exception).

Example request:

```json
{
  "contact_id": "ghl-abc-123",
  "tag": "Offer Placement",
  "applied_at": "2026-08-04T14:00:00.000Z",
  "email": "jane@example.com",
  "phone": "+15551234567",
  "name": "Jane Doe"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "tagEvent": {
    "id": "clwtag123"
  }
}
```

Example duplicate-delivery response, HTTP `200`:

```json
{
  "success": true,
  "tagEvent": {
    "duplicate": true
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `contact_id`, `tag`, or `applied_at`, a malformed `applied_at`, or an unknown field. |
| `400` | `applied_at cannot be more than 24 hours in the future.` | `applied_at` is more than 24 hours ahead of the server's clock. |
| `400` | `Timestamp cannot be earlier than 2020-01-01.` | `applied_at` predates 2020-01-01, which can only be an epoch-zero or unit-mismatch bug in the sending automation. |
| `500` | `Failed to record tag event.` | Unexpected database/server failure. |

### Linking leads to clients

The sales endpoints and the client endpoints write to two separate records for the same person: a `SalesContact` (a lead in the acquisition funnel, most of whom never buy) and a `Client` (a closed, onboarded customer). When a lead converts, the two are linked automatically so the sale can be attributed back to the funnel path that produced it.

**The link is made on the CRM contact id, and nothing else.** `POST /api/webhooks/sales/contacts` sends it as `contact_id`; `POST /api/webhooks/clients` sends it as `contactid`. When both records carry the same value, they are linked.

What this means for a sending automation:

- **Always send the CRM contact id on both routes, and make sure it is the same value.** A client created without `contactid` can never be linked automatically.
- **Order does not matter.** Whichever record arrives second completes the link, so a lead who converts before their contact webhook fires (or the reverse) still links up. There is no batch job to wait for.
- **Repeat deliveries are safe.** Linking re-runs on every client create and update and every sales-contact upsert; an existing link is never repointed and no duplicate is created.
- **Matching is never fuzzy.** Clients whose email or phone merely resembles a lead are not linked automatically, because a wrong link misattributes a sale invisibly. Those are surfaced for a human to confirm.
- **Link failures never fail your request.** Linking happens after the client or contact is committed, so a `2xx` means your data was written even if the link did not happen. A client webhook retries the link on its next delivery; the sales-contacts route is usually one-shot per lead, so there a failed link is recovered by the backfill script rather than by a later call. Failures and conflicts (the two records share an id but one is already linked elsewhere) are recorded for a human in Admin, Activity, Errors.

### POST /api/webhooks/sales/contacts - Create Sales Contact

Records a sales-funnel contact creation from the CRM (GHL), so a `SalesContact` row exists (and its `first_seen_at` reflects the CRM's own `created_at`, when the workflow sends it) even before any tag has been applied. If the contact already exists, this call enriches it the same way `POST /api/webhooks/sales/tag-events` does: only fields not already on file are filled in, and `first_seen_at` never moves later.

**Send the same contact id you send as `contactid` on `POST /api/webhooks/clients`.** That shared id is the only thing that links a lead to the client they become when they buy (see [Linking leads to clients](#linking-leads-to-clients)). If the two workflows send different ids, the lead and the client stay unconnected and the sale cannot be attributed to a funnel path.

```http
POST /api/webhooks/sales/contacts
```

Headers:

```http
x-api-key: <API_KEY>
Content-Type: application/json
Accept: application/json
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `contact_id` | string | Yes | CRM (GHL) contact id. |
| `email` | string | No | Contact email. Loosely validated (must contain `@`, max 320 characters); an invalid value is silently **dropped** (treated as absent) rather than rejecting the whole event. |
| `phone` | string | No | Contact phone. An empty or whitespace-only value is silently **dropped** (treated as absent) rather than rejecting the event; junk values too short to be a real number are also dropped during ingestion. |
| `name` | string | No | Contact name, max 300 characters. |
| `created_at` | ISO datetime | No | The CRM's own contact-creation timestamp, with a timezone offset. When omitted, `first_seen_at` is set to the time this webhook was received. When present, rejected with `400` if more than 24 hours in the future (clock-skew guard) or earlier than 2020-01-01 (implausibly-old guard). |

Unknown extra fields are rejected with `400` (this route has no lenient exception).

Example request:

```json
{
  "contact_id": "ghl-abc-123",
  "email": "jane@example.com",
  "phone": "+15551234567",
  "name": "Jane Doe",
  "created_at": "2026-08-04T10:00:00.000Z"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "salesContact": {
    "id": "clwcontact123",
    "created": true
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `contact_id`, a malformed `created_at`, or an unknown field. |
| `400` | `created_at cannot be more than 24 hours in the future.` | `created_at` is more than 24 hours ahead of the server's clock. |
| `400` | `Timestamp cannot be earlier than 2020-01-01.` | `created_at` predates 2020-01-01, which can only be an epoch-zero or unit-mismatch bug in the sending automation. |
| `500` | `Failed to record sales contact.` | Unexpected database/server failure. |

### POST /api/webhooks/sales/dialer-calls - Record Dialer Call

Records one dialer call event, pushed per completed dial. These rows feed the Outbound view's activity metrics (dials, pickups, and conversations over 3 minutes). Send `contact_id` when the sender can resolve the GHL contact, which additionally enables Sets booked and Convo to set; those stay pending while no dial carries one. No contact needs to exist before a dial is recorded.

**Idempotency:** `call_id` is the dedupe key; when omitted, one is synthesized from the normalized phone and `occurred_at`, so re-posting the identical event returns HTTP `200` with `{ "dialerCall": { "duplicate": true } }`. Two genuinely distinct dials to the same number at the same second would collapse into one row when `call_id` is omitted; send `call_id` whenever the dialer exposes one.

```http
POST /api/webhooks/sales/dialer-calls
```

Headers:

```http
x-api-key: <API_KEY>
Content-Type: application/json
Accept: application/json
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `phone` | string | Yes | The dialed number, max 50 characters. A number too short to normalize (fewer than 7 digits) is **skipped** with HTTP `200` and `{ "dialerCall": { "skipped": "unparseable_phone" } }`, not rejected, since a retry would never improve it. |
| `occurred_at` | ISO datetime | Yes | When the dial happened, with a timezone offset. Rejected with `400` if more than 24 hours in the future or earlier than 2020-01-01. |
| `call_id` | string | No | The dialer's own call id, max 200 characters; the idempotency key. Synthesized from phone + `occurred_at` when omitted. |
| `contact_id` | string | No | The GHL contact this dial reached, when the sender can resolve it (the dialer itself has no contact reference). Max 200 characters; a blank string is treated as absent. This is what makes Sets booked and Convo to set computable: without it those stay pending rather than being guessed from the phone number, which could credit outbound for an inbound sale. Cold-list numbers that match no contact are expected and fine to omit. |
| `picked_up` | boolean | No | Whether the call was answered. When omitted, inferred as `true` iff `duration_seconds` is greater than 0, so send it explicitly if the dialer reports voicemail durations. |
| `duration_seconds` | integer or string | No | Conversation length in seconds, 0 to 86400. Defaults to 0. A numeric string is also accepted. As a targeted rescue of a known sender bug, a value of the form `"NaN<disposition label>"` (e.g. `"NaNNo Answer"`) is treated as `duration_seconds: 0` and, if `disposition` was not sent, the trailing label (truncated to 200 characters) is used as `disposition`. Any other non-numeric value is still rejected with `400`. |
| `disposition` | string | No | The dialer disposition verbatim (e.g. `call booked`), max 200 characters. An empty string is treated as absent. |
| `rep` | string | No | The calling rep, max 200 characters. An empty string is treated as absent. |

Unknown extra fields are rejected with `400` (this route has no lenient exception).

Example request:

```json
{
  "call_id": "dial-789",
  "phone": "+15551234567",
  "contact_id": "ghl-abc-123",
  "occurred_at": "2026-08-06T15:04:05.000Z",
  "picked_up": true,
  "duration_seconds": 245,
  "disposition": "call booked",
  "rep": "phillip@impactteam.us"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "dialerCall": {
    "id": "clwdial123"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `phone` or `occurred_at`, a malformed value, or an unknown field. |
| `400` | `occurred_at cannot be more than 24 hours in the future.` | `occurred_at` is more than 24 hours ahead of the server's clock. |
| `400` | `Timestamp cannot be earlier than 2020-01-01.` | `occurred_at` predates 2020-01-01. |
| `500` | `Failed to record dialer call.` | Unexpected database/server failure. |

### POST /api/webhooks/sales/attendance - Record Call Attendance

Records attendance for one implementation or strategy call, pushed after Fathom/Google Meet report the call. The Showed/No Show verdict is computed server-side from `contact_present_seconds` (at least 2 minutes of presence scores Showed); the sender pushes raw presence data, never a judgment. A call with no presence data at all is stored `UNSCORED`, which is excluded from show rates rather than counted as a no-show.

**Idempotency:** `external_id` (the Fathom recording id, or the sender's own id) is the upsert key: a replay or corrected push updates the same row. A replay **without** `contact_present_seconds` never downgrades a previously scored row back to `UNSCORED`.

```http
POST /api/webhooks/sales/attendance
```

Headers:

```http
x-api-key: <API_KEY>
Content-Type: application/json
Accept: application/json
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `external_id` | string | Yes | Stable id for this call's recording (Fathom recording id preferred). The idempotency/upsert key. |
| `call_type` | string | Yes | `imp` or `strategy`. Case-insensitive; a trailing " call" is stripped and `implementation` maps to `imp`, so `"Strategy Call"` and `"implementation call"` are accepted. Anything else is a `400`. |
| `scheduled_at` | ISO datetime | Yes | The appointment's scheduled start, with a timezone offset. Rejected with `400` if more than 24 hours in the future or earlier than 2020-01-01. |
| `contact_id` | string | No | CRM (GHL) contact id of the booked contact, max 200 characters. An unknown id stores the row unlinked rather than rejecting it. |
| `contact_present_seconds` | integer | No | Total seconds the CONTACT (not the rep/host) was present, 0 to 86400. Omit entirely when the call was not recorded; sending `0` means the contact never joined (a No Show). |
| `rep` | string | No | The rep who took the call, max 200 characters. Used for the per-rep unscored count. An empty string is treated as absent. |
| `recording_url` | string | No | Link to the recording, http(s) only, max 2000 characters. Kept as evidence when a rep disputes a no-show. |

Unknown extra fields are rejected with `400` (this route has no lenient exception).

Example request:

```json
{
  "external_id": "fathom-170593705",
  "contact_id": "ghl-abc-123",
  "call_type": "strategy",
  "rep": "phillip@impactteam.us",
  "scheduled_at": "2026-08-06T15:00:00.000Z",
  "contact_present_seconds": 1620,
  "recording_url": "https://fathom.video/calls/12345"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "attendance": {
    "id": "clwatt123",
    "verdict": "SHOWED",
    "contactMatched": true
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `external_id`, `call_type`, or `scheduled_at`, a malformed value, a non-http(s) `recording_url`, or an unknown field. |
| `400` | `scheduled_at cannot be more than 24 hours in the future.` | `scheduled_at` is more than 24 hours ahead of the server's clock. |
| `400` | `Timestamp cannot be earlier than 2020-01-01.` | `scheduled_at` predates 2020-01-01. |
| `500` | `Failed to record call attendance.` | Unexpected database/server failure. |

### POST /api/webhooks/sales/ad-spend - Record Weekly Ad Spend

Records one weekly ad-spend figure from the client's Funnels & Ads sheet (updated Mondays by their ads team). Spend is weekly only, by design: the dashboard never splits a weekly figure into synthetic daily values, and `week_start` must be the Monday (UTC) the week begins so a week-boundary drift between the sheet and the dashboard cannot be introduced silently.

**Idempotency:** upserted on (`week_start`, `source`): re-posting a corrected figure for the same week replaces the stored value rather than duplicating it.

```http
POST /api/webhooks/sales/ad-spend
```

Headers:

```http
x-api-key: <API_KEY>
Content-Type: application/json
Accept: application/json
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `week_start` | ISO date | Yes | The Monday the spend week begins, as a date only (`YYYY-MM-DD`), interpreted as UTC. Rejected with `400` if it is not a Monday, is more than 8 days in the future, or predates 2020-01-01. |
| `spend` | number | Yes | Total spend for the week in dollars, 0 to 10,000,000. Stored as integer cents. |
| `source` | string | No | Platform/campaign identifier when the sheet carries one, max 100 characters. Defaults to `all`. |

Unknown extra fields are rejected with `400` (this route has no lenient exception).

Example request:

```json
{
  "week_start": "2026-08-03",
  "spend": 4250.75,
  "source": "all"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "adSpendWeek": {
    "id": "clwspend123",
    "spendCents": 425075
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `week_start` or `spend`, a malformed value, or an unknown field. |
| `400` | `week_start must be a Monday (UTC).` | `week_start` is any other weekday. |
| `400` | `week_start cannot be more than 8 days in the future.` | Future-typo guard. |
| `400` | `Timestamp cannot be earlier than 2020-01-01.` | `week_start` predates 2020-01-01. |
| `500` | `Failed to record ad spend.` | Unexpected database/server failure. |

### POST /api/webhooks/sales/cash - Record Cash Collected

Records one cash-collection event (a closed sale / payment collected) from the A1 mastersheet. Feeds units sold, average cash per unit, and GHL pathway cash attribution on the Sales dashboard. The canonical Cash collected hero uses the client Payment ledger instead. Distinct from `POST /api/webhooks/contacts/{contactId}/payments`: these are funnel-level sales records that may precede (or never become) an onboarded Client.

**Idempotency:** `external_id` (sheet row id or payment id) is the dedupe key; when omitted, one is synthesized from the identifying fields + `collected_at` + amount + units + closer, so a re-pushed row returns HTTP `200` with `{ "cashEvent": { "duplicate": true } }` instead of double-counting revenue. Caveat: without `external_id`, two genuinely distinct sales that share every synthesized field (same contact, timestamp, amount, units, closer) collapse into one row and the second is silently swallowed, undercounting revenue. This matters especially if `collected_at` only carries day granularity. Always send `external_id` when the sheet has a row id.

```http
POST /api/webhooks/sales/cash
```

Headers:

```http
x-api-key: <API_KEY>
Content-Type: application/json
Accept: application/json
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | number | Yes | Cash collected in dollars, 0 to 10,000,000. Stored as integer cents. |
| `collected_at` | ISO datetime | Yes | When the cash was collected, with a timezone offset. Rejected with `400` if more than 24 hours in the future or earlier than 2020-01-01. |
| `external_id` | string | No | Stable id for this sale (sheet row id / payment id), max 200 characters. Strongly recommended; the dedupe key. |
| `contact_id` | string | No | CRM (GHL) contact id, max 200 characters. First choice for linking the sale to a funnel contact. |
| `email` | string | No | Fallback contact match. Loosely validated; an invalid value is silently dropped. |
| `phone` | string | No | Second fallback contact match. An empty value is silently dropped. |
| `units` | integer | No | Units sold in this event, 1 to 1000. Defaults to 1. |
| `closer` | string | No | The closing rep, max 200 characters. An empty string is treated as absent. |

An unmatched sale (no contact found by any identifier) is still stored and counts in the top-line total; the response reports `contactMatched: false`.

Unknown extra fields are rejected with `400` (this route has no lenient exception).

Example request:

```json
{
  "external_id": "a1-row-1042",
  "contact_id": "ghl-abc-123",
  "amount": 3000,
  "units": 1,
  "closer": "phillip@impactteam.us",
  "collected_at": "2026-08-06T18:30:00.000Z"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "cashEvent": {
    "id": "clwcash123",
    "contactMatched": true
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `amount` or `collected_at`, a malformed value, or an unknown field. |
| `400` | `collected_at cannot be more than 24 hours in the future.` | `collected_at` is more than 24 hours ahead of the server's clock. |
| `400` | `Timestamp cannot be earlier than 2020-01-01.` | `collected_at` predates 2020-01-01. |
| `500` | `Failed to record cash event.` | Unexpected database/server failure. |

### POST /api/webhooks/sales/slot-utilization - Record Slot Utilization

Records one day's booked/available slot counts for one sales calendar (daily slot utilization = booked / total availability on GHL calendars). Push once per calendar per day; a same-day re-push (counts changing as bookings come in) replaces the stored figures.

Send `calendar_group` (`Imp Calls` or `Strategy Calls`) so the dashboard can report implementation and strategy capacity separately. The dashboard falls back to matching call-type words in the calendar name when no group is sent, which does not work for the real calendar names in use, so a row without a group is reported as unclassified capacity rather than being guessed into a bucket.

```http
POST /api/webhooks/sales/slot-utilization
```

Headers:

```http
x-api-key: <API_KEY>
Content-Type: application/json
Accept: application/json
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `date` | ISO date | Yes | The day the counts describe (`YYYY-MM-DD`, UTC). Rejected with `400` if earlier than 2020-01-01 or more than 62 days in the future. |
| `calendar` | string | Yes | Calendar identifier (e.g. `Gameplan Call - Patrick` or a rep's calendar name), max 200 characters. It is the grouping key, so keep it stable per calendar. |
| `calendar_group` | string | No | The GHL calendar **group** the calendar sits in: `Imp Calls` or `Strategy Calls`, max 200 characters. This is what splits implementation capacity from strategy capacity on the dashboard. Individual calendar names vary per lead source and per rep and carry no call-type information, so without this the calendar shows as unclassified and counts toward neither utilization figure. A blank string is treated as absent, and omitting it on a re-push leaves any previously stored group untouched. |
| `slots_total` | integer | Yes | Total available slots that day, 0 to 10,000. |
| `slots_booked` | integer | Yes | Booked slots that day, 0 to 10,000. Must not exceed `slots_total`. |

Unknown extra fields are rejected with `400` (this route has no lenient exception).

Example request:

```json
{
  "date": "2026-08-06",
  "calendar": "Gameplan Call - Patrick",
  "calendar_group": "Strategy Calls",
  "slots_total": 16,
  "slots_booked": 11
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "slotDay": {
    "id": "clwslot123"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing or malformed fields, `slots_booked` greater than `slots_total`, or an unknown field. |
| `400` | `date cannot be more than 62 days in the future.` | Future-typo guard (availability may be pushed ahead, but not months out). |
| `400` | `Timestamp cannot be earlier than 2020-01-01.` | `date` predates 2020-01-01. |
| `500` | `Failed to record slot utilization.` | Unexpected database/server failure. |

### POST /api/webhooks/contacts/{contactId}/engagement - Create Engagement Event

Logs a client activity event.

```http
POST /api/webhooks/contacts/{contactId}/engagement
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `event_type` | enum | Yes | One of the supported engagement event values below. |
| `event_id` | string | No | Stable upstream event id used to reject duplicate event deliveries. |
| `event_date` | ISO datetime | Yes | Event timestamp with timezone. |
| `metadata` | JSON | No | Structured event details. |

Supported `event_type` values:

| Request value | Stored dashboard value |
| --- | --- |
| `discord_message` | `DISCORD_MESSAGE` |
| `group_call_attended` | `GROUP_CALL_ATTENDED` |
| `pod_call_attended` | `POD_CALL_ATTENDED` |
| `one_on_one_attended` | `ONE_ON_ONE_ATTENDED` |
| `module_completed` | `MODULE_COMPLETED` |
| `todo_completed` | `TODO_COMPLETED` |
| `todo_incomplete` | `TODO_INCOMPLETE` |
| `survey_completed` | `SURVEY_COMPLETED` |
| `recording_watched` | `RECORDING_WATCHED` |

Example request:

```json
{
  "event_type": "module_completed",
  "event_id": "lms_event_123",
  "event_date": "2026-05-06T14:00:00.000Z",
  "metadata": {
    "module": "Discovery Calls",
    "lesson_id": "lesson_42"
  }
}
```

Example success response, HTTP `201`:

```json
{
  "success": true,
  "event": {
    "id": "clwevent456",
    "clientId": "clwclient123",
    "eventType": "MODULE_COMPLETED",
    "happenedAt": "2026-05-06T14:00:00.000Z",
    "details": {
      "module": "Discovery Calls",
      "lesson_id": "lesson_42"
    },
    "createdAt": "2026-05-06T14:00:02.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid `event_type`, invalid `event_date`, invalid metadata, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `409` | `Duplicate engagement webhook.` | The same `Idempotency-Key` or `event_id` was already processed. |
| `500` | `Failed to create engagement event.` | Unexpected database/server failure. |

### POST /api/webhooks/contacts/{contactId}/placement - Update Placement

Syncs an active individual client's placement stage from a monthly survey. The
survey response id is required and idempotent. A delayed survey whose
`submitted_at` predates the client's latest placement update is accepted but
reported as `stale`; it does not overwrite newer dashboard data.

```json
{
  "stage": "applying_to_offers",
  "survey_response_id": "monthly-survey-123",
  "submitted_at": "2026-07-31T10:30:00.000Z",
  "loom_video_url": "https://www.loom.com/share/abc123"
}
```

`loom_video_url` is optional: the client's most recent Loom application video, collected on the monthly survey. Webhook-wins: each survey submission that carries it refreshes the stored link, which is shown on the client's profile in the dashboard. Must be http(s); any other scheme is rejected with `400`.

Supported `stage` values. Each stage accepts either the canonical snake_case
key or the verbatim monthly-survey answer text, so the automation can forward
the raw survey answer with no mapping step. Matching is case-insensitive and
exact after trimming whitespace; any other wording (including a drifted survey
answer) returns HTTP `400` rather than guessing a stage.

| Request value | Survey answer alias | Dashboard label |
| --- | --- | --- |
| `not_on_offer` | `Taking a break from sales` | Not on Offer |
| `applying_to_offers` | `Looking for an offer` | Applying to Offers |
| `on_offer_happy` | `Happy on this offer` | On an Offer Happy |
| `on_offer_wants_better_one` | `Looking for a better offer` | On an Offer Wants a Better One |
| `on_offer_wants_to_get_better` | `Not making as much as I could be on this offer` | On an Offer Wants To Get Better |

Any other value returns HTTP `400`. Clients start in a sixth stage, **No
Stage**, which means nobody has tagged them yet; it is set by the dashboard
only, so this webhook cannot move a client back to it.

A successful response contains `placement.status` as `changed`, `unchanged`, or
`stale`.

| Status | Error | When |
| --- | --- | --- |
| `400` | `Invalid placement stage. Use one of: ...` | `stage` is not one of the five values above. |
| `400` | `submitted_at cannot be more than five minutes in the future.` | Clock skew beyond the allowed window. |
| `400` | `Invalid contact id.` | `{contactId}` is blank. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `404` | `Placement client not found.` | The client exists but is a B2B company or archived. |
| `409` | `Duplicate placement survey.` | The same `survey_response_id` was already processed. |
| `500` | `Failed to update placement.` | Unexpected database/server failure. |

### POST /api/webhooks/contacts/{contactId}/dev-docs - Create Development Document

Saves an AI-generated or automation-generated development document for a client. If `call_id` is included, the call must belong to the same client.

```http
POST /api/webhooks/contacts/{contactId}/dev-docs
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `week_label` | string | Yes | Week label, for example `Week 1` or `2026-W18`. |
| `call_id` | string | No | Dashboard call ID to link to this document. A call can have only one development document. |
| `income_goal` | string | No | Income goal text. |
| `weeks_focus` | JSON | No | Structured focus areas for the week. |
| `weeks_outcome` | JSON | No | Structured outcome or result data. |
| `how` | JSON | No | Structured implementation plan or recommendations. |
| `development_doc_url` | URL | No | Link to the client's living Development Doc. When present, updates the client profile link while creating this note snapshot. |

Example request:

```json
{
  "week_label": "Week 1",
  "call_id": "clwcall123",
  "income_goal": "$10k/month",
  "weeks_focus": {
    "primary": "Improve follow-up speed",
    "secondary": ["Roleplay objections", "Book 10 discovery calls"]
  },
  "weeks_outcome": {
    "target": "10 calls booked"
  },
  "how": {
    "actions": [
      "Send follow-up within 5 minutes",
      "Complete two objection roleplays"
    ]
  },
  "development_doc_url": "https://docs.google.com/document/d/example"
}
```

Example success response, HTTP `201`:

```json
{
  "success": true,
  "developmentDoc": {
    "id": "clwdevdoc123",
    "clientId": "clwclient123",
    "callId": "clwcall123",
    "weekLabel": "Week 1",
    "incomeGoal": "$10k/month",
    "weeksFocus": {
      "primary": "Improve follow-up speed",
      "secondary": ["Roleplay objections", "Book 10 discovery calls"]
    },
    "weeksOutcome": {
      "target": "10 calls booked"
    },
    "how": {
      "actions": [
        "Send follow-up within 5 minutes",
        "Complete two objection roleplays"
      ]
    },
    "createdAt": "2026-05-06T15:00:00.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/blank `week_label`, blank optional string, invalid URL, invalid JSON field, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `404` | `Call not found.` | `call_id` does not exist or belongs to a different client. |
| `409` | `Development doc already exists for call.` | The provided `call_id` already has a development document. |
| `500` | `Failed to create development doc.` | Unexpected database/server failure. |

### POST /api/webhooks/contacts/{contactId}/one-on-one-requests - Create 1-on-1 Booking Request

Stores a client's 1-on-1 booking form submission (the Google Form for the 2nd, 3rd, and 4th 1-on-1 calls) so Elle or Mary can approve or deny it on the client's profile (other admins can view requests but not decide). Each stored request also sends an in-app notification linking straight to the request card, targeted at Elle and Mary plus any extra addresses in `ONE_ON_ONE_NOTIFY_EMAILS` (falling back to all admins when none of those emails match a user). Posting to Discord is the sender's job; this endpoint only records the request in the dashboard. On approval the dashboard mints a single-use Cal.com booking link and (when configured) pushes the decision to the outbound decision webhook described below; the booked call should then be recorded through the existing `POST /api/webhooks/contacts/{contactId}/calls` endpoint with `call_type: one_on_one`, which is what the profile's 1-on-1 counter is derived from.

```http
POST /api/webhooks/contacts/{contactId}/one-on-one-requests
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `answers` | JSON | Yes | The form submission as question/answer data: a non-empty object keyed by question, or a non-empty array of `{ "question", "answer" }` pairs. Free-form so the form can change without an API change. |
| `submitted_at` | ISO datetime | No | When the form was submitted. Defaults to the time of delivery. |
| `event_id` | string | No | Stable submission id from the form tool. Used for duplicate rejection like an `Idempotency-Key` header. |

Example request:

```json
{
  "answers": {
    "What do you want to cover?": "Objection handling on discovery calls",
    "What have you tried so far?": "Roleplays from module 3",
    "Preferred days": "Tuesday or Thursday"
  },
  "submitted_at": "2026-07-17T14:30:00.000Z",
  "event_id": "form_response_8842"
}
```

Example success response, HTTP `201`:

```json
{
  "success": true,
  "request": {
    "id": "clwreq123",
    "clientId": "clwclient123",
    "answers": {
      "What do you want to cover?": "Objection handling on discovery calls",
      "What have you tried so far?": "Roleplays from module 3",
      "Preferred days": "Tuesday or Thursday"
    },
    "submittedAt": "2026-07-17T14:30:00.000Z",
    "status": "PENDING",
    "createdAt": "2026-07-17T14:30:02.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing or empty `answers`, invalid `submitted_at`, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `409` | `Duplicate one-on-one request webhook.` | The same `Idempotency-Key` or `event_id` was already processed. |
| `500` | `Failed to create one-on-one request.` | Unexpected database/server failure. |

#### Outbound decision webhook

When an operator approves or denies a request, the dashboard POSTs a JSON body to the URL in the server's `ONE_ON_ONE_DECISION_WEBHOOK_URL` environment variable (HTTPS, public host; delivery is fire-and-forget and logged in the dashboard's outbound webhook log). If the variable is unset, decisions still save and the operator is told no notification was sent. The receiving automation is responsible for delivering the outcome to the client (for example a Discord DM or channel message).

```json
{
  "event": "one_on_one_request.decided",
  "request_id": "clwreq123",
  "status": "approved",
  "client": {
    "id": "clwclient123",
    "name": "Jamie Rivera",
    "email": "jamie.rivera@example.com",
    "contactid": "zMC7sAfinnBzqYy8n98V",
    "discord_channel_id": "123456789012345678",
    "discord_user_id": "987654321098765432"
  },
  "booking_url": "https://cal.com/d/abc123def456",
  "denial_notes": null,
  "decided_by": "Elle",
  "decided_at": "2026-07-17T15:00:00.000Z"
}
```

`status` is `approved` or `denied`. `booking_url` is the single-use Cal.com link on approval and `null` on denial; `denial_notes` is the operator's explanation on denial and `null` on approval.

### POST /api/webhooks/contacts/{contactId}/refund-requests - Create Refund Request

Stores a refund request filed by an Impact team member through the internal GHL form, so the designated approver can approve or deny it on the client's Payments tab. The sender (Make) resolves the client's GHL contact first: this endpoint is contact-scoped, so a request always lands on a named client rather than being matched by typed name or email.

Each stored request notifies the approvers listed in `REFUND_APPROVER_EMAILS`, in-app and by email, with a link straight to the request card. When that variable is unset or matches no admin/owner user, the in-app notification falls back to all admins and owners so a pending refund is never announced to nobody. After a decision, the requester is notified by email, and in-app when `requester_email` matches a dashboard user.

This endpoint records a decision only. The refund itself is issued manually in Stripe/GHL: nothing here writes to the cash ledger, and client balances are unaffected.

```http
POST /api/webhooks/contacts/{contactId}/refund-requests
```

Request body schema (strict: an unknown key is a `400`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `requester_name` | string | Yes | The Impact team member filing the request. |
| `requester_email` | string | Yes | Their email. Lowercased on write, and used to notify them of the decision. |
| `requester_role` | string | No | Their role, as typed on the form. |
| `requester_department` | string | No | Their department, as typed on the form. |
| `client_name` | string | No | The client name as typed on the form. Stored verbatim so a mis-resolved contact is diagnosable. |
| `client_email` | string | No | The client email as typed on the form. Stored verbatim for the same reason. |
| `programme` | string | No | Programme / product purchased. |
| `refund_type` | string | No | Refund type, for example full or partial. |
| `amount_requested` | number or string | No | Amount requested. Accepts `1500`, `"1500"`, `"$1,500.00"`, `"1500 USD"`. Anything still ambiguous is a `400` rather than a silently dropped field. Negative values and more than two decimal places are rejected. |
| `cause_of_refund` | string | No | Where the cause of the refund lies. |
| `reason` | string | Yes | The reason the client gave. |
| `agreement_url` | string | No | Link to the signed agreement. Must be `http` or `https`. |
| `evidence_urls` | string or string[] | No | Supporting evidence link(s). A single URL string (what the form's one file-upload field produces) or an array of up to 10. Each must be `http` or `https`. An empty string counts as no evidence. Defaults to `[]`. |
| `answers` | JSON | No | The raw form submission, kept so a later form change stays diagnosable. Defaults to a summary of the typed fields. |
| `submitted_at` | ISO datetime | No | When the form was submitted. Defaults to the time of delivery. |
| `event_id` | string | No | Submission id or static form label from the form tool. Used for duplicate rejection: the stored key is `event_id` plus a hash of the submission content (excluding `submitted_at`), so a static value (e.g. the form name) is safe and only an identical redelivery is rejected as a duplicate. Note this means two legitimate submissions with exactly the same answers also bounce with a 409; change any field to resubmit. An explicit `Idempotency-Key` header takes precedence (trimmed; max 200 chars, longer keys are ignored). |

Example request:

```json
{
  "requester_name": "Sam Okafor",
  "requester_email": "sam@impactteam.us",
  "requester_role": "Client Success",
  "requester_department": "Impact",
  "client_name": "Jamie Rivera",
  "client_email": "jamie.rivera@example.com",
  "programme": "Accelerator",
  "refund_type": "Full refund",
  "amount_requested": "$1,500.00",
  "cause_of_refund": "Client circumstance",
  "reason": "Client lost their job and cannot continue the programme.",
  "agreement_url": "https://app.pandadoc.com/documents/abc123",
  "evidence_urls": "https://drive.google.com/file/d/123/view",
  "submitted_at": "2026-08-15T10:00:00.000Z",
  "event_id": "ghl_form_9931"
}
```

Example success response, HTTP `201`:

```json
{
  "success": true,
  "request": {
    "id": "clwrefund123",
    "clientId": "clwclient123",
    "requesterName": "Sam Okafor",
    "amountRequested": "1500.00",
    "status": "PENDING",
    "submittedAt": "2026-08-15T10:00:00.000Z",
    "createdAt": "2026-08-15T10:00:02.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid contact id.` | `{contactId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing `requester_name`, `requester_email`, or `reason`; an unparseable `amount_requested`; a non-http URL; or an unknown field. |
| `404` | `Client not found.` | No client exists for `{contactId}`. |
| `409` | `Duplicate refund request webhook.` | The same `Idempotency-Key` or `event_id` was already processed. |
| `500` | `Failed to create refund request.` | Unexpected database/server failure. |

### Impact Accelerator operations

These endpoints support the Impact Accelerator fulfillment workflow. They are
strict and require the normal webhook API key.

`POST /api/webhooks/accelerator/enrollments` creates the product membership.
It also sets the client's `program` to `Impact Accelerator` (recorded in the
client's property history), so the sender does not need a separate clients
webhook call to update the program.

```json
{
  "contact_id": "ghl-contact-id",
  "enrolled_at": "2026-08-19T09:00:00.000Z",
  "whop_user_id": "whop-member-id"
}
```

`PATCH /api/webhooks/accelerator/{contactId}/onboarding` records one milestone.
Accepted `milestone` values are `pandadoc_sent`, `pandadoc_signed`,
`whop_access_granted`, `discord_linked`, `onboarding_sent`,
`portal_completed`, and `certification_unlocked`.

```json
{
  "milestone": "portal_completed",
  "happened_at": "2026-09-01T14:00:00.000Z"
}
```

`PATCH /api/webhooks/accelerator/{contactId}/rsvp-count` mirrors the GHL
custom field "IA Coaching Calls RSVP Count" (GHL is the source of truth).
Send the current **absolute** count every time the field changes, never a
delta, so retries are idempotent. The dashboard stamps the first-RSVP and
last-increase timestamps only when the count strictly increases; a lower
count (cancelled bookings) is stored without moving either timestamp. An
increase also resolves any open "no first RSVP" CX action.

```json
{
  "rsvp_count": 3
}
```

The protected daily endpoint `POST /api/automation/run-accelerator-operations`
creates durable CX actions for no first RSVP after 24 hours, no attended call
after 7 days, and certification review. Schedule it once daily with the
existing cron authentication.

## Curl Examples

These examples use **`x-api-key`** for authentication.

Create a client:

```bash
curl -X POST "https://dashboard.impactteam.us/api/webhooks/clients" \
  -H "x-api-key: $IMPACT_DASHBOARD_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "contactid": "zMC7sAfinnBzqYy8n98V",
    "name": "Jamie Rivera",
    "email": "jamie.rivera@example.com"
  }'
```

Create a call using the GHL contact ID:

```bash
curl -X POST "https://dashboard.impactteam.us/api/webhooks/contacts/zMC7sAfinnBzqYy8n98V/calls" \
  -H "x-api-key: $IMPACT_DASHBOARD_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "call_type": "onboarding",
    "scheduled_at": "2026-05-03T09:00:00.000Z"
  }'
```

Update the returned call ID:

```bash
curl -X PATCH "https://dashboard.impactteam.us/api/webhooks/calls/clwcall123" \
  -H "x-api-key: $IMPACT_DASHBOARD_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "status": "completed",
    "happened_at": "2026-05-03T09:04:00.000Z"
  }'
```
