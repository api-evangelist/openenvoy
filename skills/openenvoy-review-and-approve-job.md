---
name: openenvoy-review-and-approve-job
description: >-
  Find OpenEnvoy jobs needing attention, inspect their matching discrepancies, re-run matching after
  correcting documents, and approve or reject a job. Covers the highest-consequence action in the
  API — approving an invoice for payment.
api: OpenEnvoy API
generated: '2026-08-26'
method: generated
source: postman/openenvoy-postman-collection.json
base_url: https://backend.openenvoy.io/public/api/v2
operations:
  - POST /public/api/v2/jobs
  - GET /public/api/v2/jobs/{jobNumber}
  - POST /public/api/v2/jobs/{jobNumber}/rematch
  - POST /public/api/v2/jobs/{jobNumber}/status
  - POST /public/api/v2/jobs/{jobNumber}/approve
  - PATCH /public/api/v2/jobs/{jobNumber}/delete
---

# Review and approve a job

This flow lives on the **v2** base URL: `https://backend.openenvoy.io/public/api/v2`.
Job *creation* is v1-only — the two versions are complementary, not successive, so a full workflow
uses both.

Headers are the same on both versions: `X-CLIENT-ID` and `Authorization: Bearer {auth_token}`.

## Consequence warning — read before acting

`POST /public/api/v2/jobs/{jobNumber}/approve` **approves an invoice for payment.** OpenEnvoy
publishes no reversal window for it, and the permitted status transitions are not documented, so it
cannot be confirmed from the public contract that an approval can be undone via the status endpoint.

**Treat approval as irreversible.** An agent must not call it autonomously on the strength of a
clean match alone. Require explicit human confirmation, and surface the discrepancy summary and the
amount at the point of confirmation.

## Step 1 — Find jobs

`POST /public/api/v2/jobs` — a search-shaped POST with a JSON body.

> The request body schema is **not published**. Do not invent filter fields. Use the example body in
> the provider's Postman collection (`postman/openenvoy-postman-collection.json`, folder *JobsV2 /
> Get all jobs*) as the only trustworthy source of its shape, and treat anything beyond it as
> unsupported. Pagination behaviour is likewise undocumented.

## Step 2 — Inspect a job

`GET /public/api/v2/jobs/{jobNumber}`

Work through `matching_info[]` on each document version, comparing the `invoice_*` fields to the
`matched_*` fields per `charge_type`. Where currencies differ, `exchange_rate` is supplied.

Report, per job: total invoiced vs total matched, and the specific charge lines that diverge. Do not
summarise a job as "clean" if any charge line lacks a matched counterpart.

## Step 3 — Correct and re-match

If the discrepancy is caused by a missing or stale baseline document:

1. Attach the correct document — `POST /public/api/v1/jobs/{jobNumber}/upload` (**v1**), or replace
   a specific document version with `PUT /public/api/v1/jobs/{jobNumber}/job_documents/{documentId}`.
   Document updates are versioned, not overwritten (`version_number` increments), so prior versions
   survive.
2. `POST /public/api/v2/jobs/{jobNumber}/rematch`

Rematch recomputes rather than destroys, and is safe to re-run. It is the one write operation in
this flow an agent can take on its own initiative.

## Step 4 — Resolve

- **Approve** — `POST /public/api/v2/jobs/{jobNumber}/approve`. Gate behind human confirmation.
- **Change status** — `POST /public/api/v2/jobs/{jobNumber}/status`. The permitted transition matrix
  is unpublished; read the current `status` first and do not assume a transition is legal.
- **Delete** — `PATCH /public/api/v2/jobs/{jobNumber}/delete`. Note the verb is `PATCH`, not
  `DELETE`. No window is documented.

## Constraints that apply to every call here

- **No idempotency key exists.** `approve`, `status` and `delete` have no replay protection. Never
  retry them blind; re-read the job with `GET` and decide from its current state.
- **Errors** are `{"errorCode","errorMessage","key"}` JSON, not RFC 9457 problem+json. A missing
  credential returns **400**, not 401.
- **No rate limits, no 429, no Retry-After** are published.
