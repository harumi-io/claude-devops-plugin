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
    (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100 > 85
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
  expr: increase(kube_pod_container_status_restarts_total[1h]) > 3
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} crash looping"
    description: "{{ $value | printf \"%.0f\" }} restarts in the last hour"
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
    description: "Current error rate: {{ $value | humanizePercentage }}"
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
      sum(rate(http_requests_total{job="api",status_code=~"5.."}[1h]))
      / sum(rate(http_requests_total{job="api"}[1h]))
    ) > (14.4 * (1 - 0.999))
    and
    (
      sum(rate(http_requests_total{job="api",status_code=~"5.."}[5m]))
      / sum(rate(http_requests_total{job="api"}[5m]))
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

# Lint Alertmanager config
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
