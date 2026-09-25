# API Integration Guide

This guide helps third-party developers integrate with the xConfess HTTP API.
It covers authentication, public endpoints, rate limits, error handling, webhook delivery, and copy-paste examples in `curl`, JavaScript, and Python.

## Base URL

All endpoints are served under the server base URL plus the global API prefix:

- `https://<your-host>/api`

For local development, the backend runs at `http://localhost:5000/api` by default.

## Authentication Overview

xConfess uses stateless JWT authentication for protected routes.
Third-party integrations should authenticate by exchanging email/password credentials for an access token.

### Supported auth endpoints

- `POST /api/users/register` — create a new account
- `POST /api/users/login` — login with email/password
- `POST /api/auth/login` — alternative login endpoint (same payload)
- `GET /api/auth/me` — get profile for current JWT user
- `GET /api/auth/session` — get authenticated session information
- `POST /api/auth/logout` — acknowledge logout
- `POST /api/auth/forgot-password` — request password reset
- `POST /api/auth/reset-password` — complete password reset

### Login request

`POST /api/auth/login`

Request body:

```json
{
  "email": "alice@example.com",
  "password": "Str0ng!Pass#1"
}
```

Successful response:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "anonymousUserId": "anon_7f3a2b1c",
  "user": {
    "id": 1,
    "username": "alice_42",
    "role": "user",
    "is_active": true
  }
}
```

> Tip: `POST /api/users/login` is equivalent for login flows and can be used interchangeably.

### Use the JWT

Include the token on protected requests:

```http
Authorization: Bearer <access_token>
```

### Profile endpoints

- `GET /api/auth/me`
- `GET /api/auth/session`
- `GET /api/users/profile`

These return the current authenticated user profile. Use whichever route best fits your integration.

## Account deletion orchestration

Account deletion is a stateful, idempotent job. Deletion spans posts, messages,
exports, notifications, analytics, and chain references, each with different
retention requirements, so the API exposes an explicit lifecycle rather than a
single destructive call.

### Lifecycle states

| `state`         | Meaning | Terminal |
|-----------------|---------|----------|
| `requested`     | Deletion was requested but not yet confirmed by the user. | no |
| `confirmed`     | The user confirmed intent; the grace period has started. | no |
| `grace_period`  | Waiting out the configurable grace window; deletion can still be cancelled. | no |
| `processing`    | The job is actively deleting and anonymizing records. | no |
| `completed`     | All deletable records are gone and retained records are de-identified. | yes |
| `failed`        | The job hit a terminal error; inspect `failureReason` and retry. | yes |
| `cancelled`     | The user cancelled during the grace period; nothing was deleted. | yes |

Transitions are one-directional: `requested → confirmed → grace_period →
processing → completed | failed`, with `cancelled` reachable only from
`confirmed` or `grace_period`.

### Endpoints

- `POST /api/account/deletion` — request deletion (enters `requested`)
- `POST /api/account/deletion/confirm` — confirm intent (enters `confirmed`/`grace_period`)
- `POST /api/account/deletion/cancel` — cancel during the grace period
- `GET /api/account/deletion` — fetch current job status

All endpoints require the `Authorization: Bearer <access_token>` header and
operate on the authenticated user only.

### Request deletion

`POST /api/account/deletion`

```json
{
  "reason": "Leaving the platform"
}
```

Response:

```json
{
  "state": "requested",
  "jobId": "del_9f2c1a",
  "gracePeriodSeconds": 604800,
  "requestedAt": "2026-04-25T10:00:00.000Z",
  "scheduledFor": null
}
```

### Confirm deletion

`POST /api/account/deletion/confirm`

```json
{
  "confirmationToken": "del_9f2c1a"
}
```

Confirmation is required before any data is touched. On success the job enters
`grace_period` and `scheduledFor` is set to the end of the grace window.

### Cancel deletion

`POST /api/account/deletion/cancel`

Cancellation is only accepted while the job is in `confirmed` or
`grace_period`. Once `processing` begins, cancellation returns `409 Conflict`.

### Status

`GET /api/account/deletion`

```json
{
  "state": "grace_period",
  "jobId": "del_9f2c1a",
  "gracePeriodSeconds": 604800,
  "requestedAt": "2026-04-25T10:00:00.000Z",
  "scheduledFor": "2026-05-02T10:00:00.000Z",
  "retainedRecords": [
    { "category": "financial", "reason": "legal_retention", "anonymized": true }
  ]
}
```

### Idempotency and observability

Deletion requests are idempotent: repeating `POST /api/account/deletion` while a
job is active returns the existing job rather than creating a new one. Every
state transition is recorded with a timestamp so the user-facing status is
always accurate.

### Anonymization and legal retention

Records that must be retained for legal or financial reasons (for example,
settled tips and audit logs) are **de-identified** rather than deleted: direct
identifiers are replaced with a stable pseudonym and the original values are
discarded. Each retained category is reported in `retainedRecords` with a
`reason` and `anonymized: true` so the retention is justified and auditable.

## Public endpoint reference

### Create confession

`POST /api/confessions`

Request body:

```json
{
  "message": "I finally took a break and it helped.",
  "gender": "other",
  "tags": ["wellbeing", "work"],
  "stellarTxHash": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
}
```

Response example:

```json
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "message": "I finally took a break and it helped.",
  "gender": "other",
  "tags": ["wellbeing", "work"],
  "view_count": 0,
  "created_at": "2026-04-25T10:00:00.000Z"
}
```

### List confessions

`GET /api/confessions`

Query parameters:

- `page` (optional)
- `limit` (optional)

Response shape:

```json
{
  "data": [ /* confession objects */ ],
  "total": 1,
  "page": 1,
  "limit": 20
}
```

### Search confessions

`GET /api/confessions/search`

Search parameters are validated and allow hybrid query behavior.

### Full-text search

`GET /api/confessions/search/fulltext`

Same query shape as `/confessions/search` but performs a full-text search over confession content.

### Trending confessions

`GET /api/confessions/trending/top`

Returns the current top trending confessions.

### Tags

- `GET /api/confessions/tags` — list all available tags
- `GET /api/confessions/tags/:tag` — list confessions for a tag

### Confession details and updates

- `PUT /api/confessions/:id` — update an existing confession
- `DELETE /api/confessions/:id` — soft-delete a confession
- `PATCH /api/confessions/:id/restore` — restore a soft-deleted confession

### Stellar anchoring

- `POST /api/confessions/:id/anchor` — anchor a confession on Stellar
- `GET /api/confessions/:id/stellar/verify` — verify a confession anchor

### Reactions

`POST /api/reactions`

Request body:

```json
{
  "confessionId": "4f8f8eb0-b6d8-4a92-8f77-6fa3c7aa2e67",
  "anonymousUserId": "2c11e9ce-4f2f-4f06-a5d8-faf2917fd5d9",
  "emoji": "🔥"
}
```

### Messages

- `POST /api/messages` — send an anonymous message to a confession author
- `POST /api/messages/reply` — reply to an anonymous message as the confession author
- `GET /api/messages/threads` — list message threads for authenticated user
- `GET /api/messages` — list messages in a conversation thread

`POST /api/messages` body:

```json
{
  "confession_id": "4f8f8eb0-b6d8-4a92-8f77-6fa3c7aa2e67",
  "content": "Thanks for sharing this."
}
```

### Reports

`POST /api/reports`

Request body:

```json
{
  "confessionId": "4f8f8eb0-b6d8-4a92-8f77-6fa3c7aa2e67",
  "type": "spam",
  "reason": "Repeated promotional content"
}
```

### Tipping

- `GET /api/confessions/:id/tips` — list tips for a confession
- `GET /api/confessions/:id/tips/stats` — tip aggregate stats
- `POST /api/confessions/:id/tips/verify` — verify an XLM tip transaction

`POST /api/confessions/:id/tips/verify` body:

```json
{
  "txId": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
}
```

#### Response states

Every response carries a typed `state` field. Consumers should switch on
`state` rather than parsing the `message` text, since message wording may
change across releases. `verified` and `duplicate` are success outcomes
(2xx); everything else is a typed error.

| `state`     | HTTP status | Meaning | `canRetry` |
|-------------|-------------|---------|------------|
| `verified`  | 201 | This request performed first-writer settlement. | — |
| `duplicate` | 201 | A prior request already settled this exact `(confessionId, txId)` pair. This is a safe, canonical replay — not an error. | — |
| `pending`   | 409 | Another request is actively settling this pair right now. | `true` |
| `stale`     | 409 | Verification exceeded the SLA threshold and is under reconciliation review. This is not a terminal failure. | `true` |
| `conflict`  | 409 | The transaction ID is already bound to a *different* confession, or reconciliation flagged a genuine conflict. | `false` |
| `failed`    | 400 | Verification failed terminally (invalid amount, transaction not found or invalid on-chain) — or a transient/retryable error such as a Horizon network failure. | `true` for transient errors, `false` for terminal ones |

Malformed transaction IDs (not a 64-character hex string) are rejected by
request validation before reaching this logic, returning a standard
`400 Bad Request` with a `message` array — they do not carry a `state`
field, since they never reach the verification pipeline.

Success response body (`verified` or `duplicate`):

```json
{
  "state": "verified",
  "success": true,
  "isNew": true,
  "isIdempotent": false,
  "tip": {
    "id": "tip-abc-123",
    "confessionId": "4f8f8eb0-b6d8-4a92-8f77-6fa3c7aa2e67",
    "amount": 100,
    "txId": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "senderAddress": null,
    "status": "verified",
    "verifiedAt": "2026-04-25T10:00:00.000Z",
    "createdAt": "2026-04-25T10:00:00.000Z"
  }
}
```

The `tip` object in this response is intentionally a reduced, public-safe
view. Internal-only fields (`idempotencyKey`, `processingLock`, `lockedAt`,
`lockedBy`, `retryCount`, `lastChainStatus`, `lastCheckedAt`,
`reconciliationMetadata`) are never included on the wire.

Typed error response body (`pending` / `stale` / `conflict` / `failed`):

```json
{
  "message": "Transaction aaaa...aaaa verification has exceeded the expected processing time and is under review. It has not failed — check back shortly or contact support with this reference.",
  "state": "stale",
  "conflictReason": "ALREADY_PROCESSING",
  "canRetry": true
}
```

### Health checks

- `GET /api/health/live`
- `GET /api/health/ready`

These endpoints are useful for monitoring and verifying backend availability.

## Rate limiting

xConfess enforces request throttling both globally and on sensitive endpoints.

### Default API rate limits

- `GET` requests: 50 requests per 60 seconds per client IP
- `POST`, `PUT`, `PATCH`, `DELETE` requests: 5 requests per 60 seconds per client IP

### Route-specific limits

- `POST /api/auth/login` and `POST /api/users/login`: 5 requests / 60 second

/* … truncated 6303 chars — edit only what you need near the top … */
