---
name: openenvoy-manage-users-and-roles
description: >-
  Provision, update and role-assign OpenEnvoy platform users through the v1 administration surface.
api: OpenEnvoy API
generated: '2026-08-26'
method: generated
source: postman/openenvoy-postman-collection.json
base_url: https://backend.openenvoy.io/public/api/v1
operations:
  - GET /public/api/v1/users
  - GET /public/api/v1/users/{userId}
  - POST /public/api/v1/users/
  - PATCH /public/api/v1/users/{userId}
  - PATCH /public/api/v1/users/assign/role
  - GET /public/api/v1/roles
---

# Manage users and roles

Administration lives on **v1**: `https://backend.openenvoy.io/public/api/v1`.
Headers: `X-CLIENT-ID` and `Authorization: Bearer {auth_token}`.

> The User and Role schemas are **not declared** anywhere in OpenEnvoy's published Swagger
> definition. The only trustworthy description of the request and response bodies is the set of
> example payloads in the provider's own Postman collection, saved at
> `postman/openenvoy-postman-collection.json`. Read the shape from there; do not infer fields.

## Read

- `GET /public/api/v1/users` — all users. **No pagination parameters are documented**, so the
  response is unbounded. Expect the full list and handle it accordingly.
- `GET /public/api/v1/users/{userId}` — a single user, keyed on the user UUID.
- `GET /public/api/v1/roles` — the assignable roles. Call this before any role assignment rather
  than hardcoding role names.

## Create

`POST /public/api/v1/users/` with a JSON body (`Content-Type: application/json`).
Note the **trailing slash** — the collection publishes this path as `/users/`, while the read
collection is `/users`.

Two documented rejections, both HTTP 400:

- *Bad request: Email exists* — the email is already registered.
- *Bad request: External ID exists* — the supplied external ID is already in use.

Both are conflict conditions returned as 400 rather than 409. Check with `GET /users` before
creating if you need to avoid them, because **create is not idempotent** and there is no
idempotency key.

## Update

- `PATCH /public/api/v1/users/{userId}` — modify a user.
- `PATCH /public/api/v1/users/assign/role` — assign a role. This is a `PATCH` to a fixed collection
  path, not to a user-scoped path; the target user is identified in the body.

**There is no undo.** No prior-value retrieval, audit-restore or revert operation is published for
user mutations. Read the user with `GET` and retain the current values before patching, so you can
restore them yourself if needed.

## Errors

`{"errorCode","errorMessage","key"}` JSON. `key` names the offending field or header. A missing or
malformed credential returns **400**, not 401.
