# ArgoCD Deployment Patterns

Three patterns for deploying applications. All patterns are config-driven — read `harumi.yaml` for cluster details, domains, registries, and gitops repo path.

## Placeholders

All templates use these placeholders. Replace with values from `harumi.yaml`:

| Placeholder | Source |
|-------------|--------|
| `<cluster-dir>` | Infer from target cluster (e.g., `eks` for prod, `eks-dev` for dev) |
| `<environment>` | `kubernetes.clusters[].environment` |
| `<domain>` | `kubernetes.clusters[].domain` |
| `<registry>` | `kubernetes.clusters[].registry` |
| `<gitops-repo>` | `kubernetes.gitops_repo` |
| `<namespace>` | Derived from naming pattern |

## Pattern 1: Application Deployment

For new applications with their own GitHub repository and CI-driven image builds.

### Before Starting

```bash
# Check existing ArgoCD apps
argocd app list

# Check if namespace exists
kubectl get namespace <app-namespace> --context <context>

# Check ECR for existing repo
aws ecr describe-repositories --repository-names <app-name> 2>/dev/null
```

### Directory Structure in Gitops Repo

```
<cluster-dir>/argocd/
└── <app-name>-app.yaml              # ArgoCD Application
```

The app itself lives in its own repo with:
```
<app-repo>/
├── .github/workflows/ci.yaml        # Build + push to ECR + update image tag
├── deploy/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── externalsecret.yaml           # If secrets needed
├── Dockerfile
└── src/
```

### Steps

1. Inspect existing apps and namespaces in the cluster
2. Create the app repository structure (Dockerfile, manifests, CI workflow)
3. Create ArgoCD Application manifest in the gitops repo
4. If ECR is needed, provide Terraform handoff for ECR repository creation
5. Provide handoff for applying the ArgoCD Application

### Handoff

```
Application '<app-name>' manifests ready!

1. Push app repo:
   cd <app-repo> && git add . && git commit -m "feat: initial setup" && git push

2. (If ECR needed) Apply Terraform:
   cd core-infrastructure && terraform apply -var-file=<environment>.tfvars

3. Deploy ArgoCD Application:
   kubectl apply -f <cluster-dir>/argocd/<app-name>-app.yaml --context <context>

Verification:
   argocd app get <app-name>
   kubectl get pods -n <app-namespace> --context <context>
```

---

## Pattern 2: Cluster Service

For Helm chart-based infrastructure services using the App-of-Apps pattern.

### Before Starting

```bash
# Check existing Helm releases in target namespace
helm list -n <namespace> --context <context>

# Check if chart is available
helm search repo <chart-name> --version <chart-version>

# Check for CRD conflicts
kubectl get crd --context <context> | grep <related-crds>
```

### Directory Structure in Gitops Repo

For a new stack (group of related services):
```
<cluster-dir>/<stack-name>/
├── argocd/
│   ├── <stack-name>-app.yaml         # Parent app (App-of-Apps)
│   ├── resources-app.yaml            # Supporting resources (wave 0)
│   ├── <component-a>-app.yaml        # Helm component (wave 1+)
│   └── <component-b>-app.yaml        # Helm component (wave 1+)
├── resources/
│   ├── namespace.yaml
│   ├── secrets.yaml
│   └── configmaps/
├── <component-a>-values.yaml
└── <component-b>-values.yaml
```

For adding a component to an existing stack:
```
<cluster-dir>/<stack-name>/
├── argocd/
│   └── <new-component>-app.yaml      # New Helm component
└── <new-component>-values.yaml       # New values file
```

### Key Conventions

- **Two-source pattern**: Helm chart source + gitops repo as `$values` ref
- **Parent app excludes itself**: `directory.exclude` prevents circular references
- **Resources app creates namespace**: Only wave 0 uses `CreateNamespace=true`
- **ServerSideApply**: Use for charts with CRDs or large resources
- **RespectIgnoreDifferences**: Use for resources with auto-populated fields (e.g., webhook caBundle)

### Steps

1. Inspect existing apps, Helm releases, and CRDs in the cluster
2. Determine if this is a new stack or adding to an existing one
3. Create parent app (if new stack) or just the component app
4. Create Helm values file with minimal overrides
5. Create supporting resources (namespace, secrets) if needed
6. Set sync wave ordering based on dependencies
7. Provide handoff

---

## Pattern 3: Helm Adoption

For onboarding an existing Helm release into ArgoCD management.

### Before Starting

```bash
# Get current release info
helm list -n <namespace> --context <context>
helm get values <release-name> -n <namespace> --context <context> -o yaml

# Check for manual modifications
helm get manifest <release-name> -n <namespace> --context <context> > /tmp/helm-manifest.yaml
kubectl diff -f /tmp/helm-manifest.yaml --context <context>
```

### Directory Structure in Gitops Repo

```
<cluster-dir>/
├── argocd/
│   ├── <app-name>-app.yaml              # ArgoCD Application
│   └── <app-name>-resources-app.yaml    # Optional: supporting resources
└── <app-name>/
    ├── helm-values.yaml                  # Extracted from running release
    └── resources/                         # Optional: ingress, configmaps
```

### Steps

1. Inspect the running Helm release (values, manifest, version)
2. Check for manual modifications outside Helm
3. Export current values to a values file
4. Create ArgoCD Application manifest with `helm.releaseName` matching the existing release name
5. Use `ServerSideApply` to handle ownership conflicts
6. Verify sync shows no drift before proceeding
7. Provide handoff

### Critical: releaseName Must Match

The `helm.releaseName` in the ArgoCD Application MUST match the existing Helm release name. If they don't match, ArgoCD will create a NEW release instead of adopting the existing one, causing duplicate resources.
