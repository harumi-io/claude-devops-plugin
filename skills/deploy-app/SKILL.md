---
name: deploy-app
description: "Onboard or update an application deployment for ArgoCD on EKS dev. Generates deploy-dev/ manifests and a GitHub Actions CI write-back workflow. Use when: user wants to deploy, onboard, or add an app to the dev cluster."
---

# Deploy App

Onboard a new application or update an existing one for ArgoCD management on the dev cluster. Uses the CI write-back pattern: manifests live in `deploy-dev/` inside the app repo; the CI pipeline writes the image tag back on every push.

## Inputs

Ask for these if not provided:

1. **App name** (e.g., `frontend`, `harumi-api`)
2. **App port** (e.g., `3000`, `8000`)
3. **Health check path** (e.g., `/api/health`, `/health`)
4. **ECR repository name** (defaults to `harumi-dev-<app-name>`)
5. **App repo** — the GitHub repository name (e.g., `harumi-io/frontend`)
6. **Target cluster** — from `.devops.yaml` `kubernetes.clusters[]` list

## Execution Steps

Follow these steps exactly. Do not skip or reorder.

### Step 1: Read config

Read `.devops.yaml` for:
- `kubernetes.gitops_repo` — gitops repo name (for reference only; manifests go in the app repo)
- `kubernetes.clusters[]` — target cluster context, domain, registry, region
- `naming` — resource naming pattern

Resolve all placeholders from `references/ci-writeback-pattern.md` using these values plus user input. If App repo was provided as `org/repo` format (e.g., `harumi-io/frontend`), split it: `<org>` = `harumi-io`, `<app-repo>` = `frontend`.

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

### Step 3: Generate deploy-dev/ manifests

Using the templates in `references/ci-writeback-pattern.md`, create these files in the app repo:

- `deploy-dev/namespace.yaml`
- `deploy-dev/configmap.yaml`
- `deploy-dev/externalsecret.yaml` (only if the app reads secrets)
- `deploy-dev/deployment.yaml`
- `deploy-dev/service.yaml`
- `deploy-dev/ingress.yaml`
- `argocd-app.yaml` (repo root — NOT inside `deploy-dev/`)

### Step 4: Generate CI workflow

Create `.github/workflows/cd-eks-dev.yaml` in the app repo using the workflow template from `references/ci-writeback-pattern.md`. Use `GITHUB_TOKEN` for the write-back commit — no additional secrets needed.

### Step 5: Add branch protection bypass

**Always** run the bypass script from `references/ci-writeback-pattern.md` directly. Do not hand this off to the user. Substitute `<org>` and `<app-repo>` from the resolved inputs and execute the script using `bash`.

### Step 6: Hand off

Use the handoff template from `references/ci-writeback-pattern.md` to provide the exact commands needed to deploy. Always include:

1. Push manifests to app repo
2. Register ArgoCD app (`kubectl apply -f argocd-app.yaml`)

Remind the user to verify after apply:
```bash
argocd app get <app-name>
kubectl get pods -n <namespace> --context <context>
curl -k https://<domain><health-path>
```
