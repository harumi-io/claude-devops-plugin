---
name: deploy-app
description: "Onboard a new application or service to ArgoCD. Supports three deployment patterns: application deployment, cluster service, and helm adoption. Use when: user wants to deploy, onboard, or add an app to ArgoCD."
---

# Deploy App

Onboard a new application to ArgoCD management.

## Inputs

Ask for these if not provided:

1. **App name** (e.g., "my-api")
2. **Deployment pattern**:
   - **Application deployment** — new app with its own repo, CI/CD, container image
   - **Cluster service** — off-the-shelf Helm chart (monitoring, ingress, etc.)
   - **Helm adoption** — onboard an existing Helm release to ArgoCD
3. **Target cluster** — from `.devops.yaml` `kubernetes.clusters[]` list

## Execution Steps

Follow these steps exactly. Do not skip or reorder.

### Step 1: Read config

Read `.devops.yaml` for:
- `kubernetes.gitops_repo` — where to create ArgoCD manifests
- `kubernetes.clusters[]` — target cluster context, domain, registry
- `kubernetes.app_of_apps` — root app paths
- `naming` — resource naming pattern

### Step 2: Inspect cluster state

```bash
# Existing ArgoCD apps
argocd app list

# Existing namespaces
kubectl get namespaces --context <context>

# Existing Helm releases (for adoption pattern)
helm list --all-namespaces --context <context>
```

If the app already exists in ArgoCD, report the conflict and ask the user how to proceed.

### Step 3: Follow the deployment pattern

Invoke the `argocd` skill and follow the appropriate pattern from `references/app-patterns.md`:

- **Application deployment**: Generate app repo structure + ArgoCD Application in gitops repo
- **Cluster service**: Generate ArgoCD Application(s) + Helm values in gitops repo
- **Helm adoption**: Export existing values + generate ArgoCD Application in gitops repo

### Step 4: Hand off

Provide the exact commands needed to deploy. Format depends on pattern — see `argocd` skill's app-patterns reference for handoff templates.

Remind the user to verify after apply:
```bash
argocd app get <app-name>
kubectl get pods -n <app-namespace> --context <context>
```
