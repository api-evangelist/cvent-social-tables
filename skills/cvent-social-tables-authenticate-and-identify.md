---
name: cvent-social-tables-authenticate-and-identify
description: >-
  Obtain a Social Tables OAuth access token with an authorization-code grant, introspect it,
  and resolve BOTH identifiers the API needs — the 4.0 alphanumeric id and the numeric
  legacy_id. Every other Social Tables flow starts here.
api: cvent-social-tables:events-api
generated: '2026-09-07'
method: generated
source: >-
  https://developer.socialtables.com/docs/authentication/,
  https://developer.socialtables.com/docs/apps/tutorial,
  https://developer.socialtables.com/docs/api-usage/legacy-ids.html,
  openapi/_original/cvent-social-tables-openapi.json
operations:
  - 'POST /4.0/oauth/token'
  - 'GET /4.0/oauth/token'
  - 'GET /4.0/teams/{team_id}/users'
note: >-
  The published Swagger 2.0 contract declares NO operationId on any of its 87 operations, so
  operations are named here by method and path — the only stable handle the provider publishes.
---

# Authenticate against Social Tables and resolve identity

Base URL: `https://api.socialtables.com`. Auth host: `https://auth.socialtables.com`.

## 1. Register the app first (human step, once)

Client ID, client secret and the redirect URI are issued on the developer portal
(<https://developer.socialtables.com>) after signing in. The redirect URI must be registered
before the handshake will work — Social Tables refuses redirects it has not seen.

## 2. Send the user to the authorization endpoint

```
GET https://auth.socialtables.com/oauth/authorize
  ?client_id=<CLIENT_ID>
  &redirect_uri=<REGISTERED_REDIRECT_URI>
  &response_type=code
```

The user authenticates and approves. Social Tables redirects back with `code` in the query
string. If the user declines you get `error` / `error_description` instead — surface it and stop.

## 3. Exchange the code for a bearer token

`POST /4.0/oauth/token` with `code`, `client_id`, `client_secret`,
`grant_type=authorization_code`, `response_type=token`.

The response carries the access token. Send it as `Authorization: Bearer <token>` on every
subsequent call.

## 4. Introspect the token — this is the step people skip

`GET /4.0/oauth/token` with the bearer header returns the identity behind the token:

```json
{ "id": "cinvup1au001as2nqyjyq07k0", "legacy_id": 1790300, "type": "individual" }
```

Keep **both**. The dual-ID rule is the single most common integration failure:

* 4.0 endpoints take the alphanumeric `id` — `GET /4.0/users/cinvup1au001as2nqyjyq07k0`
* 2.x, 3.x and `/4.0/legacy-api` endpoints take the numeric `legacy_id` —
  `GET /4.0/legacy-api/users/1790300`

Sending the wrong generation's ID surfaces as a **404**, not a validation error, so it reads
like a missing record rather than a wrong identifier.

## 5. Know what the token can do

The token represents *app + user*, and authorisation is role-based
(<https://developer.socialtables.com/docs/api-usage/permissions.html>). A user can read and
update their own user and account objects, read specifically authorised accounts, manage their
own invitations and apps, and act on teams according to their role on each team. A **401**
therefore means either "not authenticated" or "not permitted" — re-authenticating will not fix
the second case.

## Failure handling

* `401 authorization missing or invalid` — token absent, expired, or the user lacks the role.
* `404` — check ID generation before assuming the record is gone.
* No rate-limit headers and no 429 are documented anywhere, so back off conservatively on any
  repeated 4xx rather than assuming you have budget left.
* Error bodies come in two shapes: the service envelope
  `{ title, status, detail, key, requestId }` and a gateway envelope `{ code, message }`.
  Log `requestId` when you get it — it is the only correlation handle the API returns.
