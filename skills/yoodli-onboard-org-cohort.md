---
name: Onboard a cohort into a Yoodli Organization and User Group
description: Create a User Group in a Yoodli Organization and invite or add a batch of users to it, handling the 207 per-email result and the strict batch size.
api: openapi/yoodli-api-openapi.yml
base_url: https://app.yoodli.ai/api
operations:
  - POST /v3/orgs/{orgId}/hubs
  - POST /v3/orgs/{orgId}/users
  - GET /v3/orgs/{orgId}
  - GET /v3/orgs/{orgId}/invites
generated: '2026-09-04'
method: generated
source: openapi/yoodli-api-openapi.yml + https://developers.yoodli.ai/docs/managing-organization-users
---

# Onboard a cohort into a Yoodli Organization

> **Operation handles.** Yoodli's published OpenAPI declares **no `operationId`** on any operation, so every
> step below is addressed by HTTP method + path exactly as the spec declares it. Do not invent an id.

## Before you start

- Auth: `Authorization: Bearer <API key>`. The key is an **Organization Management API key**, minted in the
  Yoodli app by an Organization Administrator or Owner. There is no OAuth flow and no scopes.
- Terminology: the API says **hub**, the Yoodli application says **User Group**. They are the same thing.
- There is **no idempotency key**. `POST /v3/orgs/{orgId}/hubs` is *not* safe to blind-retry — a retry
  creates a second User Group. See `conventions/yoodli-conventions.yml`.

## Steps

1. **Read the Organization and its existing User Groups** — `GET /v3/orgs/{orgId}`.
   Check whether a group with your intended name already exists before creating one; this is the only
   protection against duplicate groups, because step 2 has no replay protection.

2. **Create the User Group** — `POST /v3/orgs/{orgId}/hubs`, body per `CreateHubRequest`.
   Expect **201**. Rate-limit category **Slow API — 5 calls per minute**; this is the tightest limit on the
   whole API, so create groups serially and never in a loop without pacing.
   - `400` invalid body. `403` the Organization's User Group quota is exhausted. `404` no access.
   - On a timeout or a connection error, **do not retry blindly** — re-run step 1 and look for the group first.

3. **Add or invite the users** — `POST /v3/orgs/{orgId}/users`, body per `AddOrgUsersRequest`.
   - **Maximum 20 users per request.** Chunk your cohort into batches of 20.
   - Rate-limit category **Medium API — 20 calls per minute**.
   - A user already in the Organization is added directly to the named User Groups. A user who is not a
     member is *invited*, and only joins the Organization once they accept.
   - Expect **207 Multi-Status**, not 200. The body carries a **per-email result**. You must walk it —
     a 207 does not mean every address succeeded.

4. **Optionally set a membership expiry** — `PATCH /v3/orgs/{orgId}/members/expiration`, up to **100
   emails** per request, `expiration_date` as a UTC timestamp. Passing `null` clears it. Also returns 207.

5. **Confirm** — `GET /v3/orgs/{orgId}/users` for accepted members and `GET /v3/orgs/{orgId}/invites` for
   the ones still outstanding. Both paginate with `start` + `limit` (default 20, **max 1000**) and sort with
   `sort` (`name`/`-name`/`email`/`-email`). To look up one person, use `sort=email` with `prefix=<address>`.

## Errors and retries

- Envelope is `{ "error": "...", "code": "..." }` — `error` is a developer string and is **not translated**;
  `code` is optional. It is **not** RFC 9457 problem+json.
- `429` is documented on the rate-limit page but **is not declared on any operation** and Yoodli documents
  **no** `X-RateLimit-*`, `RateLimit-*` or `Retry-After` header. You cannot read your remaining budget —
  count your own calls against the tier above and use exponential backoff after a 429.
- Capture the **`x-request-id`** response header on every call. It is what Yoodli support asks for first.

## Reversing this

`POST /v3/orgs/{orgId}/users/remove` (up to 100 emails) removes members and deletes pending invites.
`DELETE /v3/orgs/{orgId}/hubs/{hubId}` deletes the group. **Yoodli states no time window for either** —
treat both as immediate and unbounded. See the deprovision skill before deleting anything.
