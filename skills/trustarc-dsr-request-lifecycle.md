---
name: trustarc-dsr-request-lifecycle
description: Create, label, close and reconcile data subject requests in TrustArc Individual Rights Manager, including callback registration for status changes.
api: docs
base_url: https://irm.trustarc.com/server/api/v1
operations:
  - GET /api/v1/external/forms/all
  - GET /api/v1/external/requests
  - POST /api/v1/external/requests
  - POST /api/v1/external/requests/search
  - GET /api/v1/external/requests/statuses
  - PUT /api/v1/external/requests/{requestId}/labels/name/{labelName}
  - DELETE /api/v1/external/requests/{requestId}/labels/name/{labelName}
  - POST /api/v1/external/requests/{id}/close
  - GET /api/v1/external/callback
  - POST /api/v1/external/callback
generated: '2026-08-27'
method: generated
source: https://trustarchelp.zendesk.com/hc/en-us/articles/52354102146579-DROP-Process-to-IRM-API-Integration-Guide
---

# Run a data subject request end to end in IRM

Individual Rights Manager tracks legally-binding privacy requests with SLA-bearing due
dates. Treat every write here as consequential: a request you create starts a clock, and a
request you close is effectively final from the API.

The base path is `/api/v1` on your deployed IRM host (`https://irm.trustarc.com/server` for
the shared deployment). Every call needs `Authorization: Bearer <access_token>`,
`Content-Type: application/json` and `Accept: application/json`.

## 1. Discover the form BEFORE you build a payload

Request bodies are keyed by that form's **field IDs (UUIDs)** — not by display labels — and
the ids differ per form. Never hard-code them.

```
GET /api/v1/external/forms/all
```

Optional filters: `name`, `statuses`, `brandName`, `brandIds`. Take the `id` of the form you
want as your `formId`, then read its fields:

```
GET /api/v1/external/requests?formId={formId}
```

Each field carries `id`, `display`, `type`, `dataType`, `required` and `values`. Build your
create payload from the `id` values you just read.

## 2. Create the request

```
POST /api/v1/external/requests
```

Verification behaviour is governed by the form and the account configuration — there is no
per-request bypass flag. For bulk broker ingestion, set `verifyEmail=false` and/or use a
dedicated intake form.

## 3. Tag it so you can find it again

```
PUT /api/v1/external/requests/{requestId}/labels/name/{labelName}
```

Labels are the reconciliation key for batch integrations. `DELETE` on the same path removes
one. Labels are fully reversible.

## 4. Close it after you have actually deleted the data

```
POST /api/v1/external/requests/{id}/close
```

**This is a one-way door from the API's point of view.** There is no documented reopen
operation. A tenant may have the "Auto re-open closed requests on DS reply" setting enabled,
which can return a request to an open state — but that is triggered by the data subject, not
by you. Do not close until the downstream deletion is confirmed.

## 5. Reconcile

Poll for closed, labelled requests:

```
POST /api/v1/external/requests/search
GET  /api/v1/external/requests/statuses
```

Or register a callback so IRM pushes status changes to you:

```
GET  /api/v1/external/callback
POST /api/v1/external/callback
```

The registration body carries a target `url` plus filters — `requestTypeIds`, `formIds`,
`dataSubjectTypeIds`, `requestStatuses`, `brandNames`, `residentOfs`,
`requestVerificationTypes`. Note what is **not** documented: no payload signature, no shared
secret, no retry policy and no delivery guarantee. Verify inbound callbacks against the API
before acting on them.

## Safety rules

- No idempotency key exists. A timed-out `POST /api/v1/external/requests` must be reconciled
  with `POST /api/v1/external/requests/search` before retrying, or you will create a
  duplicate legal request.
- No dry-run mode. Rehearse in staging with separate credentials.
- Access tokens are short-lived; treat any `401` as "re-authenticate", not "fail".
- `429` means back off exponentially. Thresholds are per account tier and unpublished.
