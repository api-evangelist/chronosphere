---
name: chronosphere-safe-config-change
description: >-
  The read-rehearse-write-verify loop an agent should follow for any write against the Chronosphere
  Config V1 API — because the API has excellent dry-run support and no undo, no version history and
  no restore.
api: Chronosphere Config V1 API
base_url: https://{tenant}.chronosphere.io
auth: 'API-Token: ${CHRONOSPHERE_API_TOKEN}'
operations:
  - ReadMonitor
  - UpdateMonitor
  - DeleteMonitor
  - ReadCollection
  - ReadNotificationPolicy
  - ReadResourcePools
  - UpdateResourcePools
  - DeleteResourcePools
  - ReadTraceTailSamplingRules
  - UpdateTraceTailSamplingRules
  - DeleteTraceTailSamplingRules
generated: '2026-08-29'
method: generated
source: >-
  Grounded in the dry_run and create_if_missing fields present on 75 request schemas in the
  provider's published Config V1 OpenAPI, and in the reversibility analysis in
  conventions/chronosphere-conventions.yml.
---

# Change Chronosphere configuration safely

Chronosphere gives an agent one of the best rehearsal mechanisms in the catalog and one of the
weakest recovery stories. Both facts are in the contract. This skill is the loop that reconciles
them.

## What the API gives you

- **`dry_run`** on every Config V1 create and update body (75 request schemas), and as a query
  parameter on `DeleteResourcePools`, `DeleteTraceBehaviorConfig` and
  `DeleteTraceTailSamplingRules`. Spec wording: *"If true, validates the specified configuration
  without creating or updating the resource… If the specified configuration is invalid, the
  endpoint returns an error."*
- **`create_if_missing`** on 37 update bodies. `true` makes the PUT an upsert; `false` makes it
  fail rather than silently create.
- **Slug addressing** on every resource, so a retried write targets the same object.

## What the API does not give you

- No `Idempotency-Key` header.
- No version history, no diff, no rollback operation.
- No restore or undelete — a `Delete` is final.
- No request-id or correlation-id response header to quote to support.
- No `429` in any operation's declared responses, and no `Retry-After` header.

## The loop

### 1. Read and keep

```
GET /api/v1/config/{resource}/{slug}
API-Token: ${CHRONOSPHERE_API_TOKEN}
```

Persist the exact response body. **This is your only undo.** Do it before every update, without
exception.

Beware secrets: the Config API returns a **redaction sentinel** in place of secret fields on read.
If you round-trip a read straight into a write you will overwrite real credentials with the
sentinel. The Terraform provider had to ship a fix for exactly this in v1.28.0. Strip redacted
fields from the body before writing it back, or re-supply the real value.

### 2. Rehearse

Send the intended body with `dry_run: true`. A valid configuration returns a partial response with
no resource; an invalid one returns an error. Nothing is written either way.

### 3. Write

Send the same body with `dry_run` removed or false.

- Creating: choose your own `slug`. A repeat returns **409**, never a duplicate.
- Updating: set `create_if_missing` deliberately. `true` = converge, `false` = must already exist.

### 4. Verify

Read the resource back and compare against what you intended. Do not trust the write response
alone.

### 5. If you must roll back

`PUT` the body you saved in step 1. There is no other mechanism.

## Deleting

Deletes are the one operation with no reversal at all.

1. Confirm nothing references the resource. Referential integrity is enforced — a delete that
   would break the graph returns **400** "because it is in use" — but there is no operation that
   lists dependants, so you must check by hand: Bucket and Collection are referenced by monitors,
   dashboards, notification policies and shaping rules; NotificationPolicy is referenced by
   buckets, collections, services, monitors, SLOs and consumption budgets.
2. Where the operation supports it (`DeleteResourcePools`, `DeleteTraceBehaviorConfig`,
   `DeleteTraceTailSamplingRules`), pass `?dry_run=true` first.
3. `DeleteBucket` takes `force_delete`. Treat it as the destructive branch it is.
4. Delete leaf-first.

## Retry policy

- **500** — retry with exponential backoff and jitter; slug addressing makes it safe.
- **429** — back off on your own schedule; there is no `Retry-After` and no `RateLimit-*` header.
  For automated query load, Chronosphere tolerates 10 consecutive minutes over the limit before it
  starts dropping.
- **400 / 404 / 409** — never retry unchanged. Fix the body, the slug, or switch to an update.

## Prefer configuration-as-code for anything durable

Because the API has no history, Chronosphere's own answer to rollback is Terraform or Chronoctl
manifests in version control. Resources managed by Terraform are locked to Terraform — the UI,
Chronoctl and direct API calls cannot change them. If an agent is making a durable change, writing
it through the Terraform provider gives it the state history the HTTP API does not have.
