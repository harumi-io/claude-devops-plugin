---
name: deploy-app
description: "Onboard or update an application deployment for ArgoCD. Supports dev (deploy-dev/ on dev branch) and prod (deploy/ on main branch) environments. Generates manifests and a GitHub Actions CI write-back workflow. Use when: user wants to deploy, onboard, or add an app to ArgoCD."
---

# Deploy App

Onboard a new application or update an existing one for ArgoCD management. Uses the CI write-back pattern: manifests live in the app repo; the CI pipeline writes the image tag back on every push.

## Inputs

Ask for these if not provided:

1. **App name** (e.g., `frontend`, `harumi-api`)
2. **Environment** — `dev` or `prod`
3. **App port** (e.g., `3000`, `8000`)
4. **Health check path** (e.g., `/api/health`, `/health`)
5. **ECR repository name** (defaults to `harumi-dev-<app-name>` for dev, `harumi-<app-name>` for prod)
6. **App repo** — the GitHub repository name (e.g., `harumi-io/frontend`)
7. **Target cluster** — from `harumi.yaml` `kubernetes.clusters[]` list

## Environment Reference

| Environment | Reference file | Manifest dir | Branch | CI workflow file |
|-------------|---------------|-------------|--------|-----------------|
| dev | `references/ci-writeback-pattern.md` | `deploy-dev/` | `dev` | `cd-eks-dev.yaml` |
| prod | `references/ci-writeback-prod.md` | `deploy/` | `main` | `cd-eks.yaml` |

## Execution Steps

Follow these steps exactly. Do not skip or reorder.

### Step 1: Read config

Read `harumi.yaml` for:
- `kubernetes.gitops_repo` — gitops repo name (for reference only; manifests go in the app repo)
- `kubernetes.clusters[]` — target cluster context, domain, registry, region
- `naming` — resource naming pattern

Resolve all placeholders from the environment's reference file using these values plus user input. If App repo was provided as `org/repo` format (e.g., `harumi-io/frontend`), split it: `<org>` = `harumi-io`, `<app-repo>` = `frontend`.

### Step 2: Inspect cluster state

```bash
# Existing ArgoCD apps
argocd app list

# Check if namespace exists
kubectl get namespace <namespace> --context <context>

# Check for existing ECR repo
aws ecr describe-repositories --repository-names <ecr-repo> --region <aws-region> 2>/dev/null
```

If the app already exists in ArgoCD, report the conflict and ask the user how to proceed (update manifests vs. skip).

### Step 3: Generate manifests

Using the templates in the environment's reference file, create these files in the app repo:

**For dev:**
- `deploy-dev/namespace.yaml`, `deploy-dev/configmap.yaml`, `deploy-dev/externalsecret.yaml` (if secrets needed), `deploy-dev/deployment.yaml`, `deploy-dev/service.yaml`, `deploy-dev/ingress.yaml`
- `argocd-app.yaml` (repo root — NOT inside `deploy-dev/`)

**For prod:**
- `deploy/namespace.yaml`, `deploy/configmap.yaml`, `deploy/externalsecret.yaml` (if secrets needed), `deploy/deployment.yaml`, `deploy/service.yaml`, `deploy/ingress.yaml`
- `argocd-app.yaml` (repo root — NOT inside `deploy/`)

### Step 4: Generate CI workflow

Create the CI workflow file in the app repo using the workflow template from the environment's reference file:
- Dev: `.github/workflows/cd-eks-dev.yaml`
- Prod: `.github/workflows/cd-eks.yaml`

Use `GITHUB_TOKEN` for the write-back commit — no additional secrets needed.

### Step 5: Add branch protection bypass

**Always** run the bypass script from the environment's reference file directly. Do not hand this off to the user. Substitute `<org>` and `<app-repo>` from the resolved inputs and execute the script using `bash`.

- Dev: bypass on `dev` branch
- Prod: bypass on `main` branch

### Step 6: Hand off

Use the handoff template from the environment's reference file to provide the exact commands needed to deploy. Always include:

1. Push manifests to app repo
2. Register ArgoCD app (`kubectl apply -f argocd-app.yaml`)

Remind the user to verify after apply:
```bash
argocd app get <app-name>
kubectl get pods -n <namespace> --context <context>
curl -k https://<domain><health-path>
```
