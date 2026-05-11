# Impact Dashboard API

Public API documentation for automation systems that create and update client records, calls, payments, engagement events, onboarding milestones, Discord IDs, track assignments, and development documents in Impact Dashboard.

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

1. **Create the client** with `POST /api/webhooks/clients` and a stable CRM **`contactid`**. You can reuse that same `contactid` in every later URL that shows `{clientId}`, so you do not have to store the dashboard internal id (though that internal id is also accepted).
2. **Parallel enrichment updates** as data arrives: onboarding milestones (`PATCH .../onboarding`), Discord ids (`PATCH .../discord`), track/group (`PATCH .../track`).
3. **Calls**: create with `POST .../calls`, then later finalize with `PATCH /api/webhooks/calls/{callId}`. Keep the returned **`call.id`** from step one if you need to patch by call id.
4. **Financial and activity events**: payments (`POST .../payments`) and engagement (`POST .../engagement`) should use idempotency keys or stable external ids when retries are possible.
5. **Weekly dev docs** (`POST .../dev-docs`) optionally reference an existing `call_id`.

You only need the endpoints your automation actually produces; there is no requirement to call every route.

## Quickstart

1. Get webhook credentials from the Impact Dashboard administrator.
2. Send every request with `Content-Type: application/json` and **`Accept: application/json`**, then choose **either** signed webhook headers **or** the legacy `x-api-key` header (do not mix a partial set of signed headers with the legacy key).
3. Create a client with `POST /api/webhooks/clients`.
4. Send a CRM `contactid` when creating the client.
5. Use the same `contactid` in later client-specific endpoint paths (or use the internal `client.id` from the create response).
6. Store returned `call.id` values when creating calls so they can be updated later.
7. Treat every non-2xx response as a failed write and log the response body for troubleshooting.

## Base URL

```text
https://impact-dashboard.up.railway.app
```

All paths in this document are relative to this base URL.

## Authentication

All endpoints require webhook credentials. Signed webhook headers are preferred for new integrations. The legacy shared API key remains supported for existing automations during migration.

**Choosing a mode**

- If you send **any** of `x-webhook-key-id`, `x-webhook-timestamp`, or `x-webhook-signature`, the server treats the request as **signed mode** and validates all required signed headers together. Missing pieces or a bad signature produce `401`.
- Otherwise the server expects **`x-api-key`** (legacy).

Preferred signed headers for every request:

```http
x-webhook-key-id: default
x-webhook-timestamp: UNIX_TIMESTAMP_MS
x-webhook-signature: HMAC_SHA256_HEX
Content-Type: application/json
Accept: application/json
```

**How the signature is computed**

1. Take the **`x-webhook-timestamp`** value exactly as you will send it (digits only, Unix time in milliseconds).
2. Take the **raw HTTP body** exactly as sent (same bytes as after `Content-Type: application/json`).
3. Build one string: `timestamp + "." + rawBody` (one ASCII dot between them).
4. Compute **HMAC-SHA256** of that string using the signing secret the operator shares with you (this is **`WEBHOOK_SIGNING_SECRET`** on the server).
5. Send the digest as **lowercase hexadecimal** in `x-webhook-signature`.

The server rejects requests whose timestamp is more than **five minutes** from server time.

Every new integration uses **`x-webhook-key-id: default`** as in the preferred headers above. The operator configures **`WEBHOOK_SIGNING_SECRET`**; use that secret for the HMAC and keep it off the wire except in HTTPS headers described here.

**Legacy `x-api-key`**

```http
x-api-key: API_KEY
Content-Type: application/json
Accept: application/json
```

Do not send webhook secrets in query parameters or request bodies. Do not paste real credentials into documentation, screenshots, logs, or source code.

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

If your integration uses **`x-api-key`** (legacy) but the server operator has **not** set **`WEBHOOK_API_KEY`**, those requests receive HTTP **`401`** with the standard authentication error shape—the same **`401`** you get for a missing or incorrect key. Operators should configure **`WEBHOOK_API_KEY`** for legacy callers or migrate to signing with **`WEBHOOK_SIGNING_SECRET`** (above). Server logs may still record that **`WEBHOOK_API_KEY`** is unset when diagnosing failed deliveries.

## Request Rules

- Request bodies must be valid JSON **objects** (not bare arrays or primitives).
- **Extra JSON keys:** On `POST /api/webhooks/clients`, unknown keys are **ignored** (silently dropped). On **every other** webhook endpoint in this document, unknown keys cause **`400`** validation errors because schemas are strict.
- Required strings must be non-empty after trimming.
- Optional string fields may be omitted, but if present must be non-empty after trimming.
- Dates must be ISO 8601 datetimes with a timezone offset, for example `2026-05-01T10:00:00.000Z` or `2026-05-01T18:00:00+08:00`.
- Money fields may be JSON numbers or numeric strings such as `2500`, `"2500"`, or `"2500.00"`.
- JSON metadata fields (where documented) may contain strings, numbers, booleans, null, arrays, or nested objects, but nested metadata is capped at **8 levels** deep and **500** scalar/array/object nodes total per field.
- Client-specific paths accept either the dashboard client id or the CRM `contactid`.
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

### Client ID vs. Contact ID

All routes that take a `{clientId}` path parameter accept **either** the dashboard's internal id (returned in the `id` field when a client is created) **or** the CRM `contactid` sent during client creation. Using the CRM `contactid` directly is the recommended approach for automations so there is no need to store and forward the internal id between steps.

### Common Errors

| Status | Message | Meaning |
| --- | --- | --- |
| `400` | `Invalid request payload.` | JSON parsed but failed schema validation, or a path ID was blank/invalid. |
| `401` | `Invalid webhook credentials.` | Missing or incorrect signed headers, wrong legacy **`x-api-key`**, or legacy mode used while **`WEBHOOK_API_KEY`** is unset on the server (see [Authentication](#authentication)). |
| `403` | `Request origin not permitted.` | Caller IP not in the server's webhook IP allowlist (when configured). |
| `404` | `Client not found.` | The `{clientId}` path parameter does not match a client by internal id or `contactid`. |
| `404` | `Call not found.` | The `{callId}` path parameter or linked `call_id` does not match a call. |
| `409` | Resource-specific conflict message | A unique value already exists. |
| `429` | `Too many requests.` | Webhook rate limit exceeded for this IP; retry with backoff. |
| `500` | Resource-specific failure message | Unexpected server or database failure. Retry with backoff and alert if repeated. |

### Rate Limits

Webhook routes may return HTTP **`429`** with `Too many requests.` when a caller exceeds the configured per-IP limit for `/api/webhooks` (the reference implementation uses **120 requests per rolling minute** per IP). Treat **`429`** like other transient errors: back off and retry.

In **this repository**, those limits are enforced in **`proxy.ts`** at the project root (Next.js 16’s proxy hook convention; **`middleware.ts`** is not used). Paths under **`/api/auth`** share a tighter per-IP limit for abuse protection.

For payment and engagement event webhooks, send an `Idempotency-Key` header or a stable external event/payment id in the request body where documented. Replays with the same key are rejected as duplicates instead of creating a second financial or event record.

## Endpoint Summary

| Name | Method | Path | Purpose |
| --- | --- | --- | --- |
| Create Client | `POST` | `/api/webhooks/clients` | Create a client from a sale or enrollment event. |
| Update Onboarding | `PATCH` | `/api/webhooks/clients/{clientId}/onboarding` | Move a client to an onboarding milestone and record metadata as an engagement event. |
| Update Discord IDs | `PATCH` | `/api/webhooks/clients/{clientId}/discord` | Save the client's Discord channel and user IDs. |
| Update Track | `PATCH` | `/api/webhooks/clients/{clientId}/track` | Assign the client to a dashboard track and group. |
| Create Call | `POST` | `/api/webhooks/clients/{clientId}/calls` | Create a scheduled call record. |
| Update Call | `PATCH` | `/api/webhooks/calls/{callId}` | Update a call after completion, no-show, cancellation, or rebooking. |
| Create Payment | `POST` | `/api/webhooks/clients/{clientId}/payments` | Record a backend payment and recalculate remaining balance. |
| Create Engagement Event | `POST` | `/api/webhooks/clients/{clientId}/engagement` | Log client activity or learning engagement. |
| Create Development Document | `POST` | `/api/webhooks/clients/{clientId}/dev-docs` | Save an AI-generated or automation-generated development document. |

## Endpoint Reference

### POST /api/webhooks/clients - Create Client

Creates a new dashboard client from a sale or enrollment event. Extra JSON keys in the body are **ignored** on this route only.

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
| `contactid` | string | Yes | CRM contact ID. Stored and used to identify the client in all follow-up webhook calls. |
| `name` | string | Yes | Client full name. |
| `email` | string | Yes | Client email. Lowercased and validated as an email address. Must be unique. |
| `phone` | string | No | Client phone number. |
| `program` | enum | No | One of `Platinum`, `Platinum Upsell (The House)`. |
| `lead_source` | enum | No | One of `Referral / Other`, `Outbound`, `VIP Onboarding`, `Application Lead`, `Discord DM's`, `Offer Placement Call`. |
| `closer` | enum | No | One of `Sam`, `Dan`, `Hunter`, `Phillip`. |
| `setter` | enum | No | One of `Sam`, `Dan`, `Hunter`, `Phillip`. |
| `sale_date` | ISO datetime | No | Sale timestamp with timezone. |
| `contract_value` | money | No | Total contract value. |
| `revenue` | money | No | Revenue amount to display. |
| `cash_collected` | money | No | Cash collected at sale. |
| `remaining_balance` | money | No | Remaining balance at creation. |
| `payment_plan` | string | No | Payment plan description. |
| `context_notes` | string | No | Internal context notes. |

Default values set by the API:

| Field | Default |
| --- | --- |
| `paymentStatus` | `INCOMPLETE` |
| `track` | `BEGINNER` |
| `groupAssignment` | `A` |
| `healthStatus` | `GREEN` |
| `onboardingStatus` | `PAID` |

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
  "context_notes": "Prefers evening calls."
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
    "groupAssignment": "A",
    "onboardingStatus": "PAID",
    "healthStatus": "GREEN",
    "remainingBalance": "4000",
    "createdAt": "2026-05-01T10:00:02.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid request payload.` | Missing `contactid`, `name`, or `email`, invalid email, invalid date/money format, or unknown fields. |
| `409` | `Client already exists.` | Another client already uses the email or `contactid`. |
| `500` | `Failed to create client.` | Unexpected database/server failure. |

### PATCH /api/webhooks/clients/{clientId}/onboarding - Update Onboarding

Updates a client's onboarding status and creates an `ONBOARDING_MILESTONE` engagement event with optional metadata.

```http
PATCH /api/webhooks/clients/{clientId}/onboarding
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `milestone` | enum | Yes | One of `contract_signed`, `questionnaire_completed`, `call_booked`, `call_completed`. |
| `metadata` | JSON | No | Any structured automation context to store with the milestone event. |

Milestone mapping:

| Request value | Stored dashboard value |
| --- | --- |
| `contract_signed` | `CONTRACT_SIGNED` |
| `questionnaire_completed` | `QUESTIONNAIRE_COMPLETED` |
| `call_booked` | `CALL_BOOKED` |
| `call_completed` | `CALL_COMPLETED` |

Example request:

```json
{
  "milestone": "questionnaire_completed",
  "metadata": {
    "form_id": "typeform_123",
    "submitted_by": "jamie.rivera@example.com"
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
| `400` | `Invalid client id.` | `{clientId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid `milestone`, invalid `metadata`, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{clientId}`. |
| `500` | `Failed to update client onboarding.` | Unexpected database/server failure. |

### PATCH /api/webhooks/clients/{clientId}/discord - Update Discord IDs

Stores Discord identifiers for a client.

```http
PATCH /api/webhooks/clients/{clientId}/discord
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
| `400` | `Invalid client id.` | `{clientId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/blank fields or unknown fields. |
| `404` | `Client not found.` | No client exists for `{clientId}`. |
| `409` | `Discord identifier already exists.` | The channel ID or user ID is already assigned to another client. |
| `500` | `Failed to update client Discord details.` | Unexpected database/server failure. |

### PATCH /api/webhooks/clients/{clientId}/track - Update Track

Assigns a client to a track and group.

```http
PATCH /api/webhooks/clients/{clientId}/track
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `track` | enum | Yes | One of `beginner`, `advanced`. |
| `group_assignment` | enum | Yes | One of `A`, `B`. |

Track mapping:

| Request value | Stored dashboard value |
| --- | --- |
| `beginner` | `BEGINNER` |
| `advanced` | `ADVANCED` |

Example request:

```json
{
  "track": "beginner",
  "group_assignment": "A"
}
```

Example success response, HTTP `200`:

```json
{
  "success": true,
  "client": {
    "id": "clwclient123",
    "track": "BEGINNER",
    "groupAssignment": "A",
    "updatedAt": "2026-05-01T10:25:00.000Z"
  }
}
```

Endpoint-specific errors:

| Status | Message | When it happens |
| --- | --- | --- |
| `400` | `Invalid client id.` | `{clientId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid enum values or unknown fields. |
| `404` | `Client not found.` | No client exists for `{clientId}`. |
| `500` | `Failed to update client track.` | Unexpected database/server failure. |

### POST /api/webhooks/clients/{clientId}/calls - Create Call

Creates a scheduled call record for a client. The call starts with status `SCHEDULED`.

```http
POST /api/webhooks/clients/{clientId}/calls
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
| `400` | `Invalid client id.` | `{clientId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid `call_type`, invalid date, non-positive `call_number`, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{clientId}`. |
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
| `contact_id` | string | No | If set, the call must belong to this client (dashboard client id or CRM `contactid`). Omit to update by call id only. |

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

Omit `contact_id` when you only have the dashboard `call.id` from the create-call response. Include it when you want the server to reject the update if the call is not tied to that CRM contact or internal client id.

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

### POST /api/webhooks/clients/{clientId}/payments - Create Payment

Records a backend payment and recalculates the client's remaining balance.

```http
POST /api/webhooks/clients/{clientId}/payments
```

Request body schema:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `external_payment_id` | string | No | Stable upstream payment id used to reject duplicate payment deliveries. |
| `amount` | money | Yes | Payment amount received. |
| `payment_date` | ISO datetime | Yes | Payment timestamp with timezone. |
| `cash_collected` | money | No | Cash collected value to store on this payment and optionally update on the client. |
| `closer` | enum | No | Closer credited for the payment. One of `Sam`, `Dan`, `Hunter`, `Phillip`. |
| `notes` | string | No | Payment notes. |

Balance behavior:

- If the client has `contractValue`, remaining balance becomes `contractValue - sum(all payments)`, clamped to `0`.
- If the client does not have `contractValue`, remaining balance becomes current `remainingBalance - amount`, clamped to `0`.
- If `cash_collected` is provided, the client's `cashCollected` field is updated to that value.

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
| `400` | `Invalid client id.` | `{clientId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing `amount` or `payment_date`, invalid money/date format, blank optional string, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{clientId}`. |
| `409` | `Duplicate payment webhook.` | The same `Idempotency-Key` or `external_payment_id` was already processed. |
| `500` | `Failed to create payment.` | Unexpected database/server failure. |

### POST /api/webhooks/clients/{clientId}/engagement - Create Engagement Event

Logs a client activity event.

```http
POST /api/webhooks/clients/{clientId}/engagement
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
| `400` | `Invalid client id.` | `{clientId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/invalid `event_type`, invalid `event_date`, invalid metadata, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{clientId}`. |
| `409` | `Duplicate engagement webhook.` | The same `Idempotency-Key` or `event_id` was already processed. |
| `500` | `Failed to create engagement event.` | Unexpected database/server failure. |

### POST /api/webhooks/clients/{clientId}/dev-docs - Create Development Document

Saves an AI-generated or automation-generated development document for a client. If `call_id` is included, the call must belong to the same client.

```http
POST /api/webhooks/clients/{clientId}/dev-docs
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
  }
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
| `400` | `Invalid client id.` | `{clientId}` is blank or invalid. |
| `400` | `Invalid request payload.` | Missing/blank `week_label`, blank optional string, invalid JSON field, or unknown fields. |
| `404` | `Client not found.` | No client exists for `{clientId}`. |
| `404` | `Call not found.` | `call_id` does not exist or belongs to a different client. |
| `409` | `Development doc already exists for call.` | The provided `call_id` already has a development document. |
| `500` | `Failed to create development doc.` | Unexpected database/server failure. |

## Curl Examples

These examples use **`x-api-key`**. For signed webhooks, send **`x-webhook-key-id`**, **`x-webhook-timestamp`**, and **`x-webhook-signature`** as described under [Authentication](#authentication), and omit **`x-api-key`**.

Create a client:

```bash
curl -X POST "https://impact-dashboard.up.railway.app/api/webhooks/clients" \
  -H "x-api-key: $IMPACT_DASHBOARD_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "contactid": "zMC7sAfinnBzqYy8n98V",
    "name": "Jamie Rivera",
    "email": "jamie.rivera@example.com"
  }'
```

Create a call using the CRM contact ID:

```bash
curl -X POST "https://impact-dashboard.up.railway.app/api/webhooks/clients/zMC7sAfinnBzqYy8n98V/calls" \
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
curl -X PATCH "https://impact-dashboard.up.railway.app/api/webhooks/calls/clwcall123" \
  -H "x-api-key: $IMPACT_DASHBOARD_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "status": "completed",
    "happened_at": "2026-05-03T09:04:00.000Z"
  }'
```

