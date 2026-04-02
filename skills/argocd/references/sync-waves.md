# ArgoCD Sync Waves and Ordering

## Wave Ordering Conventions

| Wave | Purpose | Examples |
|------|---------|---------|
| 0 | Supporting resources (namespace, secrets, configmaps) | `resources-app.yaml` |
| 1 | Independent backends (no inter-dependencies) | `loki-app.yaml`, `tempo-app.yaml`, `redis-app.yaml` |
| 2 | Services depending on wave 1 | `prometheus-stack-app.yaml` (needs loki endpoint) |
| 3+ | Services depending on earlier waves | `otel-collector-app.yaml` (needs prom + loki + tempo) |

Set wave with annotation:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

## Sync Hooks

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync    # Runs before sync
    # Options: PreSync, Sync, PostSync, SyncFail, Skip
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    # Options: HookSucceeded, HookFailed, BeforeHookCreation
```

Common uses:
- **PreSync**: Database migrations, config validation
- **PostSync**: Smoke tests, notification
- **SyncFail**: Alerting, cleanup

## Health Checks

ArgoCD has built-in health checks for common resources. Custom health checks can be defined in ArgoCD ConfigMap:

```yaml
# In argocd-cm ConfigMap
resource.customizations.health.<group_kind>: |
  hs = {}
  if obj.status ~= nil then
    if obj.status.conditions ~= nil then
      for i, condition in ipairs(obj.status.conditions) do
        if condition.type == "Ready" and condition.status == "True" then
          hs.status = "Healthy"
          hs.message = condition.message
          return hs
        end
      end
    end
  end
  hs.status = "Progressing"
  hs.message = "Waiting for Ready condition"
  return hs
```

## Retry Policy

```yaml
spec:
  syncPolicy:
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## ServerSideApply

Use for:
- Helm charts that install CRDs
- Resources with many fields (>256KB)
- Resources managed by multiple controllers

```yaml
spec:
  syncPolicy:
    syncOptions:
      - ServerSideApply=true
```

## RespectIgnoreDifferences

Use for fields auto-populated by webhooks or controllers:

```yaml
spec:
  ignoreDifferences:
    - group: admissionregistration.k8s.io
      kind: MutatingWebhookConfiguration
      jsonPointers:
        - /webhooks/0/clientConfig/caBundle
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
```

## App-of-Apps Pattern

Parent app auto-discovers child apps in its `argocd/` directory:

```yaml
spec:
  source:
    repoURL: <gitops-repo-url>
    path: <cluster-dir>/<stack-name>/argocd
    targetRevision: HEAD
    directory:
      recurse: false
      exclude: '<stack-name>-app.yaml'  # Exclude self to prevent circular reference
```

The `exclude` is critical — without it, the parent app will try to manage itself.
