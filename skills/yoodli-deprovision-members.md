---
name: Deprovision Yoodli members and User Groups safely
description: Remove users from a Yoodli Organization or User Group, or delete a User Group, without silently ejecting people from the tenant.
api: openapi/yoodli-api-openapi.yml
base_url: https://app.yoodli.ai/api
operations:
  - POST /v3/orgs/{orgId}/users/remove
  - POST /v3/orgs/{orgId}/hubs/{hubId}/users/remove
  - DELETE /v3/orgs/{orgId}/hubs/{hubId}
  - PATCH /v3/orgs/{orgId}/members/expiration
  - GET /v3/orgs/{orgId}
generated: '2026-09-04'
method: generated
source: openapi/yoodli-api-openapi.yml + https://developers.yoodli.ai/docs/managing-user-groups
---

# Deprovision Yoodli members and User Groups

> **Operation handles.** The published spec declares **no `operationId`**; steps are addressed by method + path.

## The one thing to get right

`DELETE /v3/orgs/{orgId}/hubs/{hubId}` takes a **`transfer`** query parameter, and its default is the
destructive one:

| `transfer` | What happens to members who belonged **only** to this group |
|---|---|
| `true` | They are moved into the **default** User Group and stay in the Organization |
| `false` **or omitted** | They are **removed from the Organization entirely** |

Omitting `transfer` is not a no-op. **Always pass `transfer=true`** unless a human has explicitly asked for
the members to be ejected from the tenant. The `default` User Group itself cannot be deleted.

## Steps

1. **See who is affected first** — `GET /v3/orgs/{orgId}` for the group list, then
   `GET /v3/orgs/{orgId}/users` with `field=hubs` to see each member's group memberships. Yoodli notes that
   the extra field increases response time; ask for it anyway before a destructive call. There is **no
   dry-run mode**, so this read is the only preview available.

2. **Remove from a User Group** (keeps them in the Organization) —
   `POST /v3/orgs/{orgId}/hubs/{hubId}/users/remove`, up to **100 emails**.
   Yoodli's stated behaviour: anyone who would lose access to *all* User Groups of the Organization is added
   to the **fallback** User Group instead of being dropped.

3. **Remove from the Organization** (harder) — `POST /v3/orgs/{orgId}/users/remove`, up to **100 emails**.
   Removes members *and* deletes their pending invites.

4. **Prefer an expiry to a removal when the intent is time-bounded** —
   `PATCH /v3/orgs/{orgId}/members/expiration` with a UTC `expiration_date`. This is the only reversible
   membership write in the API: send the same operation with `expiration_date: null` to clear it.

## What comes back

All three write operations return **207 Multi-Status** with a per-email result. Walk the list; a 207 is not
a success. All three are rate-limit category **Medium API — 20 calls per minute**.

## Reversibility, honestly

Re-inviting a removed user via `POST /v3/orgs/{orgId}/users` restores membership but **Yoodli publishes no
statement that their history, recordings or scores return with them**, and states **no window** for any of
this. Do not tell a user an undo window exists. See `conventions/yoodli-conventions.yml` `reversibility:`.
