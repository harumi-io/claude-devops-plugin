---
name: deploy-app
description: "Onboard or update an application deployment for ArgoCD on EKS dev or prod. Supports dev (deploy-dev/ on dev branch) and prod environments with two rollout shapes: shared-prod (standard, re-uses the stable ArgoCD app definition) and isolated-prod-test (separate rollout app for validating the prod deployment path without touching the stable app). Generates manifests and a GitHub Actions CI write-back workflow. Use when: user wants to deploy, onboard, or add an app to ArgoCD."
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

**Prod-only rollout shape decisions** (ask these when environment is `prod`):

8. **Deployment shape** — `shared-prod` (standard: re-uses the existing stable ArgoCD app definition, ArgoCD app filename `argocd-app.yaml`, CI workflow `cd-eks.yaml`) or `isolated-prod-test` (separate rollout app that validates the prod deployment path without re-pointing the stable app, ArgoCD app filename `argocd-app-prod.yaml`, CI workflow `cd-eks-prod.yaml`)
9. **ArgoCD app filename** — `argocd-app.yaml` (default for `shared-prod`) or `argocd-app-prod.yaml` (for `isolated-prod-test`). Confirm this matches the deployment shape chosen above.
10. **CI workflow filename** — `cd-eks.yaml` (default for `shared-prod`) or `cd-eks-prod.yaml` (for `isolated-prod-test`). Confirm this matches the deployment shape chosen above.
11. **Initial `targetRevision`** — `main` (standard for `shared-prod`) or a temporary feature branch (for `isolated-prod-test`, to validate without re-pointing the stable app definition). Once validation passes, the `isolated-prod-test` app can be deleted or its `targetRevision` updated to `main`.
12. **Ingress exposure** — `internal` (default, `alb.ingress.kubernetes.io/scheme: internal`) or `internet-facing` (`alb.ingress.kubernetes.io/scheme: internet-facing`)

## Environment Reference

| Environment | Reference file | Manifest dir | Branch | CI workflow file |
|-------------|---------------|-------------|--------|-----------------|
| dev | `references/ci-writeback-pattern.md` | `deploy-dev/` | `dev` | `cd-eks-dev.yaml` |
| prod | `references/ci-writeback-prod.md` | `deploy/` | usually `main` | `cd-eks.yaml` or `cd-eks-prod.yaml` |

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
```

If the app already exists in ArgoCD, report the conflict and ask the user how to proceed (update manifests vs. skip).

### Step 3: Preflight checks

Run all checks before generating any manifests. Report failures clearly and stop until resolved.

```bash
# 1. AWS Secrets Manager secret exists
aws secretsmanager get-secret-value --secret-id <secret-path> --region <aws-region>

# 2. Required JSON keys exist (adjust key names to the app's expected env vars)
aws secretsmanager get-secret-value --secret-id <secret-path> --region <aws-region> \
  --query SecretString --output text | jq 'keys'

# 3. ECR repository exists
aws ecr describe-repositories --repository-names <ecr-repo> --region <aws-region>

# 4. OIDC role exists and trust policy references the correct GitHub OIDC provider
#    Extract the role name from the ARN (last path segment after '/').
OIDC_ROLE_NAME=$(echo "<oidc-role-arn>" | awk -F'/' '{print $NF}')
aws iam get-role --role-name "${OIDC_ROLE_NAME}" \
  --query 'Role.AssumeRolePolicyDocument' --output json
# Verify the output includes "token.actions.githubusercontent.com" as the Federated principal
# and a Condition scoped to the correct repo ("repo:<org>/<app-repo>:*").
# Note: full OIDC assume-role can only be executed from a GitHub Actions runner.
# This check confirms the role exists and its trust policy is structurally correct;
# actual end-to-end push access is verified on the first CI run.

# 5. ArgoCD repo secret exists in-cluster
kubectl get secret -n argocd --context <context> | rg repo

# 6. Target cluster context exists locally
kubectl config get-contexts -o name | rg "^<context>$"
```

If any check fails, resolve it before proceeding — manifests that cannot be synced waste time to generate.

### Step 4: Generate manifests

Using the templates in the environment's reference file, create these files in the app repo:

**For dev:**
- `deploy-dev/namespace.yaml`, `deploy-dev/configmap.yaml`, `deploy-dev/externalsecret.yaml` (if secrets needed), `deploy-dev/deployment.yaml`, `deploy-dev/service.yaml`, `deploy-dev/ingress.yaml`
- `argocd-app.yaml` (repo root — NOT inside `deploy-dev/`)

**For prod (`shared-prod`):**
- `deploy/namespace.yaml`, `deploy/configmap.yaml`, `deploy/externalsecret.yaml` (if secrets needed), `deploy/deployment.yaml`, `deploy/service.yaml`, `deploy/ingress.yaml`
- `argocd-app.yaml` (repo root — NOT inside `deploy/`); `targetRevision: main`

**For prod (`isolated-prod-test`):**
- `deploy/namespace.yaml`, `deploy/configmap.yaml`, `deploy/externalsecret.yaml` (if secrets needed), `deploy/deployment.yaml`, `deploy/service.yaml`, `deploy/ingress.yaml`
- `argocd-app-prod.yaml` (repo root — NOT inside `deploy/`); `targetRevision: <feature-branch>`; `destination.namespace: <app-name>-prod-test`
- All `namespace:` fields in the generated `deploy/` manifests must use `<test-namespace>` (`<app-name>-prod-test`), not `<namespace>`. See the namespace substitution note in `references/ci-writeback-prod.md`.

### Step 5: Generate CI workflow

Create the CI workflow file in the app repo using the workflow template from the environment's reference file:
- Dev: `.github/workflows/cd-eks-dev.yaml`
- Prod (`shared-prod`): `.github/workflows/cd-eks.yaml`
- Prod (`isolated-prod-test`): `.github/workflows/cd-eks-prod.yaml`

Use `GITHUB_TOKEN` for the write-back commit — no additional secrets needed.

### Step 6: Add branch protection bypass

**Always** run the bypass script from the environment's reference file directly. Do not hand this off to the user. Substitute `<org>` and `<app-repo>` from the resolved inputs and execute the script using `bash`.

- Dev: bypass on `dev` branch
- Prod (`shared-prod`): bypass on `main` branch
- Prod (`isolated-prod-test`): bypass on the feature branch used as `targetRevision`

### Step 7: Hand off

Use the handoff template from the environment's reference file to provide the exact commands needed to deploy. Always include:

1. Push manifests to app repo
2. Register ArgoCD app:
   - `shared-prod`: `kubectl apply -f argocd-app.yaml`
   - `isolated-prod-test`: `kubectl apply -f argocd-app-prod.yaml`

Remind the user to verify after apply:

**`shared-prod`:**
```bash
argocd app get <app-name>
kubectl get pods -n <namespace> --context <context>
curl -k https://<domain><health-path>
```

**`isolated-prod-test`:**
```bash
argocd app get <app-name>-prod-test
kubectl get pods -n <test-namespace> --context <context>
curl -k https://<domain><health-path>
```
