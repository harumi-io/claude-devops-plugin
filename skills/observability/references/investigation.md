# Investigation Reference

USE/RED methodology, correlation patterns, and triage guide for active incident investigation. Read this when entering investigate mode.

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
| **U**tilization | How busy is the resource? | `(1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100` |
| **S**aturation | Is work queuing? | `node_load15 / count without (cpu, mode) (node_cpu_seconds_total{mode="idle"})` |
| **E**rrors | Are there errors? | `dmesg | grep -i "error\|fail"` (disk errors are kernel-level, not Prometheus metrics) |

### USE Checklist

```
CPU:
  U: (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100
  S: node_load15 > num_cpus (check kubectl top nodes)
  E: dmesg | grep -i "mce\|error"

Memory:
  U: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
  S: OOM kills in events: kubectl get events -A | grep OOMKilled
  E: kubectl get events -A | grep -i "oom\|memory"

Disk:
  U: 1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
  S: rate(node_disk_io_time_weighted_seconds_total[5m])
  E: dmesg | grep -i "error\|fail"  # Disk errors are kernel-level, check node logs

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
| **E**rrors | Error percentage | `sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| **D**uration | Latency percentiles | `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` |

### RED Checklist

```
For each service:
  R: Is request rate normal? Compare to same time yesterday:
     rate(http_requests_total[5m]) / rate(http_requests_total[5m] offset 24h)
  E: Is error rate elevated? What status codes?
     sum by (status_code) (rate(http_requests_total[5m]))
  D: Are P50/P95/P99 latencies elevated? Which endpoints?
     histogram_quantile(0.95, sum by (le, path) (rate(http_request_duration_seconds_bucket[5m])))
```

## Investigation Workflow

### Step 1: Triage (30 seconds)

```bash
# K8s cluster health
kubectl get nodes
kubectl get pods --all-namespaces --field-selector=status.phase!=Running

# Recent warning events
kubectl get events --sort-by='.lastTimestamp' -A --field-selector=type=Warning | tail -20
```

### Step 2: Gather Metrics (1-2 minutes)

Based on triage category, query Prometheus:

```bash
# Latency — check P95
curl -s 'http://prometheus:9090/api/v1/query?query=histogram_quantile(0.95,sum(rate(http_request_duration_seconds_bucket[5m]))by(le))'

# Errors — check error rate
curl -s 'http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{status_code=~"5.."}[5m]))/sum(rate(http_requests_total[5m]))'

# Saturation — check resource usage
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -20
```

### Step 3: Gather Logs (1-2 minutes)

Query Loki for correlated log entries:

```bash
# Recent errors (last 30m)
logcli query '{namespace="production"} |= "error"' --limit=50 --since=30m

# Specific service errors
logcli query '{namespace="production", container="api"} | json | level="error"' --limit=50 --since=30m

# Tail live logs
logcli query '{namespace="production", container="api"} |= "error"' --tail
```

### Step 4: Gather Traces (if latency issue)

Query Tempo for slow or errored traces:

```bash
# Search for error traces
curl -g -s 'http://tempo:3200/api/search?q={resource.service.name="api"&&status=error}&limit=10'

# Get specific trace by ID
curl -s "http://tempo:3200/api/traces/<traceID>"
```

### Step 5: Correlate

Build a timeline connecting the signals:

```
[timestamp] Metric anomaly detected: P95 latency spike on /api/v1/orders
[timestamp] Log correlation: "slow query" errors from database client
[timestamp] Trace correlation: DB span taking 2s on affected traces
[timestamp] K8s event: No relevant pod issues found
```

**Pattern:** Metrics tell you *something is wrong*. Logs tell you *what went wrong*. Traces tell you *where it went wrong*.

### Step 6: Diagnose and Recommend

Present findings in this format:

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
3. [Prevention / alert to add]
```

## Correlation Patterns

### Latency Spike

```
Metrics: P95 duration jumped → check by endpoint and service
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
kubectl run tmp --rm -it --image=busybox -- wget -qO- http://<service>.<ns>:<port>/health
```
