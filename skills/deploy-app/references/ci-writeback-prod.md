# CI Write-back Deployment Pattern — Production

In-repo GitOps pattern: manifests live in `deploy/` inside the app repo. The CI pipeline writes the new image tag back to `deploy/deployment.yaml` and commits it to the tracked branch. ArgoCD monitors the configured `targetRevision` and syncs automatically.

Two rollout shapes are supported — choose one based on the deployment context:

- **`shared-prod`** — standard stable deployment; `targetRevision: main`; ArgoCD app registered as `argocd-app.yaml`.
- **`isolated-prod-test`** — initial onboarding validation; `targetRevision: <feature-branch>`; ArgoCD app registered as `argocd-app-prod.yaml` in a dedicated test namespace. Use this to validate the prod CI/CD pipeline before committing to a stable app definition. Not designed to coexist alongside a live shared-prod deployment of the same app.

## Architecture

### Standard (`shared-prod`)

```
<app-repo>/
├── argocd-app.yaml          # ArgoCD Application — lives at repo root, NOT inside deploy/
├── deploy/
│   ├── namespace.yaml
│   ├── deployment.yaml      # CI updates the image tag here on every push
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── externalsecret.yaml  # Only if secrets are needed
└── .github/workflows/
    └── cd-eks.yaml
```

`argocd-app.yaml` lives at the repo root so ArgoCD doesn't self-manage its own Application resource. `targetRevision: main` is the stable default.

### Isolated Rollout Variant (`isolated-prod-test`)

Use this shape to validate the prod CI/CD pipeline during initial onboarding — before committing to a stable app definition. A separate ArgoCD Application (`<app-name>-prod-test`) tracks a temporary feature branch and deploys to a dedicated test namespace (`<app-name>-prod-test`). The stable `argocd-app.yaml` (if it already exists) is not touched. Once validation passes, delete the test Application and namespace, then register the stable `argocd-app.yaml` pointing at `main`.

> **Scope:** This is an initial onboarding validation tool. It is not designed to run in parallel alongside an existing live `shared-prod` deployment of the same app — two ArgoCD Applications targeting `path: deploy` with the same manifests would conflict over shared Kubernetes resources. The isolated namespace ensures the test deployment is self-contained.

```
<app-repo>/
├── argocd-app.yaml           # Stable prod app — unchanged, or not yet created
├── argocd-app-prod.yaml      # Isolated rollout app — tracks <feature-branch>, deploys to <app-name>-prod-test namespace
├── deploy/
│   ├── namespace.yaml
│   ├── deployment.yaml       # CI updates the image tag here via cd-eks-prod.yaml on <feature-branch>
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── externalsecret.yaml   # Only if secrets are needed
└── .github/workflows/
    ├── cd-eks.yaml            # Stable prod CI — triggers on main (not yet active if app is new)
    └── cd-eks-prod.yaml       # Isolated rollout CI — triggers on <feature-branch>
```

`argocd-app-prod.yaml` is a temporary Application resource. It uses `path: deploy` from the feature branch but deploys to a distinct namespace (`<app-name>-prod-test`) so it does not conflict with the stable app's resources. The `.github/workflows/cd-eks-prod.yaml` workflow triggers only on the feature branch and writes back to `deploy/deployment.yaml` on that branch.

## Placeholders

| Placeholder | Example | Source |
|-------------|---------|--------|
| `<app-name>` | `frontend` | User input |
| `<namespace>` | `frontend` | Derived from `<app-name>` |
| `<domain>` | `platform.harumi.io` | `harumi.yaml` `kubernetes.clusters[].domain` |
| `<registry>` | `715841362904.dkr.ecr.us-east-2.amazonaws.com` | `harumi.yaml` `kubernetes.clusters[].registry` |
| `<ecr-repo>` | `harumi-frontend` | `harumi-<app-name>` |
| `<aws-region>` | `us-east-2` | `harumi.yaml` `kubernetes.clusters[].region` |
| `<aws-account-id>` | `715841362904` | `harumi.yaml` or ECR registry URL prefix |
| `<oidc-role-arn>` | `arn:aws:iam::715841362904:role/harumi-us-east-2-github-cicd-role` | `harumi.yaml` or IAM |
| `<secret-path>` | `harumi/frontend/prod` | Naming: `harumi/<app-name>/prod` |
| `<app-port>` | `3000` | User input |
| `<health-path>` | `/api/health` | User input |
| `<certificate-arn>` | `arn:aws:acm:...` | `harumi.yaml` or ACM console |
| `<app-repo>` | `frontend` | User input — GitHub repository name (repo part only) |
| `<org>` | `harumi-io` | User input — GitHub organization name |
| `<context>` | `eks-prod` | `harumi.yaml` `kubernetes.clusters[].context` |
| `<feature-branch>` | `feat/add-prod-pipeline` | User input — feature branch used as `targetRevision` for `isolated-prod-test` |
| `<test-namespace>` | `frontend-prod-test` | Derived: `<app-name>-prod-test`; used only by `isolated-prod-test` |

## Preflight Checks

Run these before generating manifests. Each check must pass before proceeding.

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
# Verify the output includes: "token.actions.githubusercontent.com" as the Federated principal
# and a Condition restricting to the correct repo ("repo:<org>/<app-repo>:*").
# Full end-to-end OIDC assume-role validation (role-to-assume) can only run from a GitHub
# Actions runner — this check confirms the role exists and its trust policy is structurally
# correct. Actual push access is verified on the first CI run.

# 5. ArgoCD repo secret exists in-cluster
kubectl get secret -n argocd --context <context> | rg repo

# 6. Target cluster context exists locally
kubectl config get-contexts -o name | rg "^<context>$"
```

## Manifest Templates

> **Namespace substitution by rollout shape:**
> - `shared-prod`: use `<namespace>` (= `<app-name>`, e.g. `frontend`) everywhere `<namespace>` appears below.
> - `isolated-prod-test`: substitute `<test-namespace>` (= `<app-name>-prod-test`, e.g. `frontend-prod-test`) everywhere `<namespace>` appears below. This ensures all Kubernetes resources land in the dedicated test namespace and do not touch the stable prod namespace.

### namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
  labels:
    app: <app-name>
    environment: prod
```

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <app-name>-config
  namespace: <namespace>
  labels:
    app: <app-name>
data:
  APP_ENV: "prod"
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  PORT: "<app-port>"
  HOSTNAME: "0.0.0.0"
```

### externalsecret.yaml

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: <app-name>-secrets
  namespace: <namespace>
  labels:
    app: <app-name>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: <app-name>-secrets
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: <secret-path>
```

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
  namespace: <namespace>
  labels:
    app: <app-name>
    environment: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: <app-name>
  template:
    metadata:
      labels:
        app: <app-name>
      annotations:
        # Config hash updated by CI pipeline to trigger rolling updates
        harumi.io/config-hash: "00000000"
    spec:
      containers:
        - name: <app-name>
          # Image tag updated by CI pipeline
          image: <registry>/<ecr-repo>:latest
          ports:
            - containerPort: <app-port>
              name: http
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: <health-path>
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: <health-path>
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
          envFrom:
            - configMapRef:
                name: <app-name>-config
            - secretRef:
                name: <app-name>-secrets
                optional: true
```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <app-name>
  namespace: <namespace>
  labels:
    app: <app-name>
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: <app-name>
```

### ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app-name>
  namespace: <namespace>
  annotations:
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: <certificate-arn>
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/healthcheck-path: <health-path>
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
spec:
  ingressClassName: alb
  rules:
    - host: <domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <app-name>
                port:
                  number: 80
```

### argocd-app.yaml (repo root, not inside deploy/) — Standard (`shared-prod`)

```yaml
# ArgoCD Application manifest — Production
# Apply once to register the app with ArgoCD:
#   kubectl apply -f argocd-app.yaml --context <context>
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/<app-repo>.git
    targetRevision: main
    path: deploy
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### argocd-app-prod.yaml (repo root) — Isolated Rollout (`isolated-prod-test`)

Use this when validating the prod CI/CD pipeline on a feature branch before registering the stable app definition. Deploys to a dedicated test namespace (`<test-namespace>` = `<app-name>-prod-test`) so it does not conflict with any future stable prod deployment. Delete this Application and the test namespace once validation passes, then register `argocd-app.yaml` pointing at `main`.

```yaml
# ArgoCD Application manifest — Isolated Prod Rollout Test
# Apply to register the isolated rollout app:
#   kubectl apply -f argocd-app-prod.yaml --context <context>
# Delete after validation completes:
#   kubectl delete -f argocd-app-prod.yaml --context <context>
#   kubectl delete namespace <test-namespace> --context <context>
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>-prod-test
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/<app-repo>.git
    # targetRevision points to the feature branch under validation, not main.
    # This keeps the stable argocd-app.yaml (if it exists) unaffected.
    targetRevision: <feature-branch>
    path: deploy
  destination:
    server: https://kubernetes.default.svc
    # Dedicated test namespace — isolates resources from the stable prod deployment.
    namespace: <test-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## GitHub Actions Workflow Templates

### Standard (`shared-prod`): `.github/workflows/cd-eks.yaml`

Create `.github/workflows/cd-eks.yaml` in the app repo:

```yaml
name: CD Pipeline - EKS Prod

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: write  # required for git push write-back

env:
  AWS_REGION: <aws-region>
  AWS_ACCOUNT_ID: '<aws-account-id>'
  ECR_REPOSITORY: <ecr-repo>
  # Add build-time env vars here if needed

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: <oidc-role-arn>
          aws-region: <aws-region>

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker build \
            -t ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }} \
            -t ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:latest \
            .

      - name: Push Docker image to Amazon ECR
        run: |
          docker push ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          docker push ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:latest

      - name: Update deployment manifest with new image tag
        run: |
          CONFIG_HASH=$(echo -n "${{ github.sha }}-$(date +%s)" | sha256sum | cut -c1-8)
          sed -i "s|image: .*${{ env.ECR_REPOSITORY }}:.*|image: ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }}|" deploy/deployment.yaml
          sed -i "s|harumi.io/config-hash: .*|harumi.io/config-hash: \"${CONFIG_HASH}\"|" deploy/deployment.yaml

      - name: Commit and push manifest changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add deploy/deployment.yaml
          git diff --cached --quiet && echo "No changes to commit" && exit 0
          # [skip ci] prevents this write-back commit from re-triggering the workflow
          git commit -m "chore(deploy): update image tag to ${GITHUB_SHA::8} [skip ci]"
          git pull --rebase origin "${GITHUB_REF_NAME}"
          git push
```

> **Note:** This workflow uses `GITHUB_TOKEN` (the default token, automatically available in all GitHub Actions runs). No additional secrets are needed for the write-back push. The `contents: write` permission grants push access.

### Isolated Rollout (`isolated-prod-test`): `.github/workflows/cd-eks-prod.yaml`

Create `.github/workflows/cd-eks-prod.yaml` in the app repo. This workflow triggers on the feature branch under validation and writes back to `deploy/deployment.yaml` on that branch only. The stable `cd-eks.yaml` workflow (triggers on `main`) is unaffected.

```yaml
name: CD Pipeline - EKS Prod (Isolated Rollout)

on:
  push:
    branches:
      - <feature-branch>   # Replace with the actual feature branch name

permissions:
  id-token: write
  contents: write  # required for git push write-back

env:
  AWS_REGION: <aws-region>
  AWS_ACCOUNT_ID: '<aws-account-id>'
  ECR_REPOSITORY: <ecr-repo>
  # Add build-time env vars here if needed

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: <oidc-role-arn>
          aws-region: <aws-region>

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker build \
            -t ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }} \
            -t ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:latest \
            .

      - name: Push Docker image to Amazon ECR
        run: |
          docker push ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          docker push ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:latest

      - name: Update deployment manifest with new image tag
        run: |
          CONFIG_HASH=$(echo -n "${{ github.sha }}-$(date +%s)" | sha256sum | cut -c1-8)
          sed -i "s|image: .*${{ env.ECR_REPOSITORY }}:.*|image: ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }}|" deploy/deployment.yaml
          sed -i "s|harumi.io/config-hash: .*|harumi.io/config-hash: \"${CONFIG_HASH}\"|" deploy/deployment.yaml

      - name: Commit and push manifest changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add deploy/deployment.yaml
          git diff --cached --quiet && echo "No changes to commit" && exit 0
          # [skip ci] prevents this write-back commit from re-triggering the workflow
          git commit -m "chore(deploy): update image tag to ${GITHUB_SHA::8} [skip ci]"
          git pull --rebase origin "${GITHUB_REF_NAME}"
          git push
```

> **Note:** This workflow uses `GITHUB_TOKEN`. No additional secrets needed. Once the isolated rollout is validated, delete `argocd-app-prod.yaml` from the cluster and this workflow file from the repo.

## Branch Protection — Required Bypass

The branch tracked by the CI write-back workflow MUST have a branch protection bypass for `github-actions[bot]` so the write-back commit can be pushed to the protected branch.

- **`shared-prod`**: bypass on `main`
- **`isolated-prod-test`**: bypass on `<feature-branch>`

Run this script directly (not as a handoff). Replace `<org>`, `<app-repo>`, and `BRANCH` with actual values:

```bash
#!/usr/bin/env bash
set -euo pipefail

ORG="<org>"              # e.g., harumi-io
REPO="<app-repo>"        # e.g., frontend
BRANCH="main"            # shared-prod: "main" | isolated-prod-test: "<feature-branch>"

echo "Configuring github-actions[bot] bypass on ${ORG}/${REPO} (${BRANCH})..."

# Exit early if no branch protection exists
if ! gh api "repos/${ORG}/${REPO}/branches/${BRANCH}/protection" &>/dev/null; then
  echo "No branch protection on ${BRANCH} — no bypass required."
  exit 0
fi

# Add github-actions[bot] to PR review bypass allowances (only if PR reviews are required)
if gh api "repos/${ORG}/${REPO}/branches/${BRANCH}/protection/required_pull_request_reviews" &>/dev/null; then
  # Fetch current bypass apps, users, and teams to preserve existing entries
  CURRENT_APPS=$(gh api "repos/${ORG}/${REPO}/branches/${BRANCH}/protection/required_pull_request_reviews" \
    --jq '[.bypass_pull_request_allowances.apps[].slug] + ["github-actions"] | unique')
  CURRENT_USERS=$(gh api "repos/${ORG}/${REPO}/branches/${BRANCH}/protection/required_pull_request_reviews" \
    --jq '[.bypass_pull_request_allowances.users[].login]')
  CURRENT_TEAMS=$(gh api "repos/${ORG}/${REPO}/branches/${BRANCH}/protection/required_pull_request_reviews" \
    --jq '[.bypass_pull_request_allowances.teams[].slug]')

  gh api --method PATCH \
    "repos/${ORG}/${REPO}/branches/${BRANCH}/protection/required_pull_request_reviews" \
    --input - <<EOF
{
  "bypass_pull_request_allowances": {
    "apps": ${CURRENT_APPS},
    "users": ${CURRENT_USERS},
    "teams": ${CURRENT_TEAMS}
  }
}
EOF
  echo "✓ PR review bypass: github-actions[bot] added"
fi

# Add github-actions[bot] to push allowlist (only if push restrictions are enabled)
if gh api "repos/${ORG}/${REPO}/branches/${BRANCH}/protection/restrictions" &>/dev/null; then
  CURRENT_PUSH_APPS=$(gh api "repos/${ORG}/${REPO}/branches/${BRANCH}/protection/restrictions" \
    --jq '[.apps[].slug] + ["github-actions"] | unique')

  gh api --method POST \
    "repos/${ORG}/${REPO}/branches/${BRANCH}/protection/restrictions/apps" \
    --input - <<EOF
${CURRENT_PUSH_APPS}
EOF
  echo "✓ Push restrictions: github-actions[bot] added"
fi

echo "Done."
```

> **This step is mandatory.** Without it, the `git push` in the CI workflow will fail with a 403 if the tracked branch is protected.

## Handoff Templates

### Standard (`shared-prod`)

```
Application '<app-name>' production manifests ready!

1. Push manifests to app repo:
   git add deploy/ argocd-app.yaml .github/workflows/cd-eks.yaml
   git commit -m "feat: add EKS prod deployment"
   git push origin main

2. Register app with ArgoCD:
   kubectl apply -f argocd-app.yaml --context <context>

Verification:
   argocd app get <app-name>
   kubectl get pods -n <namespace> --context <context>
   curl -k https://<domain><health-path>
```

### Isolated Rollout (`isolated-prod-test`)

```
Application '<app-name>-prod-test' isolated rollout manifests ready!

1. Push manifests to app repo (feature branch):
   git add deploy/ argocd-app-prod.yaml .github/workflows/cd-eks-prod.yaml
   git commit -m "feat: add EKS prod isolated rollout"
   git push origin <feature-branch>

2. Register isolated rollout app with ArgoCD:
   kubectl apply -f argocd-app-prod.yaml --context <context>

Verification (test namespace):
   argocd app get <app-name>-prod-test
   kubectl get pods -n <test-namespace> --context <context>
   curl -k https://<domain><health-path>

After validation passes — clean up and register stable app:
   kubectl delete -f argocd-app-prod.yaml --context <context>
   kubectl delete namespace <test-namespace> --context <context>
   # Update argocd-app.yaml targetRevision to main, then:
   kubectl apply -f argocd-app.yaml --context <context>
```
