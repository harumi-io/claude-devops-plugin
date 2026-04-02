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
