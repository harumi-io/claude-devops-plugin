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
{job="api"} | json | label_format level=severity, path=url
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
