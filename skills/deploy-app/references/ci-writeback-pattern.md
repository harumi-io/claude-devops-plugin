# CI Write-back Deployment Pattern

In-repo GitOps pattern: manifests live in `deploy-dev/` inside the app repo. The CI pipeline writes the new image tag back to `deploy-dev/deployment.yaml` and commits it to the branch. ArgoCD monitors the branch and syncs automatically.

## Architecture

```
<app-repo>/
├── argocd-app.yaml          # ArgoCD Application — lives at repo root, NOT inside deploy-dev/
├── deploy-dev/
│   ├── namespace.yaml
│   ├── deployment.yaml      # CI updates the image tag here on every push
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── externalsecret.yaml  # Only if secrets are needed
└── .github/workflows/
    └── cd-eks-dev.yaml
```

`argocd-app.yaml` lives at the repo root so ArgoCD doesn't self-manage its own Application resource. This allows patching `targetRevision` for feature-branch testing without ArgoCD reverting it.

## Placeholders

| Placeholder | Example | Source |
|-------------|---------|--------|
| `<app-name>` | `frontend` | User input |
| `<namespace>` | `frontend` | Derived from `<app-name>` |
| `<domain>` | `platform.dev.harumi.io` | `harumi.yaml` `kubernetes.clusters[].domain` |
| `<registry>` | `715841362904.dkr.ecr.us-east-2.amazonaws.com` | `harumi.yaml` `kubernetes.clusters[].registry` |
| `<ecr-repo>` | `harumi-dev-frontend` | `harumi-dev-<app-name>` |
| `<aws-region>` | `us-east-2` | `harumi.yaml` `kubernetes.clusters[].region` |
| `<aws-account-id>` | `715841362904` | `harumi.yaml` or ECR registry URL prefix |
| `<oidc-role-arn>` | `arn:aws:iam::715841362904:role/harumi-us-east-2-github-cicd-role` | `harumi.yaml` or IAM |
| `<secret-path>` | `harumi/frontend/dev` | Naming: `harumi/<app-name>/dev` |
| `<app-port>` | `3000` | User input |
| `<health-path>` | `/api/health` | User input |
| `<certificate-arn>` | `arn:aws:acm:...` | `harumi.yaml` or ACM console |
| `<app-repo>` | `frontend` | User input — GitHub repository name (e.g., `harumi-io/frontend`, repo part only) |
| `<org>` | `harumi-io` | User input — GitHub organization name |
| `<context>` | `eks-dev` | `harumi.yaml` `kubernetes.clusters[].context` |

## Manifest Templates

### namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
  labels:
    app: <app-name>
    environment: dev
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
  APP_ENV: "dev"
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  PORT: "<app-port>"
  HOSTNAME: "0.0.0.0"
  # NOTE: NEXT_PUBLIC_* vars are NOT listed here because Next.js inlines them
  # at build time. The source of truth for those values is the workflow env block.
  # Adding them here would create a duplicate that can drift.
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
    environment: dev
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

### argocd-app.yaml (repo root, not inside deploy-dev/)

```yaml
# ArgoCD Application manifest
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
    targetRevision: dev
    path: deploy-dev
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

## GitHub Actions Workflow Template

Create `.github/workflows/cd-eks-dev.yaml` in the app repo:

```yaml
name: CD Pipeline - EKS Dev

on:
  push:
    branches:
      - dev

permissions:
  id-token: write
  contents: write  # required for git push write-back

env:
  AWS_REGION: <aws-region>
  AWS_ACCOUNT_ID: '<aws-account-id>'
  ECR_REPOSITORY: <ecr-repo>
  # Add NEXT_PUBLIC_* or other build-time env vars here if needed

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
          sed -i "s|image: .*${{ env.ECR_REPOSITORY }}:.*|image: ${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }}|" deploy-dev/deployment.yaml
          sed -i "s|harumi.io/config-hash: .*|harumi.io/config-hash: \"${CONFIG_HASH}\"|" deploy-dev/deployment.yaml

      - name: Commit and push manifest changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add deploy-dev/deployment.yaml
          git diff --cached --quiet && echo "No changes to commit" && exit 0
          # [skip ci] prevents this write-back commit from re-triggering the workflow
          git commit -m "chore(deploy-dev): update image tag to ${GITHUB_SHA::8} [skip ci]"
          git pull --rebase origin "${GITHUB_REF_NAME}"
          git push
```

> **Note:** This workflow uses `GITHUB_TOKEN` (the default token, automatically available in all GitHub Actions runs as `${{ secrets.GITHUB_TOKEN }}`). No additional secrets are needed for the write-back push. The `contents: write` permission grants push access.

## Branch Protection — Required Bypass

The `dev` branch MUST have a branch protection bypass for `github-actions[bot]` so the CI write-back commit can be pushed to the protected branch.

Run this script directly (not as a handoff). Replace `<org>` and `<app-repo>` with actual values:

```bash
#!/usr/bin/env bash
set -euo pipefail

ORG="<org>"              # e.g., harumi-io
REPO="<app-repo>"        # e.g., frontend
BRANCH="dev"

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

> **This step is mandatory.** Without it, the `git push` in the CI workflow will fail with a 403 if `dev` is a protected branch.

## Handoff Template

```
Application '<app-name>' manifests ready!

1. Push manifests to app repo:
   git add deploy-dev/ argocd-app.yaml .github/workflows/cd-eks-dev.yaml
   git commit -m "feat: add EKS dev deployment"
   git push origin dev

2. Register app with ArgoCD:
   kubectl apply -f argocd-app.yaml --context <context>

Verification:
   argocd app get <app-name>
   kubectl get pods -n <namespace> --context <context>
   curl -k https://<domain><health-path>
```
