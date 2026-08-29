---
name: chronosphere-alerting-setup
description: >-
  Stand up an end-to-end alert path in Chronosphere Observability Platform — a notifier, a
  notification policy that routes to it, and a monitor that fires into that policy — using the
  Config V1 API, validating every write with dry_run before committing it.
api: Chronosphere Config V1 API
base_url: https://{tenant}.chronosphere.io
auth: 'API-Token: ${CHRONOSPHERE_API_TOKEN}'
operations:
  - ListTeams
  - ListCollections
  - ListNotifiers
  - CreateNotifier
  - ListNotificationPolicies
  - CreateNotificationPolicy
  - ListMonitors
  - CreateMonitor
  - ReadMonitor
  - UpdateMonitor
  - DeleteMonitor
generated: '2026-08-29'
method: generated
source: >-
  Grounded in operationIds grepped from the provider's published Config V1 OpenAPI at
  https://docs.chronosphere.io/openapi/api_v1_config_openapi3_DOCUMENTATION_ONLY.json and the
  conventions recorded in conventions/chronosphere-conventions.yml.
---

# Set up an alert path in Chronosphere

Alerting in Chronosphere is three resources, created in dependency order: a **Notifier** (where a
notification goes), a **NotificationPolicy** (which alerts route to which notifier), and a
**Monitor** (what condition fires). Build them bottom-up.

## Before you start

- Base URL is your tenant subdomain: `https://${CHRONOSPHERE_DOMAIN}` where `CHRONOSPHERE_DOMAIN`
  is `your-org.chronosphere.io`.
- Every request carries `API-Token: ${CHRONOSPHERE_API_TOKEN}`. Note the header name — it is
  **not** `Authorization`. Only the Prometheus query API uses a bearer token.
- Choose your own `slug` for every resource you create. A server-generated slug makes a retry
  unsafe, because there is nothing to collide on.

## 1. Find the scope

A monitor must belong to either a Bucket or a Collection, and a notification policy to either a
Bucket or a Team. Resolve them first.

- `ListTeams` — `GET /api/v1/config/teams`
- `ListCollections` — `GET /api/v1/config/collections`

Both paginate with `page.max_size` and `page.token`; follow `page.next_token` until it is empty.

## 2. Create the notifier

`CreateNotifier` — `POST /api/v1/config/notifiers`

Set `slug`, `name`, and the type-specific block (`webhook`, `slack`, `pagerduty`, `opsgenie`,
`email`, `victorops`). Set `skip_resolved: true` if you do not want a second callback when the
alert clears.

Send it once with `dry_run: true` first. The API validates the whole body and returns an error if
it is wrong, without creating anything.

If a notifier with that slug already exists you get **409**, not a duplicate. That is the retry
guarantee — a repeated create is safe.

## 3. Create the notification policy

`CreateNotificationPolicy` — `POST /api/v1/config/notification-policies`

Set `bucket_slug` **or** `team_slug` — exactly one is required, and the contract expresses this
only in prose, so a schema validator will not catch the mistake. Route rules reference the
notifier slug from step 2.

Again: `dry_run: true` first.

## 4. Create the monitor

`CreateMonitor` — `POST /api/v1/config/monitors`

- `bucket_slug` **or** `collection_slug` — one is required.
- `notification_policy_slug` — the policy from step 3. If omitted, the monitor inherits the
  default policy of its bucket or collection.
- The condition is PromQL.

## 5. Verify

- `ReadMonitor` — `GET /api/v1/config/monitors/{slug}`
- `ListMonitors` — `GET /api/v1/config/monitors?slugs=<slug>`

## Changing a monitor later

`UpdateMonitor` — `PUT /api/v1/config/monitors/{slug}`

Set `create_if_missing: false` when you mean "this must already exist" and `true` when you want an
upsert. With `create_if_missing: true` the PUT is idempotent: repeating it converges on the same
state.

**Read before you write.** `ReadMonitor` and keep the body. Chronosphere stores no prior version
and exposes no rollback, diff or history operation — that saved body is your only undo.

## Undoing

| What you did | How to undo it | Window |
| --- | --- | --- |
| Created a monitor | `DeleteMonitor` | none stated |
| Updated a monitor | `UpdateMonitor` with the body you saved | none stated |
| Deleted a monitor | nothing — there is no restore operation | — |

## Errors you will actually hit

- **400** — invalid body, or on delete "because it is in use": something still references the
  resource. Delete leaf-first; there is no cascade.
- **404** — the slug does not exist for that resource type.
- **409** — slug conflict. Read the existing resource and update it instead.
- **429** — query-side throttling. Undeclared in the spec, and there is **no** `Retry-After`
  header. Back off exponentially with jitter on your own schedule.
- **500** — retry with backoff; slug-addressed writes make the retry safe.

The error body is `application/json` with no declared fields. Do not write a parser that depends
on a `code` or `message` key being present.

## Teardown order

Monitor → NotificationPolicy → Notifier. Referential integrity is enforced server-side, so
deleting out of order returns 400.
