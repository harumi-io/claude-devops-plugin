---
name: observability
description: "Query authoring and active incident investigation for observability stacks. Use when: (1) Writing PromQL/LogQL/TraceQL queries, (2) Creating Prometheus alert rules, (3) Building Grafana dashboards, (4) Investigating incidents — debugging latency, errors, crashes, (5) Any monitoring, alerting, SLO, or dashboard task."
---

# Observability

Act as a **Senior SRE / Observability Engineer**. Read the active repo config (injected at session start) for observability endpoints: Prometheus, Grafana, Loki, Tempo, Alertmanager.

## Mode Detection

This skill has two modes. Detect from user intent:

| Mode | Keywords | Purpose |
|------|----------|---------|
| **Author** | "write", "create", "build", "query for", "alert when", "dashboard" | Write queries, alert rules, dashboards |
| **Investigate** | "investigate", "debug", "check", "what's happening", "incident", "why is X slow/down" | Active incident investigation |

If ambiguous, ask the user which mode they need.

## Safety Rules (NON-NEGOTIABLE)

These apply across both modes:

1. **Never run `kubectl apply` without user confirmation** — Always provide the exact command and wait for "yes"
2. **Never restart, scale, or delete resources** in investigate mode without explicit user confirmation
3. **Always state what command you're about to run and why** before running it
4. **If a command requires credentials or access the user hasn't set up**, ask rather than assume

## Author Mode

### Workflow

1. **Read context** — Check the active repo config for endpoints, scan the project's `docs/references/` for service topology and runbooks
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

> **Note:** The UID pattern and datasource UID above are Harumi project conventions, not values read from the active repo config.

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
4. If K8s deployment needed, provide `kubectl apply` handoff

## Investigate Mode

### Methodology

USE/RED framework as the investigation backbone:
- **USE** (infrastructure): Utilization, Saturation, Errors — for nodes, disks, network
- **RED** (services): Rate, Errors, Duration — for application endpoints

See [references/investigation.md](references/investigation.md) for detailed methodology and correlation patterns.

### Workflow

1. **Triage** — Classify the problem: latency, errors, saturation, crash, connectivity
2. **Read context** — Check the active repo config for endpoints, scan the project's `docs/references/` for service topology and runbooks
3. **Gather signals** — Run read-only CLI commands across three pillars (metrics, logs, traces)
4. **Correlate** — Connect metrics anomalies to log patterns to trace spans. Present a timeline.
5. **Diagnose** — Propose root cause with supporting evidence
6. **Recommend** — Suggest fix or mitigation. Destructive actions require explicit user confirmation.

### CLI Tools

Priority order:
1. Dedicated CLI if available (`promtool`, `logcli`, `amtool`)
2. `aws` CLI for AWS-level investigation
3. `kubectl` for K8s-level investigation
4. `curl` against observability endpoints (from the active repo config `observability.endpoints`)

#### Deep Stack (Prometheus/Loki/Tempo)

| Pillar | Commands |
|--------|----------|
| Metrics | `curl 'http://prometheus:9090/api/v1/query?query=...'` (instant), `curl 'http://prometheus:9090/api/v1/query_range?...'` (range) |
| Logs | `logcli query` against Loki, `logcli series` |
| Traces | `curl` Tempo HTTP API (`/api/traces/{traceID}`, `/api/search`) |
| K8s | `kubectl logs`, `kubectl top`, `kubectl get events`, `kubectl describe` |

## Reference Documentation

Consult these based on the task:

- **[references/promql.md](references/promql.md)** — PromQL syntax, functions, common patterns
- **[references/logql.md](references/logql.md)** — LogQL syntax, parsers, metric queries
- **[references/traceql.md](references/traceql.md)** — TraceQL syntax, span filtering
- **[references/alerting.md](references/alerting.md)** — Alert rule best practices, USE/RED templates
- **[references/dashboards.md](references/dashboards.md)** — Grafana JSON structure, panels, grid, thresholds
- **[references/deployment.md](references/deployment.md)** — ConfigMap creation, kubectl deploy, verification
- **[references/investigation.md](references/investigation.md)** — USE/RED methodology, correlation patterns, triage flowchart
