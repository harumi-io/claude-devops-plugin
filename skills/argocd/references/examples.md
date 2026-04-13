# ArgoCD Application YAML Examples

Config-driven examples. Replace placeholders with values from `harumi.yaml`.

## Application Deployment (Pattern 1)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: <project>
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/<app-name>.git
    targetRevision: HEAD
    path: deploy
  destination:
    server: https://kubernetes.default.svc
    namespace: <app-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Cluster Service — Parent App (Pattern 2)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <stack-name>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/<gitops-repo>.git
    targetRevision: HEAD
    path: <cluster-dir>/<stack-name>/argocd
    directory:
      recurse: false
      exclude: '<stack-name>-app.yaml'
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Cluster Service — Helm Component (Pattern 2)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <component-name>
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: <stack-name>
  annotations:
    argocd.argoproj.io/sync-wave: "<wave>"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: <chart-repo-url>
      chart: <chart-name>
      targetRevision: <chart-version>
      helm:
        releaseName: <component-name>
        valueFiles:
          - $values/<cluster-dir>/<stack-name>/<component-name>-values.yaml
    - repoURL: https://github.com/<org>/<gitops-repo>.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
      - ServerSideApply=true
```

## Cluster Service — Resources App (Pattern 2, Wave 0)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <stack-name>-resources
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: <stack-name>
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/<gitops-repo>.git
    targetRevision: HEAD
    path: <cluster-dir>/<stack-name>/resources
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Helm Adoption (Pattern 3)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "<wave>"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: <chart-repo-url>
      chart: <chart-name>
      targetRevision: <chart-version>
      helm:
        releaseName: <existing-release-name>   # MUST match existing release
        valueFiles:
          - $values/<cluster-dir>/<app-name>/helm-values.yaml
    - repoURL: https://github.com/<org>/<gitops-repo>.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
```

## AppProject (for namespace isolation)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: <project-name>
  namespace: argocd
spec:
  description: <project-description>
  sourceRepos:
    - https://github.com/<org>/<gitops-repo>.git
    - <chart-repo-url>
  destinations:
    - namespace: <namespace>
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
```
