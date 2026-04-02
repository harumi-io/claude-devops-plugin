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

> **Note:** `>` as a numeric comparison operator applies *within* a span filter (`{ duration > 1s }`). As a structural operator, `>` means "direct child" and applies *between* span filter groups (`{ service.name = "api" } > { db.system = "postgresql" }`).

## Aggregate Functions

```traceql
# Count spans matching condition
{ resource.service.name = "api" } | count() > 5

# Average duration
{ resource.service.name = "api" } | avg(duration) > 500ms

# Max duration
{ resource.service.name = "api" } | max(duration) > 2s

# Min duration
{ resource.service.name = "api" } | min(duration)
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

# Search tag values (Tempo v2)
curl -s "http://tempo:3200/api/v2/search/tag/service.name/values"
```

## Best Practices

1. Start with `resource.service.name` to narrow scope — most efficient filter
2. Use `duration` filters to find performance issues quickly
3. Use structural operators (`>>`) to trace cross-service latency
4. Combine with Loki: find trace IDs in logs, then query Tempo for the full trace
5. Use `traceDuration` for end-to-end SLO monitoring
