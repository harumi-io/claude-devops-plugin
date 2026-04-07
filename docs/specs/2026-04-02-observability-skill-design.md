# Observability Skill Design

**Date:** 2026-04-02
**Status:** Draft
**Plugin:** harumi-devops-plugin

## Overview

A new `observability` skill for the harumi-devops-plugin that provides two capabilities: **query/artifact authoring** and **active incident investigation**. The skill is config-adaptive — it reads `.devops.yaml` to determine the observability stack and tailors guidance accordingly.

## Scope

**In scope:**
- PromQL/LogQL/TraceQL query authoring
- Prometheus alert rule authoring and validation
- Grafana dashboard JSON generation with ConfigMap deployment
- Active incident investigation using CLI tools
- Artifact export, validation, and commit to repo

**Out of scope:**
- Stack setup/deployment (Terraform for Prometheus, Grafana, etc.)
- Port-forwarding to in-cluster services
- Destructive operations without explicit user confirmation

## Skill Identity

- **Name:** `observability`
- **Persona:** Senior SRE / Observability Engineer
- **Config-aware:** Reads `observability` section from `.devops.yaml`

### Trigger Rules

Invoke `harumi-devops-plugin:observability` when:
- User asks about metrics, logs, traces, alerts, or dashboards
- User mentions PromQL, LogQL, Grafana, Prometheus, Loki, Tempo
- User says "investigate", "debug", "what's wrong with", "why is X slow/down"
- User references monitoring, alerting, SLOs, SLIs, error rates

### Mode Detection

Two modes, detected from user intent:

| Mode | Keywords | Purpose |
|------|----------|---------|
| **Author** | "write", "create", "build", "query for", "alert when", "dashboard" | Write queries, alert rules, dashboards |
| **Investigate** | "investigate", "debug", "check", "what's happening", "incident" | Active incident investigation |

If ambiguous, ask the user.

## Author Mode

### Workflow

1. **Read context** — Check `.devops.yaml` for stack, scan `docs/references/` for service/architecture context
2. **Understand intent** — What metric/log/trace? What threshold? What service?
3. **Write artifact** — Query, alert rule YAML, or dashboard JSON
4. **Validate** — Run `promtool check rules` for alert rules, JSON lint for dashboards if available
5. **Export** — Write to file in the appropriate repo location, offer to commit

### Stack Depth

| Stack | Depth | Capabilities |
|-------|-------|-------------|
| Prometheus | Deep | PromQL authoring, recording rules, `promtool` validation |
| Grafana | Deep | Dashboard JSON generation, panel types, variables, USE/RED templates |
| Loki | Deep | LogQL authoring, log-based alerts, pattern/metric queries |
| Tempo | Deep | TraceQL authoring, trace-to-logs/metrics correlation |
| Datadog | Light | DQL syntax guidance, monitor definitions |
| CloudWatch | Light | Insights query syntax, metric math, alarm definitions |
| X-Ray | Light | Filter expressions, trace analysis basics |

### Alert Rule Authoring

- Suggest thresholds based on USE method (utilization, saturation, errors) or RED method (rate, errors, duration)
- Include `for` duration, labels, annotations with runbook links
- Write to repo path, validate with `promtool check rules`

### Grafana Dashboard Authoring

Conventions drawn from the harumi-k8s grafana-dashboards skill:

- Search Grafana template library first (grafana.com/grafana/dashboards)
- Follow directory convention: `grafana-dashboards/{env}/{category}/`
- Dashboard UID pattern: `harumi-[category]-[name]`
- Datasource: always `type: "prometheus"`, `uid: "prometheus"`
- 24-column grid layout (quarters w:6, thirds w:8, halves w:12, full w:24)
- Generate both `[name].json` and `[name].configmap.yaml`
- ConfigMap labels: `grafana_dashboard: "1"`, standard `app.kubernetes.io` labels
- Schema version 38, 30s refresh, dark style
- Deploy via `kubectl apply -f`, verify via port-forward to :3000

## Investigate Mode

### Methodology

USE/RED framework as the investigation backbone:
- **USE** (infrastructure): Utilization, Saturation, Errors — for nodes, disks, network
- **RED** (services): Rate, Errors, Duration — for application endpoints

### Workflow

1. **Triage** — Ask what's wrong (or detect from user's description). Classify: latency, errors, saturation, crash, connectivity
2. **Read context** — Check `.devops.yaml` for stack, scan `docs/references/` for service topology and existing runbooks
3. **Gather signals** — Run read-only commands to collect data across the three pillars
4. **Correlate** — Connect metrics anomalies to log patterns to trace spans. Present a timeline.
5. **Diagnose** — Propose root cause with supporting evidence from the gathered data
6. **Recommend** — Suggest fix or mitigation. Destructive actions require explicit user confirmation.

### CLI Tools by Pillar

| Pillar | Deep stack (Prometheus/Loki/Tempo) | Light stack (Datadog/CloudWatch) |
|--------|-----------------------------------|----------------------------------|
| Metrics | `promtool query instant/range` against Prometheus API, `curl` Prometheus HTTP API | `aws cloudwatch get-metric-data`, `aws cloudwatch get-metric-statistics` |
| Logs | `logcli query` against Loki, `logcli series` | `aws logs filter-log-events`, `aws logs start-query` |
| Traces | `curl` Tempo HTTP API (`/api/traces/{traceID}`, `/api/search`) | `aws xray get-trace-summaries`, `aws xray batch-get-traces` |
| K8s | `kubectl logs`, `kubectl top`, `kubectl get events`, `kubectl describe` | Same |

### CLI Tool Priority

1. Dedicated CLI if available (`promtool`, `logcli`, `amtool`)
2. Cloud provider CLI (`aws`, `gcloud`, `az`)
3. `kubectl` for K8s-level investigation
4. `curl` against HTTP APIs as fallback

### Safety Rules

- All investigation commands are **read-only** — no restarts, no scaling, no deletes without user saying "yes"
- Always state what command you're about to run and why before running it
- If a command requires port-forwarding or credentials the user hasn't set up, ask rather than assume

## Artifact Export & Commit

### Export Paths

Auto-detected from repo structure, with fallback conventions:

| Artifact | Discovery | Fallback path |
|----------|-----------|---------------|
| Alert rules | Grep for existing `groups:` YAML files | `monitoring/alerts/` |
| Recording rules | Grep for existing `record:` entries | `monitoring/rules/` |
| Grafana dashboards | Look for existing `grafana-dashboards/` dir | `grafana-dashboards/{env}/{category}/` |
| Dashboard ConfigMaps | Same dir as dashboard JSON | `[dashboard-path]/[name].configmap.yaml` |
| Runbooks | Check `docs/references/` or `docs/runbooks/` | `docs/runbooks/` |

### Export Workflow

1. Write artifact to detected/fallback path
2. Validate if tooling exists (`promtool check rules`, JSON lint)
3. Show diff summary to user
4. Offer to commit — user confirms with "yes"
5. If K8s deployment needed, provide `kubectl apply` handoff (never run apply without confirmation)

Commit message convention follows repo's existing git style.

## Skill File Structure

```
skills/observability/
├── SKILL.md                      # Core skill: modes, workflow, safety rules, config reading
├── references/
│   ├── promql.md                 # PromQL syntax, functions, common patterns
│   ├── logql.md                  # LogQL syntax, parsers, metric queries
│   ├── traceql.md                # TraceQL syntax, span filtering
│   ├── alerting.md               # Alert rule best practices, USE/RED templates
│   ├── dashboards.md             # Grafana JSON structure, panels, grid, thresholds
│   ├── deployment.md             # ConfigMap creation, kubectl deploy, verification
│   ├── investigation.md          # USE/RED methodology, correlation patterns, triage flowchart
│   └── alternatives.md           # Datadog DQL, CloudWatch Insights, X-Ray syntax
└── evals/
    └── evals.json                # Test cases for both author and investigate modes
```

**Built-in references** are plugin-bundled tool syntax/methodology (PromQL cheat sheets, USE/RED templates). **Repo context** (service topology, existing dashboards, runbooks) comes from `docs/references/` in the user's repo.

## Plugin Updates Required

### Bootstrap (`using-devops/SKILL.md`)

- Move `observability` from "future skills" to active skills table
- Add trigger rule block for observability

### Setup Config (`setup-devops-config`)

- Scan repo for observability-related files (dashboards, alert rules, runbooks, monitoring configs)
- Add discovered paths to `docs/references/` index
