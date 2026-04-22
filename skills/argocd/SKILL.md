---
name: argocd
description: "Manage ArgoCD applications, app-of-apps patterns, sync waves, and GitOps deployments. Use when: (1) Creating or modifying ArgoCD Application manifests, (2) Onboarding apps or services to ArgoCD, (3) Troubleshooting sync/drift issues, (4) Working with app-of-apps patterns, (5) Managing sync waves and deployment ordering."
---

# ArgoCD

Act as a **Principal Platform Engineer** for ArgoCD and GitOps operations. Read the active `harumi.yaml` config (injected at session start) for cluster context, gitops repo, and app-of-apps structure.

## Critical Rules

### 1. Inspect before acting — ALWAYS

Before creating any ArgoCD Application, check what already exists:

```bash
# List all ArgoCD apps
argocd app list

# Check specific app
argocd app get <app-name>

# Check if namespace exists
kubectl get namespace <namespace> --context <context>

# Check for existing Helm releases
helm list -n <namespace> --context <context>
```

If cluster state differs from reference documentation or `harumi.yaml`, **update the references first**.

### 2. Safety rules apply to ALL environments

- **Never** run `argocd app sync`, `argocd app delete`, or `argocd app patch` — always provide a handoff
- **Never** run `kubectl apply` for ArgoCD Application manifests — always provide a handoff
- **Read-only commands are always safe**: `argocd app list`, `argocd app get`, `argocd app diff`, `argocd app history`
- **Always** verify app health and sync status before and after changes

### 3. Ask when ambiguous

When encountering ambiguity about deployment pattern, target cluster, sync wave, or namespace:

```
I found multiple options for [X]:
1. Option A: [describe]
2. Option B: [describe]
Which approach should I follow?
```

### 4. Update documentation after changes

After user confirms successful deployment, update relevant references if the cluster state reveals patterns not yet documented.

## Handoff Pattern (NON-NEGOTIABLE)

**NEVER execute ArgoCD mutations.** Always provide a handoff:

```
ArgoCD Application ready for deployment!

Please review the manifest(s), then execute:
kubectl apply -f [file] --context [context]

What this will do:
- [summary: new app, new component, adopted release, etc.]

Verification:
argocd app get <app-name>
kubectl get pods -n <namespace> --context <context>
```

## Rollout Lifecycle — targetRevision

When rolling out a new version of an app, use `targetRevision` to gate traffic at the GitOps layer:

### Step 1 — Point at a feature branch for validation

Edit the ArgoCD Application manifest and set `targetRevision` to the feature branch. **Only `targetRevision` changes — `repoURL` and `path` remain exactly as they are.**

```yaml
spec:
  source:
    targetRevision: feature/<branch-name>   # temporary; reverts to main after validation
```

Provide this as a handoff — do not apply directly.

### Step 2 — Verify before promoting

After the user applies the change, run these read-only checks in order:

**Required gates** (always run):

```bash
# 1. Sync status
argocd app get <app-name>

# 2. Pod health
kubectl get pods -n <namespace> --context <context>

# 3. External reachability
curl -sI https://<app-domain>/healthz
```

**Conditional gate** (run only if the app triggers Argo Workflows):

```bash
# 4. Workflow write-back
kubectl get workflows -n <namespace> --context <context>
```

All applicable gates must pass before promoting.

### Step 3 — Move targetRevision back to main

Once the rollout is stable, provide a handoff to restore the Application manifest:

```yaml
spec:
  source:
    targetRevision: main
```

**Never leave `targetRevision` pinned to a feature branch after rollout is complete.**

## Rollout Lifecycle — App-Repo-Owned Manifest

When an application repository owns its own ArgoCD Application manifest (e.g. `argocd-app-prod.yaml` committed inside the app repo), treat that file as **rollout control state** even if the main GitOps repo (`harumi-k8s`) remains unchanged.

Rules:
- Changes to `argocd-app-prod.yaml` in the app repo are equivalent to changes in the GitOps repo — apply the same inspect-before-acting and handoff requirements.
- When asked to promote a rollout, check the app-repo manifest first:

  ```bash
  # Confirm which manifest ArgoCD is currently tracking
  argocd app get <app-name> -o yaml | grep -E 'repoURL|targetRevision|path'
  ```

- If the app-repo manifest and the GitOps repo manifest conflict (e.g. different `targetRevision`), flag the conflict to the user before proceeding.
- After a stable rollout, ensure both the app-repo manifest and the GitOps repo reflect `targetRevision: main` (or the agreed default branch).

## Deployment Patterns

Three patterns for deploying applications via ArgoCD:

### 1. Application Deployment

New app with its own repo, CI/CD pipeline, and container image. Use when deploying a custom application.

### 2. Cluster Service

Off-the-shelf Helm chart deployed as cluster infrastructure. Uses two-source pattern (Helm chart + values from gitops repo). Use for monitoring stacks, ingress controllers, cert-manager, etc.

### 3. Helm Adoption

Onboard an existing Helm release already running in the cluster into ArgoCD management. Use when migrating from manual Helm to GitOps.

See [references/app-patterns.md](references/app-patterns.md) for detailed steps for each pattern.

## Config Reference

```yaml
# From harumi.yaml
kubernetes:
  gitops: argocd
  gitops_repo: harumi-k8s           # where manifests live
  app_of_apps:
    prod: eks/bootstrap/eks-app.yaml
    dev: eks-dev/bootstrap/eks-dev-app.yaml
  clusters:
    - name: eks-prod
      context: eks-prod
      environment: production
      domain: harumi.io
      registry: <ecr-url>
```

Always read these values to determine target paths, domains, and registry URLs.

## Reference Documentation

Consult these based on the task:

- **[references/app-patterns.md](references/app-patterns.md)** — Detailed steps for each deployment pattern
- **[references/sync-waves.md](references/sync-waves.md)** — Wave ordering, hooks, health checks, retry policies
- **[references/examples.md](references/examples.md)** — ArgoCD Application YAML examples
