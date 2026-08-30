---
name: trustarc-cookie-consent-website-scan
description: Register websites against a TrustArc Cookie Consent Manager instance, queue a tracker scan, and read the resulting tracker inventory using the CCM External API v1.
api: docs
base_url: https://<your-ccm-domain>/external/v1
operations:
  - POST /external/v1/websites
  - DELETE /external/v1/websites
  - POST /external/v1/scans
  - GET /external/tracker/inventory/{cmId}
generated: '2026-08-27'
method: generated
source: https://trustarchelp.zendesk.com/hc/en-us/articles/53517557106963-API-Reference
---

# Add websites and scan them for trackers

The CCM External API v1 is TrustArc's newest and best-documented external surface (v1.0,
released 2025-04-23). It has no OpenAPI document — every operation below is transcribed
from the published API Reference, not from a spec.

## 1. Authenticate and check your role

Get a client-credentials bearer token (see `authentication/trustarc-authentication.yml`).
**Every** CCM External API endpoint requires one of `SUPER_ADMIN`, `ADMIN` or `DESIGNER`. A
403 means the role is missing — and a role change only applies to the *next* token you
issue, so re-authenticate after any change.

## 2. Find your `cmUuid`

Every call is scoped to one Consent Manager instance. Log into the CCM portal, open the
Cookie Consent Manager Settings page, and read the UUID from the page URL or the Settings
panel. Websites are scoped per instance — the wrong `cmUuid` is the usual cause of a 404 on
scan creation.

## 3. Add websites

```
POST /external/v1/websites
Authorization: Bearer <token>
Content-Type: application/json

{ "cmUuid": "<uuid>", "websites": ["https://example.com", "https://shop.example.com"] }
```

Batch multiple URLs in one call — each is processed independently.

**Read every response element.** The body is an array of `WebsiteResultDTO`
(`url`, `status`, `message`):

- `201 Created` — every URL succeeded
- `207 Multi-Status` — **partial success.** At least one failed. Iterate the array.
- `400 Bad Request` — every URL failed; nothing was added

`status: ALREADY_EXISTS` is not a failure worth retrying — the site is already registered.

## 4. Queue a scan

```
POST /external/v1/scans
```

A `201` means the scan was **queued, not completed**. Cookie scans run asynchronously for
minutes to hours depending on site size. A `409 Conflict` means a scan is already running
for that website — wait rather than resubmitting. A `404` means the `websiteId` does not
exist under that `cmUuid`; add the website first.

## 5. Read the tracker inventory

```
GET /external/tracker/inventory/{cmId}
```

## 6. Removing a website

`DELETE /external/v1/websites` takes the **same JSON body** as the add call. This is a
DELETE with a request body — some HTTP clients and proxies will not send one. Verify your
client supports it before relying on this path.

## Limits and failure handling

- Rate limits exist but are not published; they vary by account tier. Back off
  exponentially on `429`.
- There is no idempotency key. A timed-out `POST /external/v1/scans` cannot be safely
  replayed — check for a running scan (409 behaviour) before retrying.
- Adding a website is reversible via the DELETE. Queuing a scan is not cancellable through
  any documented operation.

Support: `api-support@trustarc.com`. Include the full request (token redacted), the
complete response body, the error timestamp and your account identifier.
