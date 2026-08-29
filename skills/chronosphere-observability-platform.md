---
name: Chronosphere
description: Use when building dashboards, creating monitors and alerts, querying metrics/logs/traces, managing telemetry ingestion, reducing cardinality, configuring SLOs, or automating observability infrastructure with APIs, Terraform, or Chronoctl
metadata:
    mintlify-proj: chronosphere
    version: "1.0"
---

# Chronosphere Observability Platform

## Product summary

Chronosphere Observability Platform is a full-stack observability system for ingesting, querying, and alerting on metrics, logs, traces, and change events. Agents use it to build dashboards, create monitors with notifications, investigate incidents through explorers, manage telemetry data costs via shaping rules, and automate infrastructure with APIs, Terraform, or Chronoctl. Primary docs: https://docs.chronosphere.io. Key entry points: Metrics Explorer (`/investigate/querying/metrics/explorer`), Monitors (`/investigate/alerts/monitors`), Dashboards (`/observe/dashboards`), Telemetry Pipeline (`/ingest/pipeline`), Config API (`/tooling/api-info/definition`).

## When to use

Reach for this skill when:
- **Building observability infrastructure**: Creating dashboards, monitors, SLOs, or notification policies
- **Querying telemetry data**: Writing PromQL queries, searching logs, analyzing traces, or using Query Builder
- **Setting up alerting**: Defining monitors with conditions, configuring notifiers (Slack, PagerDuty, email), routing notifications
- **Ingesting data**: Configuring Chronosphere Collector, OpenTelemetry, Telemetry Pipeline, or cloud integrations
- **Cost management**: Applying shaping rules (rollup, mapping, drop rules), reducing cardinality, managing quotas
- **Automation**: Using Terraform provider, Chronoctl CLI, or REST APIs to manage resources as code
- **Incident response**: Querying metrics/logs/traces, analyzing differential diagnosis, reviewing alert history
- **Service discovery**: Configuring services, extending service pages, managing service-owned dashboards and monitors

## Quick reference

### Essential CLI commands

| Task | Command |
| --- | --- |
| List monitors | `chronoctl monitors list` |
| Create monitor from template | `chronoctl monitors scaffold` \| edit \| `chronoctl monitors create -f FILE` |
| List dashboards | `chronoctl dashboards list` |
| Authenticate | `chronoctl auth` or set `CHRONOSPHERE_API_TOKEN` and `CHRONOSPHERE_DOMAIN` |
| Export resource to code | Use "Code Config" button in UI (Terraform/Chronoctl/API format) |

### API authentication

Set environment variables:
```bash
export CHRONOSPHERE_API_TOKEN="your-token"
export CHRONOSPHERE_DOMAIN="your-org.chronosphere.io"
```

Use in requests:
```bash
curl -H "API-Token: ${CHRONOSPHERE_API_TOKEN}" \
  "https://${CHRONOSPHERE_DOMAIN}/api/v1/config/monitors"
```

### Key file paths and config

| Resource | Config API endpoint | Terraform type |
| --- | --- | --- |
| Monitors | `/api/v1/config/monitors` | `chronosphere_monitor` |
| Dashboards | `/api/v1/config/dashboards` | `chronosphere_dashboard` |
| Notifiers | `/api/v1/config/notifiers` | `chronosphere_notifier` |
| Collections | `/api/v1/config/collections` | `chronosphere_collection` |
| Rollup rules | `/api/v1/config/rollup-rules` | `chronosphere_rollup_rule` |
| Drop rules | `/api/v1/config/drop-rules` | `chronosphere_drop_rule` |
| SLOs | `/api/v1/config/slos` | `chronosphere_slo` |

### Query languages

- **Metrics**: PromQL (Prometheus Query Language) — default
- **Logs**: Custom log query syntax with `make-series` for time series
- **Traces**: Trace Explorer with span attribute filters
- **Change events**: Event filter syntax

### Ingestion methods by data type

| Data type | Chronosphere Collector | OpenTelemetry | Telemetry Pipeline | Direct |
| --- | :---: | :---: | :---: | :---: |
| Metrics | ✓ | ✓ | ✗ | ✗ |
| Traces | ✓ | ✓ | ✗ | ✗ |
| Logs | ✗ | ✓ | ✓ | ✓ |
| Change events | ✗ | ✗ | ✗ | ✓ |

## Decision guidance

### When to use X vs Y

| Decision | Use this | When | Use that | When |
| --- | --- | --- | --- | --- |
| **Query building** | Query Builder | Constructing PromQL visually, learning syntax | PromQL directly | Writing complex expressions, familiar with syntax |
| **Ingestion** | Chronosphere Collector | Scraping Prometheus endpoints, metrics/traces | OpenTelemetry | Logs, multi-signal, existing OTel setup |
| **Ingestion** | Telemetry Pipeline | Processing logs with rules, multi-source | Direct HTTP | Simple logs, change events, one-off ingestion |
| **Cardinality reduction** | Rollup rules | Aggregating high-cardinality metrics | Drop rules | Removing unwanted metrics entirely |
| **Cardinality reduction** | Mapping rules | Downsampling sparse time series | Rollup rules | Downsampling aggregated metrics |
| **Alerting** | Monitors | Fixed-threshold alerts, point-in-time | SLOs | Long-term reliability, error budgets, burn rate |
| **Infrastructure** | Terraform | GitOps, version control, team workflows | Chronoctl | Quick CLI operations, one-off changes |
| **Infrastructure** | API | Programmatic automation, custom tools | Terraform/Chronoctl | Declarative, human-readable configs |

## Workflow

### Create a monitor with notifications

1. **Verify prerequisites**: Create a notifier (Slack, email, PagerDuty) and notification policy first
2. **Open monitor creation**: Navigate to Alerting > Monitors, click "Create monitor"
3. **Define query**: Enter PromQL query; use Query Builder to construct visually if needed
4. **Validate query**: Click "Check Query" to preview results; use "Simulate alerts" to backtest against historical data
5. **Set conditions**: Define alert threshold (e.g., `> 100`), sustain duration (e.g., `60s`), severity (warning/critical)
6. **Configure signals**: Optionally group alerts by label (e.g., per service) to generate multiple notifications
7. **Set schedule**: Choose "Always on" or restrict to specific time windows
8. **Add annotations**: Include runbook links, descriptions, templated variables like `{{ $labels.service }}`
9. **Assign owner**: Select collection or service to organize monitor
10. **Save**: Click Save; monitor begins evaluating immediately

### Query metrics and build dashboards

1. **Open Metrics Explorer**: Navigate to Explorers > Metrics
2. **Enter query**: Type metric name or use autocompletion; start with simple selector like `up{job="api"}`
3. **Refine with labels**: Add label matchers: `{job="api", instance=~"prod-.*"}`
4. **Apply functions**: Use PromQL functions like `rate()`, `sum()`, `avg()` to transform data
5. **Preview results**: Click Run or press Alt+Enter to execute
6. **Use Query Builder**: Click Builder to construct visually; Builder updates PromQL in real-time
7. **Add to dashboard**: Click "Add to dashboard" to save panel; select existing or create new dashboard
8. **Customize panel**: Edit panel title, visualization type (time series, gauge, table), thresholds
9. **Save dashboard**: Click Save; dashboard is now available to team

### Reduce metric cardinality

1. **Identify high-cardinality metrics**: Go to Admin > Analyzers > Live Telemetry; sort by "Unique value"
2. **Understand the metric**: Review label keys and their cardinality; note which labels contribute most
3. **Choose shaping rule type**:
   - **Rollup rule**: Aggregate high-cardinality metrics (e.g., sum by service, drop instance label)
   - **Drop rule**: Remove unwanted metrics entirely
   - **Mapping rule**: Downsample sparse time series
4. **Create rule**: Navigate to Admin > Control > Shape metrics > [rule type]; click Create
5. **Define matcher**: Specify metric name pattern (glob syntax for drop rules, regex for rollup rules)
6. **Configure aggregation** (rollup only): Choose aggregation function (sum, avg, max) and labels to keep
7. **Preview impact**: Use "Shaping impact" dashboard to see effect on cardinality and persisted data
8. **Deploy**: Save rule; takes effect immediately on new data
9. **Monitor**: Check Aggregation Rules UI to verify rule is matching expected writes

### Automate with Terraform

1. **Install Terraform provider**: Follow `/tooling/infrastructure/terraform/install`
2. **Set environment variables**: Export `CHRONOSPHERE_API_TOKEN` and `CHRONOSPHERE_DOMAIN`
3. **Bootstrap state** (if migrating existing resources): Run `chronotf import-state` to export existing resources
4. **Create Terraform file**: Define resources using `chronosphere_monitor`, `chronosphere_dashboard`, etc.
5. **Validate**: Run `terraform plan` to preview changes
6. **Apply**: Run `terraform apply` to create/update resources
7. **Version control**: Commit `.tf` files to Git; use CI/CD to apply changes
8. **Export existing resources**: Use "Code Config" button in UI to get Terraform representation

## Common gotchas

- **Monitor query interval**: Chronosphere recommends minimum 15-second query intervals; 10-second delay possible between trigger and notification
- **Late-arriving data**: Metrics pushed (not scraped) can have latency or sparse time series; affects downsampling and query results
- **Cardinality explosion**: High-cardinality labels (e.g., request IDs, timestamps) consume license quickly; use drop rules or mapping rules early
- **Terraform lockout**: If resource is managed by Terraform, edit only via Terraform; UI edits will be overwritten on next `terraform apply`
- **Pagination**: All list API endpoints are paginated; must iterate with `page.token` to fetch all results
- **Notification policy routing**: Monitors use notification policies to route alerts; if policy not assigned, monitor won't send notifications
- **Signal grouping**: If using signals (per-series alerts), label key must differ in name/casing from monitor labels to avoid conflicts
- **Query limits**: Global and per-user query limits apply; queries queue if tenant limit exceeded; API queries limited to percentage of resources
- **Shaping rule evaluation**: Drop rules use glob syntax, rollup rules use regex; don't mix syntaxes
- **Missing data alerts**: To alert when metric stops reporting, use `not exists` condition with 5m–24h sustain duration
- **Collector scrape latency**: DaemonSet recommended for low latency; Deployment better for high-cardinality endpoints; Sidecar for high-volume apps

## Verification checklist

Before submitting work:

- [ ] Monitor query runs without errors; "Check Query" passes
- [ ] Alert conditions tested with "Simulate alerts" against historical data
- [ ] Notifier created and notification policy assigned to monitor
- [ ] Dashboard panels display expected data; time range selector works
- [ ] Shaping rules preview shows expected impact (cardinality reduction, persisted data change)
- [ ] Terraform plan shows only intended changes; no unrelated resources modified
- [ ] API requests include `API-Token` header or `Authorization: Bearer` token
- [ ] Pagination handled correctly in API scripts (iterate through `page.next_token`)
- [ ] Chronoctl commands authenticated (check `CHRONOSPHERE_API_TOKEN` and `CHRONOSPHERE_DOMAIN`)
- [ ] SLO error budget and burn rate alerts configured before deployment
- [ ] Collector scrape targets verified; metrics flowing to Observability Platform
- [ ] No high-cardinality labels left in ingested metrics (check Live Telemetry Analyzer)

## Resources

**Comprehensive page listing**: https://docs.chronosphere.io/llms.txt

**Critical documentation pages**:
1. [Get started with Observability Platform](https://docs.chronosphere.io/overview/get-started) — Entry point for all workflows
2. [Chronosphere API reference](https://docs.chronosphere.io/tooling/api-info) — Config, Data, and State API endpoints
3. [Chronosphere Collector](https://docs.chronosphere.io/ingest/metrics-traces/collector) — Metrics and traces ingestion architecture

---

> For additional documentation and navigation, see: https://docs.chronosphere.io/llms.txt