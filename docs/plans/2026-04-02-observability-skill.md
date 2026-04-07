# Observability Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an `observability` skill with two modes (author and investigate) for query authoring, alert/dashboard creation, and active incident investigation.

**Architecture:** Single skill with SKILL.md defining modes and workflow, reference docs for tool syntax (PromQL, LogQL, TraceQL, alerting, dashboards, investigation, alternatives), and evals for both modes. Grafana conventions drawn from harumi-k8s grafana-dashboards skill. Config-adaptive via `.devops.yaml` with deep Prometheus/Grafana/Loki/Tempo coverage and light Datadog/CloudWatch/X-Ray support.

**Tech Stack:** SKILL.md (YAML frontmatter + Markdown), reference docs (Markdown), JSON evals

**Spec:** `docs/specs/2026-04-02-observability-skill-design.md`

---

## File Map

**New files:**

| File | Responsibility |
|------|---------------|
| `skills/observability/SKILL.md` | Core skill: modes, workflow, safety rules, config reading |
| `skills/observability/references/promql.md` | PromQL syntax, functions, common patterns |
| `skills/observability/references/logql.md` | LogQL syntax, parsers, metric queries |
| `skills/observability/references/traceql.md` | TraceQL syntax, span filtering |
| `skills/observability/references/alerting.md` | Alert rule best practices, USE/RED templates |
| `skills/observability/references/dashboards.md` | Grafana JSON structure, panels, grid, thresholds |
| `skills/observability/references/deployment.md` | ConfigMap creation, kubectl deploy, verification |
| `skills/observability/references/investigation.md` | USE/RED methodology, correlation patterns, triage |
| `skills/observability/evals/evals.json` | Test cases for author and investigate modes |

**Modified files:**

| File | Change |
|------|--------|
| `skills/using-devops/SKILL.md` | Move observability from future to active skills, add trigger rules |

---

### Task 1: Create core SKILL.md

**Files:**
- Create: `skills/observability/SKILL.md`

- [ ] **Step 1: Create SKILL.md**

Write `skills/observability/SKILL.md`:

```markdown
---
name: observability
description: "Query authoring and active incident investigation for observability stacks. Use when: (1) Writing PromQL/LogQL/TraceQL queries, (2) Creating Prometheus alert rules, (3) Building Grafana dashboards, (4) Investigating incidents — debugging latency, errors, crashes, (5) Any monitoring, alerting, SLO, or dashboard task."
---

# Observability

Act as a **Senior SRE / Observability Engineer**. Read the active `.devops.yaml` config (injected at session start) for the observability stack: metrics, dashboards, logs, traces.

## Mode Detection

This skill has two modes. Detect from user intent:

| Mode | Keywords | Purpose |
|------|----------|---------|
| **Author** | "write", "create", "build", "query for", "alert when", "dashboard" | Write queries, alert rules, dashboards |
| **Investigate** | "investigate", "debug", "check", "what's happening", "incident", "why is X slow/down" | Active incident investigation |

If ambiguous, ask the user which mode they need.

## Author Mode

### Workflow

1. **Read context** — Check `.devops.yaml` for stack, scan `docs/references/` for service/architecture context
2. **Understand intent** — What metric/log/trace? What threshold? What service?
3. **Write artifact** — Query, alert rule YAML, or dashboard JSON
4. **Validate** — Run `promtool check rules` for alert rules, JSON lint for dashboards if available
5. **Export** — Write to file in the appropriate repo location, offer to commit

### Stack Depth

| Stack | Depth | Reference |
|-------|-------|-----------|
| Prometheus | Deep | [references/promql.md](references/promql.md) |
| Grafana | Deep | [references/dashboards.md](references/dashboards.md), [references/deployment.md](references/deployment.md) |
| Loki | Deep | [references/logql.md](references/logql.md) |
| Tempo | Deep | [references/traceql.md](references/traceql.md) |

### Alert Rules

- Suggest thresholds based on USE method (utilization, saturation, errors) or RED method (rate, errors, duration)
- Include `for` duration, labels, annotations with runbook links
- Validate with `promtool check rules [file]`
- See [references/alerting.md](references/alerting.md) for templates and best practices

### Grafana Dashboards

- Search [grafana.com/grafana/dashboards/](https://grafana.com/grafana/dashboards/) for existing templates first
- Dashboard UID pattern: `harumi-[category]-[name]`
- Datasource: always `type: "prometheus"`, `uid: "prometheus"`
- 24-column grid layout (quarters w:6, thirds w:8, halves w:12, full w:24)
- Generate both `[name].json` and `[name].configmap.yaml`
- ConfigMap labels: `grafana_dashboard: "1"`, `app.kubernetes.io/name: grafana`, `app.kubernetes.io/component: dashboard`
- Schema version 38, 30s refresh, dark style
- See [references/dashboards.md](references/dashboards.md) for panel types and JSON structure
- See [references/deployment.md](references/deployment.md) for kubectl deploy and verification

### Artifact Export

Auto-detect paths from repo structure, with fallbacks:

| Artifact | Discovery | Fallback |
|----------|-----------|----------|
| Alert rules | Grep for existing `groups:` YAML files | `monitoring/alerts/` |
| Recording rules | Grep for existing `record:` entries | `monitoring/rules/` |
| Grafana dashboards | Look for existing `grafana-dashboards/` dir | `grafana-dashboards/{env}/{category}/` |
| Dashboard ConfigMaps | Same dir as dashboard JSON | `[dashboard-path]/[name].configmap.yaml` |
| Runbooks | Check `docs/references/` or `docs/runbooks/` | `docs/runbooks/` |

After writing artifacts:
1. Validate if tooling exists (`promtool check rules`, `jq` for JSON)
2. Show diff summary to user
3. Offer to commit — user confirms with "yes"
4. If K8s deployment needed, provide `kubectl apply` handoff (never apply without confirmation)

## Investigate Mode

### Methodology

USE/RED framework as the investigation backbone:
- **USE** (infrastructure): Utilization, Saturation, Errors — for nodes, disks, network
- **RED** (services): Rate, Errors, Duration — for application endpoints

See [references/investigation.md](references/investigation.md) for detailed methodology and correlation patterns.

### Workflow

1. **Triage** — Classify the problem: latency, errors, saturation, crash, connectivity
2. **Read context** — Check `.devops.yaml` for stack, scan `docs/references/` for service topology and runbooks
3. **Gather signals** — Run read-only CLI commands across three pillars (metrics, logs, traces)
4. **Correlate** — Connect metrics anomalies to log patterns to trace spans. Present a timeline.
5. **Diagnose** — Propose root cause with supporting evidence
6. **Recommend** — Suggest fix or mitigation. Destructive actions require explicit user confirmation.

### CLI Tools

Priority order:
1. Dedicated CLI if available (`promtool`, `logcli`, `amtool`)
2. Cloud provider CLI (`aws`, `gcloud`, `az`)
3. `kubectl` for K8s-level investigation
4. `curl` against HTTP APIs as fallback

#### Deep Stack (Prometheus/Loki/Tempo)

| Pillar | Commands |
|--------|----------|
| Metrics | `promtool query instant/range` against Prometheus API, `curl` Prometheus HTTP API |
| Logs | `logcli query` against Loki, `logcli series` |
| Traces | `curl` Tempo HTTP API (`/api/traces/{traceID}`, `/api/search`) |
| K8s | `kubectl logs`, `kubectl top`, `kubectl get events`, `kubectl describe` |


### Safety Rules (NON-NEGOTIABLE)

1. **All investigation commands are read-only** — no restarts, no scaling, no deletes without user saying "yes"
2. **Always state what command you're about to run and why** before running it
3. **If a command requires credentials or access the user hasn't set up**, ask rather than assume

## Reference Documentation

Consult these based on the task:

- **[references/promql.md](references/promql.md)** — PromQL syntax, functions, common patterns
- **[references/logql.md](references/logql.md)** — LogQL syntax, parsers, metric queries
- **[references/traceql.md](references/traceql.md)** — TraceQL syntax, span filtering
- **[references/alerting.md](references/alerting.md)** — Alert rule best practices, USE/RED templates
- **[references/dashboards.md](references/dashboards.md)** — Grafana JSON structure, panels, grid, thresholds
- **[references/deployment.md](references/deployment.md)** — ConfigMap creation, kubectl deploy, verification
- **[references/investigation.md](references/investigation.md)** — USE/RED methodology, correlation patterns, triage flowchart
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/SKILL.md
git commit -m "feat: add observability skill core SKILL.md"
```

---

### Task 2: Create references/promql.md

**Files:**
- Create: `skills/observability/references/promql.md`

- [ ] **Step 1: Create promql.md**

Write `skills/observability/references/promql.md`. Content drawn from harumi-k8s grafana-dashboards `references/queries.md` and expanded with general PromQL reference:

```markdown
# PromQL Reference

PromQL syntax, functions, and common patterns. Read this when writing or debugging Prometheus queries.

## Query Structure

```promql
metric_name{label="value", label2=~"regex.*"}[time_range]
```

| Component | Example | Description |
|-----------|---------|-------------|
| Metric name | `kube_pod_info` | The metric to query |
| Label selector | `{namespace="production"}` | Filter by labels |
| Range vector | `[5m]` | Time range for rate/increase |
| Aggregation | `sum()`, `count()` | Combine series |

## Label Selectors

```promql
{namespace="production"}         # Exact match
{namespace=~"production|staging"} # Regex match
{namespace!="kube-system"}        # Negative match
{namespace!~"kube.*"}             # Negative regex
```

## Time Ranges

| Range | Use Case |
|-------|----------|
| `[1m]` | High resolution, noisy |
| `[5m]` | Standard rate calculations |
| `[15m]` | Smoother trends |
| `[1h]` | Hourly aggregations |
| `[24h]` | Daily summaries |

## Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `rate()` | Per-second rate of counter | `rate(http_requests_total[5m])` |
| `increase()` | Total increase over range | `increase(http_requests_total[1h])` |
| `sum()` | Aggregate series | `sum by (namespace) (kube_pod_info)` |
| `count()` | Count series | `count by (phase) (kube_pod_status_phase)` |
| `avg()` | Average | `avg(rate(node_cpu_seconds_total{mode!="idle"}[5m]))` |
| `max()/min()` | Extremes | `max(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)` |
| `histogram_quantile()` | Percentiles from histograms | `histogram_quantile(0.95, sum(rate(bucket[5m])) by (le))` |
| `topk()/bottomk()` | Top/bottom N series | `topk(5, sum by (pod) (container_memory_usage_bytes))` |
| `vector()` | Constant scalar as vector | `sum(metric) or vector(0)` |
| `absent()` | Alert when metric missing | `absent(up{job="api"})` |
| `changes()` | Number of value changes | `changes(process_start_time_seconds[1h])` |
| `delta()` | Difference over range (gauge) | `delta(temperature[1h])` |
| `deriv()` | Per-second derivative (gauge) | `deriv(node_filesystem_avail_bytes[1h])` |
| `predict_linear()` | Linear prediction | `predict_linear(node_filesystem_avail_bytes[6h], 24*3600)` |
| `clamp_min()/clamp_max()` | Clamp values | `clamp_min(free_bytes, 0)` |

## Offset and Subqueries

```promql
# Compare to 24h ago
rate(http_requests_total[5m]) / rate(http_requests_total[5m] offset 24h)

# Max of 5m rate over last hour
max_over_time(rate(http_requests_total[5m])[1h:])

# Average over time
avg_over_time(cpu_usage[1h])
```

## Kubernetes Metrics

### Nodes

```promql
sum(kube_node_status_condition{condition="Ready",status="true"})  # Ready nodes
count by (label_node_kubernetes_io_capacity_type) (kube_node_info) # By type
```

### Pods

```promql
count by (phase) (kube_pod_status_phase)                          # By phase
sum(kube_pod_status_phase{phase="Pending"})                        # Pending
sum by (namespace, pod) (kube_pod_container_status_restarts_total) # Restarts
sum(kube_pod_status_ready{condition="false"})                      # Not ready
```

### Container Resources

```promql
sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})    # CPU requests
sum by (namespace) (kube_pod_container_resource_requests{resource="memory"}) # Memory requests
sum by (namespace) (kube_pod_container_resource_limits{resource="cpu"})      # CPU limits
sum by (namespace) (kube_pod_container_resource_limits{resource="memory"})   # Memory limits
```

### Deployments

```promql
kube_deployment_spec_replicas{namespace="production"}              # Desired
kube_deployment_status_replicas_available{namespace="production"}  # Available
kube_deployment_status_replicas_unavailable{namespace="production"} # Unavailable
```

## Node Exporter Metrics

```promql
# CPU usage %
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage %
100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)

# Disk usage %
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)

# Network rates
rate(node_network_receive_bytes_total{device="eth0"}[5m])
rate(node_network_transmit_bytes_total{device="eth0"}[5m])

# Disk I/O
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])
```

## Application Metrics (RED)

```promql
# Rate
rate(http_requests_total[5m])
sum by (status_code) (rate(http_requests_total[5m]))

# Errors
sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Duration (latency percentiles)
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

## Common Patterns

```promql
# CPU utilization % (requests-based)
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m]))
/ sum(kube_pod_container_resource_requests{namespace="production", resource="cpu"}) * 100

# Error budget (99.9% SLO over 30d)
1 - (sum(rate(http_requests_total{status_code=~"5.."}[30d]))
/ sum(rate(http_requests_total[30d]))) >= 0.999

# Memory saturation by namespace
sum by (namespace) (container_memory_usage_bytes)
/ sum by (namespace) (kube_pod_container_resource_limits{resource="memory"})

# Disk full prediction (24h)
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 24*3600) < 0
```

## Best Practices

1. Always use `rate()` with counters — raw counter values reset and are not useful directly
2. Use appropriate time ranges — 5m for rates, 24h for daily summaries
3. Aggregate before querying — `sum by` reduces cardinality
4. Handle missing data — `or vector(0)` for empty results
5. Manage cardinality — too many labels = slow queries
6. Use `without` instead of `by` when dropping few labels from many
7. Use recording rules for expensive queries that run repeatedly
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/references/promql.md
git commit -m "feat: add PromQL reference for observability skill"
```

---

### Task 3: Create references/logql.md

**Files:**
- Create: `skills/observability/references/logql.md`

- [ ] **Step 1: Create logql.md**

Write `skills/observability/references/logql.md`:

```markdown
# LogQL Reference

LogQL syntax, parsers, and metric queries for Grafana Loki. Read this when writing or debugging log queries.

## Query Types

| Type | Purpose | Example |
|------|---------|---------|
| Log query | Return log lines | `{namespace="production"} \|= "error"` |
| Metric query | Return numeric values from logs | `rate({namespace="production"} \|= "error" [5m])` |

## Log Stream Selectors

```logql
{job="api"}                           # Exact match
{namespace=~"prod.*"}                 # Regex match
{namespace!="kube-system"}            # Negative match
{namespace!~"kube.*"}                 # Negative regex
{namespace="production", container="api"}  # Multiple labels
```

## Line Filters

Applied after stream selector, processed in order:

```logql
{job="api"} |= "error"               # Contains (case-sensitive)
{job="api"} != "healthcheck"          # Does not contain
{job="api"} |~ "error|warn"           # Regex match
{job="api"} !~ "debug|trace"          # Negative regex
```

Chain filters for precision:

```logql
{job="api"} |= "error" != "healthcheck" |~ "timeout|connection"
```

## Parsers

Extract structured fields from log lines:

### JSON parser

```logql
{job="api"} | json
{job="api"} | json | level="error"
{job="api"} | json | status >= 500
{job="api"} | json first_name="João"      # Nested: {"first_name": "João"}
```

### Logfmt parser

```logql
{job="api"} | logfmt
{job="api"} | logfmt | level="error" | duration > 5s
```

### Pattern parser

```logql
# Apache common log format
{job="nginx"} | pattern `<ip> - - [<timestamp>] "<method> <path> <_>" <status> <size>`
{job="nginx"} | pattern `<ip> - - [<timestamp>] "<method> <path> <_>" <status> <size>` | status >= 500
```

### Regex parser

```logql
{job="api"} | regexp `(?P<method>GET|POST|PUT|DELETE) (?P<path>\S+) (?P<status>\d+)`
{job="api"} | regexp `duration=(?P<duration>\d+)ms` | duration > 1000
```

## Label Filter Expressions

After parsing, filter on extracted labels:

```logql
| level = "error"                     # String equality
| status >= 500                       # Numeric comparison
| duration > 5s                       # Duration comparison
| size > 1KB                          # Bytes comparison
| level = "error" or level = "warn"   # OR
| level = "error" and method = "POST" # AND
```

## Formatting

```logql
{job="api"} | json | line_format "{{.method}} {{.path}} {{.status}} {{.duration}}"
{job="api"} | json | label_format duration_seconds="{{divide .duration 1000}}"
```

## Metric Queries

Build metrics from log streams:

```logql
# Log line rate
rate({job="api"}[5m])

# Error rate
rate({job="api"} |= "error"[5m])

# Count over time
count_over_time({job="api"} | json | level="error"[1h])

# Bytes rate
bytes_rate({job="api"}[5m])

# Sum by label
sum by (level) (count_over_time({job="api"} | json [5m]))

# Quantile from parsed duration
quantile_over_time(0.95, {job="api"} | json | unwrap duration [5m]) by (method)

# Average extracted value
avg_over_time({job="api"} | logfmt | unwrap request_time [5m]) by (path)
```

## Aggregation Functions

| Function | Purpose |
|----------|---------|
| `rate()` | Log lines per second |
| `count_over_time()` | Total log lines in range |
| `bytes_rate()` | Bytes per second |
| `bytes_over_time()` | Total bytes in range |
| `sum_over_time()` | Sum of unwrapped values |
| `avg_over_time()` | Average of unwrapped values |
| `min_over_time()/max_over_time()` | Min/max of unwrapped values |
| `quantile_over_time()` | Quantile of unwrapped values |
| `first_over_time()/last_over_time()` | First/last value in range |
| `absent_over_time()` | Returns 1 if no logs in range |

## Common Patterns

```logql
# Error rate by service
sum by (service) (rate({namespace="production"} | json | level="error" [5m]))

# Slow requests (>1s)
{job="api"} | json | duration > 1s | line_format "{{.method}} {{.path}} took {{.duration}}"

# Top error messages
topk(10, sum by (message) (count_over_time({job="api"} | json | level="error" [1h])))

# 5xx responses per path
sum by (path) (count_over_time({job="api"} | json | status >= 500 [1h]))

# Log volume by namespace
sum by (namespace) (bytes_over_time({namespace=~".+"}[1h]))

# Alert: no logs from service for 15m
absent_over_time({job="api"}[15m])
```

## Best Practices

1. Always start with the most selective stream selector — labels are indexed, line filters are not
2. Use line filters (`|=`) before parsers — cheaper to filter raw text than parse everything
3. Chain filters from broadest to narrowest for performance
4. Use `unwrap` to convert extracted labels to numeric values for metric queries
5. Prefer `rate()` over `count_over_time()` for alerting — rate is independent of time range
6. Use `topk()` with `count_over_time()` to find the noisiest log sources
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/references/logql.md
git commit -m "feat: add LogQL reference for observability skill"
```

---

### Task 4: Create references/traceql.md

**Files:**
- Create: `skills/observability/references/traceql.md`

- [ ] **Step 1: Create traceql.md**

Write `skills/observability/references/traceql.md`:

```markdown
# TraceQL Reference

TraceQL syntax for querying traces in Grafana Tempo. Read this when searching traces or correlating with metrics/logs.

## Basic Syntax

```traceql
{ span.attribute = "value" }
```

TraceQL queries select spans. A trace is returned if any of its spans match.

## Span Intrinsics

| Intrinsic | Description | Example |
|-----------|-------------|---------|
| `name` | Span name | `{ name = "HTTP GET" }` |
| `status` | Span status | `{ status = error }` |
| `duration` | Span duration | `{ duration > 1s }` |
| `kind` | Span kind | `{ kind = server }` |
| `rootName` | Root span name | `{ rootName = "POST /api/v1/orders" }` |
| `rootServiceName` | Root span service | `{ rootServiceName = "api" }` |
| `traceDuration` | Full trace duration | `{ traceDuration > 5s }` |

## Resource Attributes

```traceql
{ resource.service.name = "api" }
{ resource.namespace = "production" }
{ resource.k8s.pod.name =~ "api-.*" }
{ resource.deployment.environment = "production" }
```

## Span Attributes

```traceql
{ span.http.method = "POST" }
{ span.http.status_code >= 500 }
{ span.http.url =~ ".*/api/v1/.*" }
{ span.db.system = "postgresql" }
{ span.db.statement =~ "SELECT.*FROM users.*" }
```

## Operators

| Operator | Description |
|----------|-------------|
| `=` | Equal |
| `!=` | Not equal |
| `>`, `>=`, `<`, `<=` | Numeric comparison |
| `=~` | Regex match |
| `!~` | Regex not match |

## Combining Conditions

```traceql
# AND within a span
{ span.http.method = "POST" && span.http.status_code >= 500 }

# OR within a span
{ span.http.status_code = 500 || span.http.status_code = 503 }

# Pipeline: spans from different services in the same trace
{ resource.service.name = "api" } >> { resource.service.name = "database" }

# Sibling: spans at the same level
{ resource.service.name = "cache" } ~ { resource.service.name = "database" }
```

## Structural Operators

| Operator | Meaning |
|----------|---------|
| `>>` | Descendant (child, grandchild, etc.) |
| `>` | Direct child |
| `~` | Sibling |
| `!>` | Not direct child |
| `!>>` | Not descendant |
| `!~` | Not sibling |

## Aggregate Functions

```traceql
# Count spans matching condition
{ resource.service.name = "api" } | count() > 5

# Average duration
{ resource.service.name = "api" } | avg(duration) > 500ms

# Max duration
{ resource.service.name = "api" } | max(duration) > 2s

# Min/sum
{ resource.service.name = "api" } | min(duration), max(duration)
```

## Common Patterns

```traceql
# Slow API calls
{ resource.service.name = "api" && duration > 1s }

# Failed database queries
{ span.db.system = "postgresql" && status = error }

# Traces where API calls database and it's slow
{ resource.service.name = "api" } >> { span.db.system = "postgresql" && duration > 500ms }

# Error traces from a specific endpoint
{ span.http.url =~ ".*/api/v1/users.*" && status = error }

# Long traces (end-to-end)
{ traceDuration > 5s }

# Traces with many spans (fan-out)
{ resource.service.name = "api" } | count() > 20
```

## Tempo HTTP API

When CLI tools are not available, query Tempo directly:

```bash
# Search traces
curl -s "http://tempo:3200/api/search?q={resource.service.name=\"api\" && status=error}&limit=20"

# Get trace by ID
curl -s "http://tempo:3200/api/traces/<traceID>"

# Search tags
curl -s "http://tempo:3200/api/search/tags"

# Search tag values
curl -s "http://tempo:3200/api/search/tag/service.name/values"
```

## Best Practices

1. Start with `resource.service.name` to narrow scope — most efficient filter
2. Use `duration` filters to find performance issues quickly
3. Use structural operators (`>>`) to trace cross-service latency
4. Combine with Loki: find trace IDs in logs, then query Tempo for the full trace
5. Use `traceDuration` for end-to-end SLO monitoring
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/references/traceql.md
git commit -m "feat: add TraceQL reference for observability skill"
```

---

### Task 5: Create references/alerting.md

**Files:**
- Create: `skills/observability/references/alerting.md`

- [ ] **Step 1: Create alerting.md**

Write `skills/observability/references/alerting.md`:

```markdown
# Alerting Reference

Prometheus alert rule best practices, USE/RED templates, and recording rules. Read this when creating or reviewing alert rules.

## Alert Rule Structure

```yaml
groups:
  - name: group-name
    rules:
      - alert: AlertName
        expr: <PromQL expression>
        for: <duration>
        labels:
          severity: critical|warning|info
          team: <team-name>
        annotations:
          summary: "Short description with {{ $labels.instance }}"
          description: "Detailed description with {{ $value }}"
          runbook_url: "https://docs.example.com/runbooks/AlertName"
```

## Severity Levels

| Severity | Meaning | `for` Duration | Action |
|----------|---------|----------------|--------|
| `critical` | Service down or data loss risk | 1-5m | Page on-call immediately |
| `warning` | Degraded but functional | 5-15m | Investigate during business hours |
| `info` | Notable but not actionable | 15-30m | Dashboard visibility only |

## USE Method Templates (Infrastructure)

### Utilization

```yaml
- alert: HighCPUUtilization
  expr: |
    100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "High CPU on {{ $labels.instance }}"
    description: "CPU utilization is {{ $value | printf \"%.1f\" }}% for 10m"

- alert: HighMemoryUtilization
  expr: |
    100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100) > 85
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "High memory on {{ $labels.instance }}"
    description: "Memory utilization is {{ $value | printf \"%.1f\" }}%"
```

### Saturation

```yaml
- alert: DiskSpaceRunningLow
  expr: |
    predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 24*3600) < 0
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Disk will be full within 24h on {{ $labels.instance }}"
    description: "Current available: {{ $value | humanize1024 }}B"

- alert: HighPodMemorySaturation
  expr: |
    sum by (namespace, pod) (container_memory_usage_bytes)
    / sum by (namespace, pod) (kube_pod_container_resource_limits{resource="memory"}) > 0.9
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.pod }} near memory limit"
```

### Errors

```yaml
- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Node {{ $labels.node }} not ready"

- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} crash looping"
    description: "Restart rate: {{ $value | printf \"%.2f\" }}/sec"
```

## RED Method Templates (Services)

### Rate

```yaml
- alert: LowRequestRate
  expr: |
    sum(rate(http_requests_total{job="api"}[5m])) < 1
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Unusually low request rate for API"
    description: "Current rate: {{ $value | printf \"%.2f\" }} req/s"
```

### Errors

```yaml
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{job="api",status_code=~"5.."}[5m]))
    / sum(rate(http_requests_total{job="api"}[5m])) > 0.05
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error rate above 5% for API"
    description: "Current error rate: {{ $value | printf \"%.2f\" }}%"
```

### Duration

```yaml
- alert: HighLatencyP95
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket{job="api"}[5m])) by (le)
    ) > 1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "P95 latency above 1s for API"
    description: "Current P95: {{ $value | printf \"%.2f\" }}s"

- alert: HighLatencyP99
  expr: |
    histogram_quantile(0.99,
      sum(rate(http_request_duration_seconds_bucket{job="api"}[5m])) by (le)
    ) > 3
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "P99 latency above 3s for API"
    description: "Current P99: {{ $value | printf \"%.2f\" }}s"
```

## SLO-Based Alerts

```yaml
# Error budget burn rate (multiwindow)
- alert: ErrorBudgetBurnRate
  expr: |
    (
      sum(rate(http_requests_total{status_code=~"5.."}[1h]))
      / sum(rate(http_requests_total[1h]))
    ) > (14.4 * (1 - 0.999))
    and
    (
      sum(rate(http_requests_total{status_code=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m]))
    ) > (14.4 * (1 - 0.999))
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning fast (14.4x)"
    description: "At this rate, 30-day error budget exhausted in 2h"
```

## Recording Rules

Pre-compute expensive queries for faster alerting and dashboards:

```yaml
groups:
  - name: recording-rules
    interval: 30s
    rules:
      - record: job:http_requests_total:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_request_errors:rate5m
        expr: sum by (job) (rate(http_requests_total{status_code=~"5.."}[5m]))

      - record: job:http_request_error_ratio:rate5m
        expr: |
          job:http_request_errors:rate5m / job:http_requests_total:rate5m

      - record: instance:node_cpu_utilization:ratio
        expr: |
          1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

## Validation

Always validate alert rules before committing:

```bash
# Check rule syntax
promtool check rules alerts.yaml

# Test rules against live data
promtool test rules test.yaml

# Lint with amtool
amtool check-config alertmanager.yml
```

## Best Practices

1. Every alert needs a `runbook_url` annotation — alerts without runbooks cause panic
2. Use `for` duration to avoid flapping — never alert on instantaneous spikes
3. Use `predict_linear()` for saturation alerts — proactive is better than reactive
4. Include `{{ $labels }}` and `{{ $value }}` in annotations for context
5. Group related alerts (USE per service, RED per endpoint)
6. Use recording rules for expressions used in both alerts and dashboards
7. Test alerts with `promtool test rules` before deploying
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/references/alerting.md
git commit -m "feat: add alerting reference for observability skill"
```

---

### Task 6: Create references/dashboards.md

**Files:**
- Create: `skills/observability/references/dashboards.md`

- [ ] **Step 1: Create dashboards.md**

Write `skills/observability/references/dashboards.md`. Content drawn from harumi-k8s `references/panels.md`:

```markdown
# Grafana Dashboards Reference

Grafana JSON structure, panel types, grid layout, and thresholds. Read this when creating or modifying dashboards.

## Minimal Dashboard JSON

```json
{
  "annotations": { "list": [] },
  "description": "Dashboard description",
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "liveNow": false,
  "panels": [],
  "refresh": "30s",
  "schemaVersion": 38,
  "style": "dark",
  "tags": ["harumi"],
  "templating": { "list": [] },
  "time": { "from": "now-6h", "to": "now" },
  "timepicker": {},
  "timezone": "browser",
  "title": "Dashboard Title",
  "uid": "harumi-category-name",
  "version": 1,
  "weekStart": ""
}
```

## Panel Types

| Type | Use Case | Example |
|------|----------|---------|
| `stat` | Single value with optional sparkline | Node count, error rate |
| `timeseries` | Line/area graphs over time | CPU usage, request rate |
| `gauge` | Circular percentage/threshold | Memory %, disk usage |
| `piechart` | Distribution | Node types, pod status |
| `table` | Tabular data | Pod list, service status |
| `bargauge` | Comparative values | Resource usage by namespace |

## Stat Panel

```json
{
  "type": "stat",
  "title": "Node Ready Status",
  "datasource": { "type": "prometheus", "uid": "prometheus" },
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "thresholds" },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "red", "value": null },
          { "color": "yellow", "value": 1 },
          { "color": "green", "value": 2 }
        ]
      },
      "unit": "short"
    },
    "overrides": []
  },
  "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
  "options": {
    "colorMode": "value",
    "graphMode": "none",
    "justifyMode": "auto",
    "orientation": "auto",
    "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
    "textMode": "auto"
  },
  "targets": [{
    "datasource": { "type": "prometheus", "uid": "prometheus" },
    "editorMode": "code",
    "expr": "sum(kube_node_status_condition{condition=\"Ready\",status=\"true\"})",
    "legendFormat": "Ready Nodes",
    "range": true,
    "refId": "A"
  }]
}
```

Options: `colorMode` (value/background/none), `graphMode` (none/area), `textMode` (auto/value/name/value_and_name).

## Timeseries Panel

```json
{
  "type": "timeseries",
  "title": "Request Rate",
  "datasource": { "type": "prometheus", "uid": "prometheus" },
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "palette-classic" },
      "custom": {
        "drawStyle": "line",
        "fillOpacity": 20,
        "lineInterpolation": "smooth",
        "lineWidth": 2,
        "pointSize": 5,
        "showPoints": "auto",
        "stacking": { "group": "A", "mode": "none" },
        "thresholdsStyle": { "mode": "off" }
      },
      "unit": "reqps"
    },
    "overrides": []
  },
  "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 },
  "options": {
    "legend": { "calcs": ["sum"], "displayMode": "list", "placement": "bottom", "showLegend": true },
    "tooltip": { "mode": "single", "sort": "none" }
  },
  "targets": [{
    "datasource": { "type": "prometheus", "uid": "prometheus" },
    "editorMode": "code",
    "expr": "sum by (status_code) (rate(http_requests_total[5m]))",
    "legendFormat": "{{ status_code }}",
    "range": true,
    "refId": "A"
  }]
}
```

Options: `drawStyle` (line/bars/points), `lineInterpolation` (linear/smooth/stepBefore/stepAfter), `fillOpacity` (0-100), `stacking.mode` (none/normal/percent).

## Gauge Panel

```json
{
  "type": "gauge",
  "title": "Memory Usage",
  "datasource": { "type": "prometheus", "uid": "prometheus" },
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "thresholds" },
      "max": 100, "min": 0,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 70 },
          { "color": "red", "value": 85 }
        ]
      },
      "unit": "percent"
    },
    "overrides": []
  },
  "gridPos": { "h": 6, "w": 6, "x": 0, "y": 0 },
  "options": {
    "orientation": "auto",
    "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
    "showThresholdLabels": false,
    "showThresholdMarkers": true
  },
  "targets": [{
    "datasource": { "type": "prometheus", "uid": "prometheus" },
    "expr": "100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)",
    "legendFormat": "Memory %",
    "refId": "A"
  }]
}
```

## Table Panel

```json
{
  "type": "table",
  "title": "Pod Status by Namespace",
  "datasource": { "type": "prometheus", "uid": "prometheus" },
  "fieldConfig": {
    "defaults": {
      "custom": { "align": "auto", "cellOptions": { "type": "auto" } }
    },
    "overrides": []
  },
  "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
  "options": { "cellHeight": "sm", "showHeader": true },
  "targets": [{
    "datasource": { "type": "prometheus", "uid": "prometheus" },
    "expr": "count by (namespace, phase) (kube_pod_status_phase)",
    "format": "table",
    "instant": true,
    "refId": "A"
  }],
  "transformations": [{
    "id": "organize",
    "options": {
      "excludeByName": { "Time": true },
      "renameByName": { "namespace": "Namespace", "phase": "Phase", "Value": "Count" }
    }
  }]
}
```

Use `"format": "table"` and `"instant": true` for table queries. Add `transformations` to rename/hide columns.

## Piechart Panel

```json
{
  "type": "piechart",
  "title": "Node Count by Type",
  "datasource": { "type": "prometheus", "uid": "prometheus" },
  "fieldConfig": {
    "defaults": { "color": { "mode": "palette-classic" } },
    "overrides": []
  },
  "gridPos": { "h": 8, "w": 6, "x": 12, "y": 0 },
  "options": {
    "displayLabels": ["name", "value"],
    "legend": { "displayMode": "list", "placement": "bottom", "showLegend": true },
    "pieType": "pie",
    "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false }
  },
  "targets": [{
    "datasource": { "type": "prometheus", "uid": "prometheus" },
    "expr": "count by (label_node_kubernetes_io_capacity_type) (kube_node_info)",
    "legendFormat": "{{ label_node_kubernetes_io_capacity_type }}",
    "range": true,
    "refId": "A"
  }]
}
```

Options: `pieType` (pie/donut), `displayLabels` (name/value/percent).

## Grid Layout

24-column grid system:

| Width | Layout | Per Row |
|-------|--------|---------|
| 6 | Quarter | 4 |
| 8 | Third | 3 |
| 12 | Half | 2 |
| 24 | Full | 1 |

```
Full width (w: 24)
Half (w: 12) | Half (w: 12)
Quarter (w: 6) | Quarter | Quarter | Quarter
```

## Thresholds

### Absolute

```json
"thresholds": {
  "mode": "absolute",
  "steps": [
    { "color": "green", "value": null },
    { "color": "yellow", "value": 70 },
    { "color": "red", "value": 90 }
  ]
}
```

### Percentage

```json
"thresholds": { "mode": "percentage", "steps": [...] }
```

## Template Variables

Add dynamic filtering to dashboards:

```json
"templating": {
  "list": [
    {
      "name": "namespace",
      "type": "query",
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "query": "label_values(kube_pod_info, namespace)",
      "refresh": 2,
      "includeAll": true,
      "multi": true,
      "current": { "text": "All", "value": "$__all" }
    },
    {
      "name": "service",
      "type": "query",
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "query": "label_values(kube_pod_info{namespace=~\"$namespace\"}, pod)",
      "refresh": 2,
      "includeAll": true,
      "multi": true
    }
  ]
}
```

Use `$namespace` in queries: `rate(http_requests_total{namespace=~"$namespace"}[5m])`

## Color Modes and Units

**Color modes**: `thresholds`, `palette-classic`, `fixed`, `continuous-GrYlRd`

**Common units**: `short` (auto-scaled counts), `percent` (%), `bytes` (B/KB/MB), `s` (seconds), `ms` (milliseconds), `reqps` (req/s), `ops` (ops/s)

**Reduce calculations**: `lastNotNull`, `last`, `mean`, `max`, `min`, `sum`, `count`
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/references/dashboards.md
git commit -m "feat: add Grafana dashboards reference for observability skill"
```

---

### Task 7: Create references/deployment.md

**Files:**
- Create: `skills/observability/references/deployment.md`

- [ ] **Step 1: Create deployment.md**

Write `skills/observability/references/deployment.md`. Content drawn from harumi-k8s `references/deployment.md`:

```markdown
# Dashboard Deployment Reference

Deploy Grafana dashboards to Kubernetes via ConfigMaps. Read this when deploying or troubleshooting dashboard delivery.

## Deployment Flow

1. Create dashboard JSON in `grafana-dashboards/[env]/[category]/[name].json`
2. Create ConfigMap alongside: `[name].configmap.yaml`
3. Apply to Kubernetes: `kubectl apply -f ... -n monitoring`
4. Grafana sidecar auto-discovers ConfigMaps with label `grafana_dashboard: "1"`
5. Dashboard appears in Grafana UI

## Directory Structure

```
grafana-dashboards/
├── prod/
│   ├── eks/              # EKS cluster dashboards
│   └── applications/     # Application dashboards
└── dev/
    ├── eks/
    └── applications/
```

## ConfigMap Template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: [dashboard-name]-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
    app.kubernetes.io/name: grafana
    app.kubernetes.io/component: dashboard
data:
  [dashboard-name].json: |
    {
      ... dashboard JSON content ...
    }
```

### Naming

| Type | Pattern | Example |
|------|---------|---------|
| Dashboard JSON | `[name].json` | `spot-monitoring.json` |
| ConfigMap file | `[name].configmap.yaml` | `spot-monitoring.configmap.yaml` |
| ConfigMap name | `[name]-dashboard` | `spot-monitoring-dashboard` |

## Deploy Commands

```bash
# Apply single dashboard
kubectl apply -f grafana-dashboards/prod/eks/[name].configmap.yaml -n monitoring

# Apply all dashboards in a category
kubectl apply -f grafana-dashboards/prod/eks/ -n monitoring

# Verify ConfigMap exists
kubectl get configmap -n monitoring | grep dashboard
```

## Verify in Grafana

```bash
# Port-forward to Grafana
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
# Open http://localhost:3000

# Get admin password
kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```

Username: `admin`. Navigate to Dashboards, search by title or browse by tags.

## Update and Delete

```bash
# Update: edit ConfigMap, then re-apply
kubectl apply -f grafana-dashboards/[env]/[category]/[name].configmap.yaml -n monitoring

# Force Grafana reload
kubectl rollout restart deployment prometheus-grafana -n monitoring

# Delete
kubectl delete configmap [dashboard-name]-dashboard -n monitoring
```

## Troubleshooting

| Issue | Check |
|-------|-------|
| Dashboard not appearing | `kubectl get configmap -n monitoring \| grep dashboard` |
| Missing labels | `kubectl get configmap [name] -n monitoring -o yaml \| grep -A5 labels` |
| Sidecar errors | `kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard` |
| Invalid JSON | Validate with `jq` before applying |
| "No Data" panels | Verify datasource UID is `prometheus`; check metrics exist |
| Force reload | `kubectl rollout restart deployment prometheus-grafana -n monitoring` |
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/references/deployment.md
git commit -m "feat: add dashboard deployment reference for observability skill"
```

---

### Task 8: Create references/investigation.md

**Files:**
- Create: `skills/observability/references/investigation.md`

- [ ] **Step 1: Create investigation.md**

Write `skills/observability/references/investigation.md`:

```markdown
# Investigation Reference

USE/RED methodology, correlation patterns, and triage flowcharts for active incident investigation. Read this when entering investigate mode.

## Triage Classification

Classify the problem first — this determines which signals to gather:

| Category | Symptoms | First Signal |
|----------|----------|-------------|
| Latency | Slow responses, timeouts | P95/P99 duration metrics |
| Errors | 5xx responses, exceptions | Error rate metrics + error logs |
| Saturation | OOM kills, throttling, disk full | Resource utilization metrics |
| Crash | Pod restarts, process exits | K8s events + container logs |
| Connectivity | Connection refused, DNS failures | Network metrics + service logs |

## USE Method (Infrastructure)

For every infrastructure resource (CPU, memory, disk, network):

| Signal | What to Check | PromQL |
|--------|--------------|--------|
| **U**tilization | How busy is the resource? | `rate(node_cpu_seconds_total{mode!="idle"}[5m])` |
| **S**aturation | Is work queuing? | `node_load15 / count(node_cpu_seconds_total{mode="idle"})` |
| **E**rrors | Are there errors? | `rate(node_disk_io_time_weighted_seconds_total[5m])` |

### USE Checklist

```
CPU:
  U: rate(node_cpu_seconds_total{mode!="idle"}[5m])
  S: node_load15 > num_cpus
  E: dmesg | grep -i "mce\|error"

Memory:
  U: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
  S: node_memory_MemAvailable_bytes < threshold, OOM kills in dmesg
  E: dmesg | grep -i oom

Disk:
  U: 1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
  S: rate(node_disk_io_time_weighted_seconds_total[5m])
  E: rate(node_disk_io_now[5m]) errors

Network:
  U: rate(node_network_receive_bytes_total[5m]) vs bandwidth
  S: rate(node_network_receive_drop_total[5m])
  E: rate(node_network_receive_errs_total[5m])
```

## RED Method (Services)

For every service endpoint:

| Signal | What to Check | PromQL |
|--------|--------------|--------|
| **R**ate | Request throughput | `sum(rate(http_requests_total[5m]))` |
| **E**rrors | Error percentage | `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| **D**uration | Latency percentiles | `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` |

### RED Checklist

```
For each service:
  R: Is request rate normal? Compare to same time yesterday (offset 24h)
  E: Is error rate elevated? What status codes?
  D: Are P50/P95/P99 latencies elevated? Which endpoints?
```

## Investigation Workflow

### Step 1: Triage (30 seconds)

Identify the category. Run quick checks:

```bash
# K8s cluster health
kubectl get nodes
kubectl get pods --all-namespaces --field-selector=status.phase!=Running

# Recent events
kubectl get events --sort-by='.lastTimestamp' -A | tail -20
```

### Step 2: Gather Metrics (1-2 minutes)

Based on triage category, query Prometheus:

```bash
# Latency — check P95
curl -s "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.95,sum(rate(http_request_duration_seconds_bucket[5m]))by(le))"

# Errors — check error rate
curl -s "http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{status_code=~\"5..\"}[5m]))/sum(rate(http_requests_total[5m]))"

# Saturation — check resource usage
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -20
```

### Step 3: Gather Logs (1-2 minutes)

Query Loki for correlated log entries:

```bash
# Recent errors
logcli query '{namespace="production"} |= "error"' --limit=50 --since=30m

# Specific service errors
logcli query '{namespace="production", container="api"} | json | level="error"' --limit=50 --since=30m

# Tail live logs
logcli query '{namespace="production", container="api"} |= "error"' --tail
```

### Step 4: Gather Traces (if needed)

Query Tempo for slow or errored traces:

```bash
# Search for error traces
curl -s "http://tempo:3200/api/search?q={resource.service.name=\"api\" && status=error}&limit=10"

# Get specific trace
curl -s "http://tempo:3200/api/traces/<traceID>"
```

### Step 5: Correlate

Build a timeline:

```
[timestamp] Metric anomaly detected: CPU spike on node-xyz
[timestamp] Log correlation: OOM kill in namespace production
[timestamp] Trace correlation: Slow DB queries from service api
[timestamp] K8s event: Pod api-xyz evicted
```

Pattern: **Metrics tell you something is wrong. Logs tell you what went wrong. Traces tell you where it went wrong.**

### Step 6: Diagnose and Recommend

Present findings:

```
## Root Cause
[One sentence summary]

## Evidence
- Metric: [what you found]
- Logs: [what you found]
- Traces: [what you found]
- K8s events: [what you found]

## Timeline
[Chronological sequence of events]

## Recommended Actions
1. [Immediate mitigation]
2. [Root cause fix]
3. [Prevention]
```

## Correlation Patterns

### Latency Spike

```
Metrics: P95 duration jumped → check by endpoint
Logs: "timeout" or "slow query" entries → identify bottleneck
Traces: Long spans → which service/operation is slow?
K8s: kubectl top pods → resource contention?
```

### Error Spike

```
Metrics: Error rate increased → which status codes? which endpoints?
Logs: Error/exception messages → stack traces, error details
Traces: Error spans → which service initiated the error?
K8s: Pod restarts, OOMKilled events → resource exhaustion?
```

### Pod Crash Loop

```
K8s: kubectl describe pod → exit code, reason
K8s: kubectl get events → scheduling/resource issues
Logs: kubectl logs --previous → last logs before crash
Metrics: Container restart count, memory usage pre-crash
```

## kubectl Quick Reference

```bash
# Pod status
kubectl get pods -n <ns> -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --tail=100
kubectl logs <pod> -n <ns> --previous          # Previous container logs

# Resource usage
kubectl top nodes
kubectl top pods -n <ns> --sort-by=memory
kubectl top pods -n <ns> --sort-by=cpu

# Events
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get events -n <ns> --field-selector=type=Warning

# Service connectivity
kubectl get endpoints <service> -n <ns>
kubectl run tmp --rm -it --image=busybox -- wget -qO- http://<service>.<ns>:port/health
```
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/references/investigation.md
git commit -m "feat: add investigation reference for observability skill"
```

---

### Task 9: Create evals/evals.json

**Files:**
- Create: `skills/observability/evals/evals.json`

- [ ] **Step 1: Create evals.json**

Write `skills/observability/evals/evals.json`:

```json
{
  "skill_name": "observability",
  "evals": [
    {
      "id": 1,
      "prompt": "Write a PromQL query to get the P95 latency for the API service over the last 5 minutes.",
      "expected_output": "Returns a PromQL query using histogram_quantile with http_request_duration_seconds_bucket.",
      "files": [],
      "expectations": [
        "Uses histogram_quantile(0.95, ...)",
        "Uses rate() with [5m] range",
        "Aggregates by (le)",
        "Targets http_request_duration_seconds_bucket metric",
        "Query is syntactically valid PromQL"
      ]
    },
    {
      "id": 2,
      "prompt": "Create a Prometheus alert rule that fires when error rate exceeds 5% for 5 minutes.",
      "expected_output": "Returns a valid Prometheus alerting rule YAML with proper labels and annotations.",
      "files": [],
      "expectations": [
        "Valid YAML with groups/rules structure",
        "Uses rate() for both error and total requests",
        "Threshold is 0.05 (5%)",
        "Has for: 5m duration",
        "Includes severity label",
        "Includes summary and description annotations with template variables",
        "Offers to validate with promtool check rules",
        "Offers to write to file and commit"
      ]
    },
    {
      "id": 3,
      "prompt": "Create a Grafana dashboard for monitoring the API service with RED metrics.",
      "expected_output": "Returns a complete Grafana dashboard JSON with panels for rate, errors, and duration.",
      "files": [],
      "expectations": [
        "Valid dashboard JSON with schemaVersion 38",
        "Dashboard UID follows harumi-[category]-[name] pattern",
        "Includes stat or timeseries panels for request rate",
        "Includes panel for error rate",
        "Includes panel for latency percentiles (P50, P95, P99)",
        "All panels use datasource type prometheus uid prometheus",
        "Follows 24-column grid layout",
        "Generates accompanying ConfigMap YAML",
        "Offers to write files and commit"
      ]
    },
    {
      "id": 4,
      "prompt": "The API is returning 500 errors intermittently. Investigate.",
      "expected_output": "Enters investigate mode, runs read-only CLI commands, correlates metrics/logs/traces, presents diagnosis.",
      "files": [],
      "expectations": [
        "Enters investigate mode (not author mode)",
        "Checks .devops.yaml for observability stack configuration",
        "Runs kubectl commands to check pod status and events",
        "Queries error rate metrics via Prometheus API or promtool",
        "Queries error logs via logcli or Loki API",
        "All commands are read-only",
        "States what each command does before running it",
        "Presents root cause analysis with evidence",
        "Recommends actions with destructive ones requiring confirmation"
      ]
    },
    {
      "id": 5,
      "prompt": "Write a LogQL query to find all error logs from the production namespace in the last hour, grouped by service.",
      "expected_output": "Returns a LogQL metric query using count_over_time with json parser and sum by.",
      "files": [],
      "expectations": [
        "Uses {namespace=\"production\"} stream selector",
        "Uses json or logfmt parser",
        "Filters on level=\"error\"",
        "Uses count_over_time or rate with [1h] range",
        "Aggregates with sum by (service or container)",
        "Query is syntactically valid LogQL"
      ]
    }
  ]
}
```

- [ ] **Step 2: Commit**

```bash
git add skills/observability/evals/evals.json
git commit -m "feat: add evals for observability skill"
```

---

### Task 10: Update bootstrap skill

**Files:**
- Modify: `skills/using-devops/SKILL.md`

- [ ] **Step 1: Add observability to Available Skills table**

In `skills/using-devops/SKILL.md`, add a row to the Available Skills table after the `setup-devops-config` row:

```markdown
| `harumi-devops-plugin:observability` | Monitoring, alerting, dashboards, PromQL/LogQL, incident investigation | Query authoring, alert rules, Grafana dashboards, active incident debugging |
```

- [ ] **Step 2: Remove observability from Future Skills list**

Remove the line `- `observability` — Monitoring, alerting, dashboards` from the **Future skills** list.

- [ ] **Step 3: Add trigger rules for observability**

Add after the `rotate-access-keys` trigger rule block:

```markdown
Invoke `harumi-devops-plugin:observability` when:
- User asks about metrics, logs, traces, alerts, or dashboards
- User mentions PromQL, LogQL, Grafana, Prometheus, Loki, Tempo
- User says "investigate", "debug", "what's wrong with", "why is X slow/down"
- User references monitoring, alerting, SLOs, SLIs, error rates
```

- [ ] **Step 4: Commit**

```bash
git add skills/using-devops/SKILL.md
git commit -m "feat: register observability skill in bootstrap"
```
