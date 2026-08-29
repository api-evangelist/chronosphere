---
name: chronosphere-metrics-cost-control
description: >-
  Find the metrics and labels driving Chronosphere spend, then shed them with drop, rollup and
  mapping rules — reading usage from the State V1 API and writing shaping rules through Config V1,
  with dry_run on every write.
api: Chronosphere State V1 API, Chronosphere Config V1 API
base_url: https://{tenant}.chronosphere.io
auth: 'API-Token: ${CHRONOSPHERE_API_TOKEN}'
operations:
  - ListMetricUsagesByMetricName
  - ListMetricUsagesByLabelName
  - ListRuleEvaluations
  - ListDropRules
  - CreateDropRule
  - ListRollupRules
  - CreateRollupRule
  - ListMappingRules
  - CreateMappingRule
  - ReadResourcePools
  - UpdateResourcePools
  - ListConsumptionBudgets
  - CreateConsumptionBudget
generated: '2026-08-29'
method: generated
source: >-
  Grounded in operationIds grepped from the provider's published Config V1 and State V1 OpenAPI
  documents at https://docs.chronosphere.io/openapi/, and the licensing model documented at
  https://docs.chronosphere.io/administer/limits-licensing/licensing.
---

# Cut Chronosphere metrics cost

Chronosphere charges against contracted license dimensions — persisted writes, persisted
cardinality and aggregations, split between a Standard Metrics License and a Histogram Metrics
License. Exceeding capacity does not bill more; it **drops data**, and quotas decide whose data
drops first. So cost control here is a reliability job, not just a finance one.

## 1. Measure before you cut

State V1 is read-only and exists for exactly this.

- `ListMetricUsagesByMetricName` — `GET /api/v1/state/metric-usages-by-metric-name`
  Finds unused or underutilised metrics. This is the list to drop from.
- `ListMetricUsagesByLabelName` — `GET /api/v1/state/metric-usages-by-label-name`
  Finds high-cardinality labels. Cardinality is usually the real cost, not metric count.
- `ListRuleEvaluations` — `GET /api/v1/state/rule-evaluations`
  Surfaces monitors and recording rules that are failing or misbehaving — often the same rules
  driving selectors-per-second load.

All three paginate with `page.max_size` / `page.token`.

## 2. Pick the right instrument

| Goal | Resource | Operation |
| --- | --- | --- |
| Stop ingesting a metric or series entirely | DropRule | `CreateDropRule` |
| Keep the metric but at lower resolution or fewer labels | RollupRule | `CreateRollupRule` |
| Change where a metric lands and how long it is kept | MappingRule | `CreateMappingRule` |
| Reserve capacity per team so the noisy tenant drops first | ResourcePools | `UpdateResourcePools` |
| Get alerted before you hit the ceiling | ConsumptionBudget | `CreateConsumptionBudget` |

RollupRule and MappingRule both require a `bucket_slug`.

## 3. Write the rule — dry run first

Every create and update body in Config V1 accepts `dry_run`. Send the rule with `dry_run: true`,
confirm it validates, then send it again without.

```
POST /api/v1/config/drop-rules
API-Token: ${CHRONOSPHERE_API_TOKEN}

{ "drop_rule": { "slug": "drop-noisy-debug-metric", ... }, "dry_run": true }
```

A drop rule takes effect on **ingest**. Data already stored is unaffected, and data dropped is not
recoverable — there is no restore operation anywhere in the Config API. Dry-run is the only
rehearsal you get.

## 4. Budget and alert

`CreateConsumptionBudget` — `POST /api/v1/config/consumption-budgets`

Set `notification_policy_slug` if you use an `ALERT_*` threshold action; it is required in that
case. Threshold `resource_group` values include the LOG_ALL, TRACE_PROCESSED_BYTES,
TRACE_PERSISTED_BYTES, TRACE_ALL and ALL groups. Do not use `sku_group` — it was deprecated and
removed on the Terraform surface in June 2026.

## 5. Re-measure

Re-run step 1 after the rules take effect. Usage is the only feedback loop; there is no public
API that returns a price.

## Undoing

Every rule created here is reversible by `Delete<Rule>` with no stated window, and updatable back
to a saved body. **Data the rule dropped in the meantime is not reversible.** Read the rule with
`Read<Rule>` and keep the body before every update.

## Watch out

- Deleting a rule that something else references returns **400** "because it is in use".
- Automated query load has its own ceiling: exceeding selectors-per-second or data-reads-per-second
  for 10 consecutive minutes returns **429** and Chronosphere then drops an indiscriminate subset
  of queries. The selectors-per-second limit cannot be raised. There is no `Retry-After` header.
- Prometheus and OpenTelemetry histograms bill against different licenses. Check which one your
  metric type lands in before assuming a rollup helps.
