---
name: trustarc-scim-user-provisioning
description: Provision, update, deprovision and audit TrustArc platform users and groups over the SCIM 2.0 endpoints on the Guardian identity API.
api: openapi/_original/trustarc-guardian-openapi.json
base_url: https://login.truste.com
operations:
  - getServiceProviderConfig
  - getSchemas
  - getResourceTypes
  - getUsers
  - createUser
  - getUser
  - updateUser
  - patchUser
  - deleteUser
  - list
  - create
  - get_2
  - update
  - patch
  - delete
generated: '2026-08-27'
method: generated
source: openapi/_original/trustarc-guardian-openapi.json (every operationId above verified verbatim in the spec)
---

# Provision TrustArc users with SCIM 2.0

TrustArc's identity service ("Guardian") exposes a standards-compliant SCIM 2.0 surface at
`https://login.truste.com/external/api/v2/scim/`. If your IdP already speaks SCIM, you do
not need a bespoke connector.

## 1. Authenticate

Get a bearer token with the OAuth 2.0 client credentials grant. Credentials are issued by a
TrustArc account administrator — there is no self-service signup.

```
POST https://login.truste.com/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=<client_id>&client_secret=<client_secret>
```

`client_id`/`client_secret` may also be sent as HTTP Basic credentials. The response carries
`access_token`, `token_type: bearer` and `expires_in` (about 6 hours in the published
sample). Send it as `Authorization: Bearer <access_token>` on every call below.

The authorization server publishes its own metadata at
`https://login.truste.com/.well-known/openid-configuration` and
`/.well-known/oauth-authorization-server` — read those rather than hard-coding endpoints.

## 2. Discover before you write

Call `getServiceProviderConfig` (`GET /external/api/v2/scim/ServiceProviderConfig`) to learn
which SCIM features this tenant supports, then `getSchemas`
(`GET /external/api/v2/scim/Schemas`) and `getResourceTypes`
(`GET /external/api/v2/scim/ResourceTypes`). Do not assume the full RFC 7643 attribute set
is honoured — read the ServiceProviderConfig first.

## 3. Read the current state

- `getUsers` — `GET /external/api/v2/scim/Users`
- `getUser` — `GET /external/api/v2/scim/Users/{id}`
- `list` — `GET /external/api/v2/scim/Groups`
- `get_2` — `GET /external/api/v2/scim/Groups/{id}`

Guardian list endpoints return the Spring Data page envelope (`content`, `totalElements`,
`totalPages`, `number`, `size`, `first`, `last`, `empty`) driven by `page`, `size` and
`sort=property,asc|desc`. Page through it; never assume one call returns everything.

## 4. Create and update

- `createUser` — `POST /external/api/v2/scim/Users` with a `ScimUserResource` body
  (`schemas`, `userName`, `name`, `emails`, `active`, `externalId`, …)
- `updateUser` — `PUT /external/api/v2/scim/Users/{id}` replaces the resource
- `patchUser` — `PATCH /external/api/v2/scim/Users/{id}` with a SCIM PatchOp document
  (`urn:ietf:params:scim:api:messages:2.0:PatchOp`) for partial change
- Groups mirror this with `create`, `update` and `patch`

Set `externalId` to your IdP's own identifier on every create. It is the only key that lets
you reconcile TrustArc back to your directory.

## 5. Safety rules an agent must follow

- **There is no idempotency key.** `createUser` is not safe to retry blindly. If a POST times
  out, `getUsers` and filter on your `externalId` before retrying, or you will create a
  duplicate.
- **`deleteUser` is not reversible.** No restore, undelete or trash endpoint exists in the
  contract. Prefer `patchUser` setting `active: false` for offboarding unless a hard delete
  is genuinely required.
- **There is no dry-run mode.** The only rehearsal surface is the staging environment, which
  needs separate credentials from your account manager.
- **Role changes take effect on the next token issuance.** After changing permissions, get a
  fresh token before verifying.

## 6. Errors

TrustArc does not use RFC 9457 problem+json. Expect either `{"error": "...",
"error_description": "..."}` or `{"timestamp", "status", "error", "path"}`.

- `401` — token missing, expired, or issued for the wrong environment. Decode the JWT `exp`
  claim. Re-authenticate.
- `403` — authenticated but under-privileged. Requires an admin-level role.
- `429` — rate limited. Thresholds are not published and vary by account tier; back off
  exponentially. No `Retry-After` or `X-RateLimit-*` header is documented, so you have no
  signal other than the 429 itself.

See `errors/trustarc-problem-types.yml` and `conventions/trustarc-conventions.yml`.
