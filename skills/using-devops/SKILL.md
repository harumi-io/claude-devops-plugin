---
name: using-devops
description: "Bootstrap skill for harumi-devops-plugin. Injected at session start. Announces available DevOps skills, loads repo config (harumi.yaml or .devops.yaml), defines trigger rules, enforces safety rules, and detects drift."
---

# DevOps Plugin

You have the **harumi-devops-plugin** installed. This plugin provides DevOps skills for harumi's infrastructure and Kubernetes operations.

## Managed Repositories

This plugin manages two repositories:

| Repo | Purpose |
|------|---------|
| `harumi-io/infrastructure` | Terraform IaC (AWS) — VPC, ECS, EKS, RDS, IAM, DNS |
| `harumi-io/harumi-k8s` | Kubernetes manifests, ArgoCD apps, Helm values, Grafana dashboards |

Read the active repo config (injected at session start) for cluster names, contexts, endpoints, and naming conventions. The hook loads `harumi.yaml` when present, otherwise falls back to the legacy `.devops.yaml`. Trust the reported **config source** and any **prerequisite warnings** in the session context over stale docs or examples.

If the session context contains a `## Prerequisites` section, interpret it as follows:
- `⚠ BLOCKING:` — stop all work immediately and ask the user to resolve this before continuing. The only blocking condition is a missing repo config, because without it the plugin cannot give accurate project-specific guidance.
- `⚠ Warning:` — proceed with awareness of the limitation (e.g. kubeconfig is present but unreadable).
- Any other line — informational; no action required.

Live cluster and AWS access is **not assumed**. Missing `kubectl`, an absent kubeconfig, or locally unconfigured contexts are normal states. The assistant can still produce manifests, Terraform, runbooks, and other guidance without live cluster reads.

## Drift Detection

If the drift detection section below reports drift, invoke `harumi-devops-plugin:sync-docs` **before** proceeding with any other work. This ensures documentation is up to date before making further changes.

When `sync-docs` runs, it prefers **live AWS and Kubernetes state** as the source of truth for deployed reality — if locally configured kubectl contexts and AWS CLI access are reachable. When live access is not available, `sync-docs` falls back to repo-only data and explicitly reports that live drift could not be verified. Do not assume live access is present for every session.

## Available Skills

Use the Skill tool to invoke these when triggered:

| Skill | Trigger | Use When |
|-------|---------|----------|
| `harumi-devops-plugin:infrastructure` | `.tf` files, Terraform, AWS infra | Creating, modifying, or reviewing Terraform/IaC configurations |
| `harumi-devops-plugin:sync-docs` | Drift detected, user asks to sync docs | Updating repo documentation to match current state |
| `harumi-devops-plugin:kubernetes` | K8s manifests, Helm, kubectl, pod issues, RBAC | Working with Kubernetes resources, debugging, manifest authoring |
| `harumi-devops-plugin:argocd` | ArgoCD Applications, sync issues, GitOps deployment | Managing ArgoCD apps, app-of-apps, onboarding services |
| `harumi-devops-plugin:observability` | Monitoring, alerting, dashboards, PromQL/LogQL, incident investigation | Query authoring, alert rules, Grafana dashboards, active incident debugging |

## Operations Commands

Quick-action skills for daily DevOps operations. Use the Skill tool to invoke:

| Command | Use When |
|---------|----------|
| `harumi-devops-plugin:create-iam-user` | Add a new developer, admin, or contributor user |
| `harumi-devops-plugin:remove-iam-user` | Remove / offboard an IAM user |
| `harumi-devops-plugin:create-vpn-creds` | Generate VPN certificate and .ovpn config |
| `harumi-devops-plugin:revoke-vpn-creds` | Revoke a VPN certificate |
| `harumi-devops-plugin:list-vpn-users` | List active VPN certificates |
| `harumi-devops-plugin:create-service-account` | Create a new IAM service account |
| `harumi-devops-plugin:rotate-access-keys` | Rotate IAM access keys for a user/service account |
| `harumi-devops-plugin:deploy-app` | Onboard a new app or service to ArgoCD |
| `harumi-devops-plugin:create-namespace` | Create a namespace with RBAC, quotas, network policies |
| `harumi-devops-plugin:rollback-deployment` | Roll back a deployment to a previous revision |
| `harumi-devops-plugin:debug-pod` | Troubleshoot a failing or misbehaving pod |
| `harumi-devops-plugin:scale-deployment` | Scale deployment replicas up or down |

## Trigger Rules

Invoke `harumi-devops-plugin:sync-docs` when:
- Drift was detected at session start (see drift detection section above)
- User asks to sync, update, or refresh documentation
- After user executes a terraform apply or kubectl apply handoff

Invoke `harumi-devops-plugin:infrastructure` when you encounter ANY of:
- `.tf` files or Terraform discussions
- AWS infrastructure tasks
- IaC changes, module creation, state management
- Infrastructure migrations or zero-downtime changes
- Cost or security review of cloud resources

Invoke `harumi-devops-plugin:create-iam-user` when:
- User wants to add, create, or onboard a new AWS user (developer, admin, contributor)

Invoke `harumi-devops-plugin:remove-iam-user` when:
- User wants to remove, delete, or offboard an IAM user

Invoke `harumi-devops-plugin:create-vpn-creds` when:
- User wants to create, generate, or set up VPN credentials or access

Invoke `harumi-devops-plugin:revoke-vpn-creds` when:
- User wants to revoke, remove, or disable VPN access

Invoke `harumi-devops-plugin:list-vpn-users` when:
- User wants to list VPN users, see who has VPN access, or check VPN certificates

Invoke `harumi-devops-plugin:create-service-account` when:
- User wants to create a new service account or programmatic IAM user

Invoke `harumi-devops-plugin:rotate-access-keys` when:
- User wants to rotate, renew, or replace IAM access keys

Invoke `harumi-devops-plugin:kubernetes` when you encounter ANY of:
- K8s manifests (`.yaml` files with `apiVersion` and `kind`)
- Helm charts, Helm values files, or Helm operations
- kubectl operations or discussions
- Pod failures, debugging, or troubleshooting
- RBAC, NetworkPolicy, or pod security configuration
- Resource limits, scaling, or HPA discussions

Invoke `harumi-devops-plugin:argocd` when you encounter ANY of:
- ArgoCD Application manifests or discussions
- Sync/drift issues or ArgoCD troubleshooting
- App-of-apps patterns or GitOps deployment
- Onboarding services to ArgoCD management

Invoke `harumi-devops-plugin:deploy-app` when:
- User wants to deploy, onboard, or add an app to ArgoCD

Invoke `harumi-devops-plugin:create-namespace` when:
- User wants to create a new Kubernetes namespace

Invoke `harumi-devops-plugin:rollback-deployment` when:
- User wants to rollback, revert, or undo a deployment

Invoke `harumi-devops-plugin:debug-pod` when:
- User wants to debug, troubleshoot, or investigate a pod issue

Invoke `harumi-devops-plugin:scale-deployment` when:
- User wants to scale up, scale down, or change replica count of a deployment

Invoke `harumi-devops-plugin:observability` when:
- User asks about metrics, logs, traces, alerts, or dashboards
- User mentions PromQL, LogQL, Grafana, Prometheus, Loki, Tempo
- User says "investigate", "debug", "what's wrong with", "why is X slow/down"
- User references monitoring, alerting, SLOs, SLIs, error rates

## Cluster Read-Access Rules

When locally configured kubectl contexts are available, skills may use them for **read-only** cluster access. Live cluster access is not assumed — skills must work usefully without it.

**Allowed commands:**
- `kubectl get`, `kubectl describe`, `kubectl logs`, `kubectl top`
- `argocd app get`, `argocd app list`
- `helm list`, `helm get values`
- `curl` against observability endpoints (from the active repo config's `observability.endpoints`)

**Forbidden commands (require handoff to user):**
- `kubectl apply`, `kubectl delete`, `kubectl edit`, `kubectl patch`, `kubectl create`
- `helm install`, `helm upgrade`, `helm uninstall`
- `argocd app sync`, `argocd app delete`

## Universal Safety Rules (NON-NEGOTIABLE)

These apply to ALL DevOps skills:

1. **Never run `terraform apply` or `terraform destroy`** — Always provide a handoff with the exact command for the user to execute
2. **Never `kubectl delete` or any write operation without explicit user confirmation** — this applies to ALL environments (production, staging, development). No exceptions.
3. **Never push images to production registries without confirmation**
4. **When live access is available, verify current state before making changes** — If `aws` CLI or `kubectl` access is present, use it to confirm resource existence and configuration; otherwise state assumptions explicitly and ask the user to verify
5. **Always present the handoff pattern for destructive actions:**

```
Configuration ready for apply!

Execute: cd [path] && terraform apply -var-file=[tfvars]
Changes: [summary]
Verification: [CLI commands to confirm]
```

## Parallel Agent Dispatch

When a user request spans multiple independent domains, dispatch agents in parallel for faster results and cleaner context isolation. Each agent runs the corresponding skill in a fresh context window.

### Agent Inventory

| Agent | Skill | Typical Use |
|-------|-------|-------------|
| `run-kubernetes` | `harumi-devops-plugin:kubernetes` | K8s investigation, manifest work |
| `run-infrastructure` | `harumi-devops-plugin:infrastructure` | Terraform state, IaC changes |
| `run-observability` | `harumi-devops-plugin:observability` | Metrics, logs, incident investigation |
| `run-argocd` | `harumi-devops-plugin:argocd` | Sync status, GitOps operations |
| `run-debug-pod` | `harumi-devops-plugin:debug-pod` | Pod troubleshooting |
| `run-deploy-app` | `harumi-devops-plugin:deploy-app` | App onboarding |

### Compatibility Matrix

| Agents | Parallel? | Notes |
|--------|-----------|-------|
| `run-infrastructure` + `run-kubernetes` | Yes | Independent targets |
| `run-kubernetes` + `run-observability` | Yes | Common in incident investigation |
| `run-infrastructure` + `run-observability` | Yes | No shared write targets |
| `run-argocd` + `run-kubernetes` | Yes | Investigation compatible |
| `run-deploy-app` + `run-debug-pod` (same app) | No | Sequence: debug first |
| Any two agents writing to the same resource | No | Always sequential |

### Dispatch Rules

When the user request maps to 2+ compatible skills:

1. **Identify** the relevant agents from the inventory
2. **Check compatibility** using the matrix — if agents are compatible, dispatch in parallel; if not, sequence them
3. **Dispatch** compatible agents simultaneously using the Agent tool, passing the specific task and context as inputs to each
4. **Collect** all results before executing any write operations
5. **Sequence writes** — execute writes one at a time, respecting declared dependencies

### Error Handling

**Parallel agent failure:**
1. Preserve and display results from successful agents
2. Name the failed agent and its error explicitly
3. Ask the user whether to proceed with partial results or abort
4. Never silently proceed into write operations with incomplete information

**Write failure:**
1. Surface the full error
2. Stop the write sequence
3. Present a manual recovery handoff command for the user to execute

**No silent retries.** All error recovery is user-gated.

## Configuration

The active repo config (loaded at session start) tells you:
- **AWS account** — account ID, region, alias
- **Terraform settings** — version, state backend, state bucket, var file, module paths
- **Clusters** — names, contexts, environments, domains, registries
- **ArgoCD** — gitops repo, app-of-apps paths
- **Observability endpoints** — Prometheus, Grafana, Loki, Tempo, Alertmanager URLs
- **Naming pattern** — how resources are named

The hook prefers `harumi.yaml`; if absent it loads the legacy `.devops.yaml`. The session context reports which file was loaded. Any missing repo config is surfaced as `⚠ BLOCKING:` in `## Prerequisites` — work must stop until the human provides one. Kubernetes access status (kubectl availability, context presence) is reported as informational or `⚠ Warning:` only; live cluster access is not assumed.
