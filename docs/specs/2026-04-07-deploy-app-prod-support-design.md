# Deploy-App Production Support — Design Spec

## Goal

Extend the `deploy-app` skill to support both dev and prod environments. The user selects the target environment; the skill generates the appropriate manifests, CI workflow, and branch protection bypass.

## Architecture

Both environments follow the same CI write-back pattern — manifests live in the app repo, the CI pipeline writes the image tag back, ArgoCD monitors the branch and syncs automatically. The `argocd-app.yaml` lives at the app repo root (applied manually once), not managed by app-of-apps.

### Environment Matrix

| | Dev | Prod |
|-|-----|------|
| Manifest dir | `deploy-dev/` | `deploy/` |
| Branch | `dev` | `main` |
| CI workflow file | `cd-eks-dev.yaml` | `cd-eks.yaml` |
| ECR repo default | `harumi-dev-<app-name>` | `harumi-<app-name>` |
| Secret path | `harumi/<app-name>/dev` | `harumi/<app-name>/prod` |
| Labels | `environment: dev` | `environment: prod` |
| Branch bypass target | `dev` | `main` |
| ArgoCD `targetRevision` | `dev` | `main` |

Everything else is identical: same 7 manifest templates, same `GITHUB_TOKEN` write-back, same bypass script structure, same 2-step handoff.

## File Structure

| Action | File | Purpose |
|--------|------|---------|
| Modify | `skills/deploy-app/SKILL.md` | Add environment input, route steps to correct reference file |
| Modify | `skills/deploy-app/references/ci-writeback-pattern.md` | Remove `feat/dev*` from CI workflow trigger |
| Create | `skills/deploy-app/references/ci-writeback-prod.md` | Complete self-contained prod reference (mirrors dev structure with prod values) |

### Why separate reference files

An AI agent following a self-contained reference for its environment makes no mistakes from filtering. A single merged file with conditional sections is more error-prone. Duplication across the two files is acceptable for markdown template documents.

## SKILL.md Changes

### New input

Add **Environment** (`dev` or `prod`) as input #2 (after App name). This drives all subsequent routing.

### ECR repo default

- Dev: `harumi-dev-<app-name>`
- Prod: `harumi-<app-name>`

### Step routing

Steps 1-6 remain the same structure. The difference is which reference file the agent reads:

- Dev → `references/ci-writeback-pattern.md`
- Prod → `references/ci-writeback-prod.md`

Step 5 (bypass script) targets the branch matching the environment: `dev` for dev, `main` for prod.

## ci-writeback-prod.md

Self-contained reference document mirroring the dev version with these changes:

1. **Architecture diagram**: `deploy/` instead of `deploy-dev/`
2. **Placeholders table**: prod-specific defaults (domain, ECR repo, secret path, etc.)
3. **Manifest templates**: `environment: prod` labels, prod naming
4. **CI workflow template**: triggers on `main` only, workflow file named `cd-eks.yaml`, `deploy/deployment.yaml` paths in sed commands, commit message prefix `chore(deploy):`
5. **Bypass script**: targets `main` branch instead of `dev`
6. **No feature branch testing section** (not applicable for prod)
7. **Handoff template**: same 2-step structure (push to `main`, register ArgoCD app)

## ci-writeback-pattern.md Fix

Remove `feat/dev*` from the CI workflow trigger branches. The dev workflow triggers only on `dev`.

## Out of Scope

- Promotion workflow (dev → prod)
- Integration with `harumi-k8s` app-of-apps
- ECS migration or dual-deploy coordination
