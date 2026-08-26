---
name: openenvoy-submit-invoice-for-audit
description: >-
  Submit a supplier invoice to OpenEnvoy for automated audit, attach the baseline documents it
  should be matched against (purchase order, contract, rate sheet, receipt), and close the job so
  the matching engine runs.
api: OpenEnvoy API
generated: '2026-08-26'
method: generated
source: openapi/openenvoy-openapi.json + postman/openenvoy-postman-collection.json
base_url: https://backend.openenvoy.io/public/api/v1
operations:
  - POST /public/api/v1/jobs/create
  - POST /public/api/v1/jobs/{jobNumber}/upload
  - POST /public/api/v1/jobs/{jobNumber}/complete
  - GET /public/api/v1/jobs/{jobNumber}
---

# Submit an invoice for audit

A **job** is one invoice under audit. Creating a job is a three-call sequence, not one call. Do not
treat the create call as the end of the flow — a job that is never completed never gets matched.

## Before you start

Every request needs **both** headers. Sending only one returns HTTP 400.

```
X-CLIENT-ID: {client_id}
Authorization: Bearer {auth_token}
```

Credentials are issued by OpenEnvoy customer success (support@openenvoy.com) to existing customers.
There is no self-service key.

## Step 1 — Initiate the job

`POST /public/api/v1/jobs/create` as `multipart/form-data`:

- `provider_name` (string, required) — the supplier's name.
- `Invoice` (file, required) — the invoice document.

The response is a `Job`. **Persist `job_number` immediately** (example form `OE0001`). Every
subsequent call in this flow is keyed on `job_number`, not on `id`.

> **This call is not idempotent and there is no idempotency key.** Retrying after a timeout creates
> a second job for the same invoice — a duplicate, which is exactly what this product exists to
> prevent. If a create call fails ambiguously, do **not** blindly retry: recover instead (see
> Failure handling).

## Step 2 — Attach baseline documents

For each document the invoice should be matched against:

`POST /public/api/v1/jobs/{jobNumber}/upload` as `multipart/form-data`, field `Baseline` (file).

Returns a `JobDocument`. Repeat per document. Skip this step only if the invoice is to be audited
with no baseline.

## Step 3 — Complete creation

`POST /public/api/v1/jobs/{jobNumber}/complete`

Returns the `Job`. This is what releases the job to the matching engine. Until it is called, no
matching happens.

## Step 4 — Read the result

`GET /public/api/v1/jobs/{jobNumber}`

Poll this. `status` moves through values including `Matching`; the full status enumeration is not
published, so treat any unrecognised status as non-terminal rather than erroring.

Read the audit result from `job_documents[].jobdocumentversions[].matching_info[]`, which pairs each
charge against its matched baseline:

- `invoice_amount` / `invoice_price` / `invoice_quantity` / `invoice_unit` / `invoice_currency`
- `matched_amount` / `matched_price` / `matched_quantity` / `matched_unit` / `matched_currency`
- `exchange_rate`, `charge_type`

A discrepancy is a row where the invoice side exceeds the matched side. Job-level totals are
`amount` vs `matched_amount`, with `matched_amount_status` as the summary flag.

Extracted invoice content is on `invoice_info` (`number`, `date`, `due_date`, `payment_terms`,
`amount_due`, `currency`, `seller`, `buyer`, `line_items`, `metadata`).

## Failure handling

- **400 with `{"errorCode":"E00400","errorMessage":"Invalid/Missing Header","key":"..."}`** — a
  required header is missing. `key` names it. This is the response for a bad or absent credential:
  the API does **not** return 401.
- **400 on create/upload/complete** — generic validation failure; the contract declares no schema
  for it.
- **Recovery after an ambiguous create** — you do not have a `job_number` yet, so you cannot
  `GET` the job. Reconcile against your own invoice identifier before re-creating, or use
  `POST /public/api/v2/jobs` (search) to look for a job already created for that supplier and
  amount. Never re-run create as a plain retry.
- **No rate limits are published.** No `429` is declared and no `RateLimit-*` or `Retry-After`
  headers are returned. Pace yourself conservatively and back off on any 5xx.

## Reversal

A created job can be deleted with `PATCH /public/api/v2/jobs/{jobNumber}/delete` (note: **v2**).
No time window for deletion is documented. Whether the delete is soft (the `Job.active` boolean
suggests it may be) or hard is not stated publicly.
