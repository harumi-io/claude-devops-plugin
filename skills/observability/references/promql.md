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
