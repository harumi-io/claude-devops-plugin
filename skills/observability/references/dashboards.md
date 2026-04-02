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
  "id": 1,
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
  "id": 2,
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
  "id": 3,
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
  "id": 4,
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
    "expr": "count by (namespace, phase) (kube_pod_status_phase == 1)",
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
  "id": 5,
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
      "query": { "query": "label_values(kube_pod_info, namespace)", "refId": "StandardVariableQuery" },
      "refresh": 2,
      "includeAll": true,
      "multi": true,
      "current": { "text": "All", "value": "$__all" }
    },
    {
      "name": "service",
      "type": "query",
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "query": { "query": "label_values(kube_pod_info{namespace=~\"$namespace\"}, pod)", "refId": "StandardVariableQuery" },
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
