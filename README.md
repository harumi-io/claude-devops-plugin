# Harumi DevOps Plugin

DevOps skills for [Claude Code](https://claude.ai/code) and [Cursor](https://cursor.com). Provides infrastructure, Kubernetes, ArgoCD, observability, and cloud operations guidance for harumi's AWS stack.

## Installation

### Claude Code

Register the marketplace and install the plugin:

```bash
/plugin marketplace add git@github.com:harumi-io/harumi-devops-plugin.git
/plugin install harumi-devops-plugin@harumi-devops-marketplace
```

For local development, register directly from a cloned copy:

```bash
/plugin marketplace add /path/to/harumi-devops-plugin
/plugin install harumi-devops-plugin@harumi-devops-marketplace
```

### Cursor

Clone the repository and register the plugin in Cursor Agent chat:

```text
/add-plugin /path/to/harumi-devops-plugin
```

## Configuration

The plugin reads a repo config file from the root of the repository it is installed into.

**Preferred:** `harumi.yaml`
**Legacy fallback:** `.devops.yaml` (backward-compatible — loaded automatically when `harumi.yaml` is absent)

The session-start hook loads `harumi.yaml` when present; otherwise it falls back to `.devops.yaml`. It validates all Kubernetes contexts declared in the config against the local kubeconfig and validates other required prerequisites. Any missing required prerequisite is emitted as a `⚠ BLOCKING:` warning in the session context — work must stop until the human resolves it.

Create a `harumi.yaml` in your repository root:

```yaml
# Identity
project: harumi
org: harumi-io

# Repositories managed by this plugin
repos:
  infra: harumi-io/infrastructure
  k8s: harumi-io/harumi-k8s

# AWS
aws:
  account_id: "123456789012"
  region: us-east-1
  account_alias: harumi

# Terraform
terraform:
  version: "1.5.7"
  state_backend: s3
  state_bucket: harumi-terraform
  var_file: prod.tfvars
  modules:
    main: /
    core_infra: /core-infrastructure
    iam: /iam

# Kubernetes
clusters:
  - name: eks-prod
    context: eks-prod
    environment: production
    domain: harumi.io
    registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
  - name: eks-dev
    context: eks-dev
    environment: development
    domain: dev.harumi.io
    registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com

# ArgoCD / GitOps
argocd:
  gitops_repo: harumi-io/harumi-k8s
  app_of_apps:
    prod: eks/bootstrap/eks-app.yaml
    dev: eks-dev/bootstrap/eks-dev-app.yaml

# CI/CD
cicd:
  platform: github-actions

# Containers
containers:
  runtime: docker
  registry: ecr

# Observability — endpoints reachable via kubectl port-forward or ingress
observability:
  metrics: prometheus
  logs: loki
  traces: tempo
  dashboards: grafana
  endpoints:
    prometheus: http://prometheus.monitoring.svc:9090
    grafana: http://grafana.monitoring.svc:3000
    loki: http://loki.monitoring.svc:3100
    tempo: http://tempo.monitoring.svc:3200
    alertmanager: http://alertmanager.monitoring.svc:9093

# Naming
naming:
  pattern: "{namespace}-{stage}-{name}"
  namespace: harumi
  stage: production

# Docs management
docs:
  generated:
    - docs/architecture/*
    - harumi.yaml
  human_authored:
    - README.md
    - CLAUDE.md
    - AGENTS.md
    - docs/runbooks/*
```

If no config file exists, the plugin surfaces a message prompting you to create `harumi.yaml` or `.devops.yaml`. The `sync-docs` skill can generate `harumi.yaml` from your codebase and cluster state.

## Skills

### Domain Skills

| Skill | Description |
|-------|-------------|
| `infrastructure` | Terraform/IaC management for AWS |
| `kubernetes` | K8s manifest management, Helm, debugging, RBAC, NetworkPolicy, HPA |
| `argocd` | ArgoCD application management, app-of-apps patterns, sync waves, GitOps |
| `observability` | PromQL/LogQL/TraceQL authoring, Grafana dashboards, Prometheus alerts, incident investigation |
| `deploy-app` | App onboarding for ArgoCD with CI write-back pattern (dev and prod environments) |

### Operations Commands

| Skill | Description |
|-------|-------------|
| `create-iam-user` | Create IAM developer/admin/contributor users via Terraform |
| `remove-iam-user` | Offboard IAM users — removes Terraform config and runs plan |
| `create-service-account` | Create IAM service accounts with optional access keys and Secrets Manager |
| `rotate-access-keys` | Rotate IAM access keys with zero-downtime (create new, then deactivate old) |
| `create-vpn-creds` | Generate VPN client certificate and export `.ovpn` config |
| `revoke-vpn-creds` | Revoke a VPN client certificate |
| `list-vpn-users` | List all VPN certificates and their status |

### Kubernetes Operations

| Skill | Description |
|-------|-------------|
| `create-namespace` | Create K8s namespace with RBAC, quotas, NetworkPolicies, and optional ArgoCD registration |
| `debug-pod` | Guided troubleshooting for failing pods with diagnostic decision trees |
| `rollback-deployment` | Roll back a K8s deployment to a previous revision with safety checks |
| `scale-deployment` | Scale deployment replicas with HPA conflict and node capacity checks |

### Meta Skills

| Skill | Description |
|-------|-------------|
| `using-devops` | Bootstrap skill injected at session start — announces available skills and trigger rules |
| `sync-docs` | Keep repo docs in sync with actual infrastructure and cluster state |

## Agents

Agents are thin wrappers that run a skill in a fresh, isolated context. They enable cross-skill parallelism — multiple agents can run concurrently without competing for the same context window.

| Agent | Skill | Key Inputs |
|-------|-------|------------|
| `run-infrastructure` | `infrastructure` | `task`, `module` |
| `run-kubernetes` | `kubernetes` | `task`, `context`, `namespace` |
| `run-argocd` | `argocd` | `task`, `app` |
| `run-observability` | `observability` | `task`, `mode` (author/investigate) |
| `run-debug-pod` | `debug-pod` | `task`, `context`, `namespace`, `pod` |
| `run-deploy-app` | `deploy-app` | `task`, `environment` (dev/prod), `app-name` |

## How It Works

1. **Session start** — The hook loads the bootstrap skill (`using-devops`), reads `harumi.yaml` (or `.devops.yaml` as fallback), validates configured Kubernetes contexts against the local kubeconfig, and checks for drift
2. **Drift detection** — Compares `.harumi-last-sync` with current HEAD. If new commits landed, triggers `sync-docs` to update documentation before other work
3. **Skill triggering** — The bootstrap skill tells the AI when to invoke domain-specific skills based on task context
4. **Safety rules** — Destructive operations (`apply`, `destroy`, `delete`) always require user confirmation via handoff
5. **Parallel agents** — For multi-domain tasks, agents dispatch work to isolated contexts that run concurrently

## Managed Repositories

The plugin is aware of two repositories:

| Repo | Purpose |
|------|---------|
| `harumi-io/infrastructure` | Terraform IaC (AWS) — VPC, ECS, EKS, RDS, IAM, DNS |
| `harumi-io/harumi-k8s` | Kubernetes manifests, ArgoCD apps, Helm values, Grafana dashboards |

Skills use locally configured kubectl contexts for **read-only** cluster access. All write operations require user confirmation via handoff.

## Evals

Each skill includes evaluation scenarios in `evals/evals.json`. Run benchmarks and aggregate results:

```bash
python scripts/aggregate_benchmark.py
```

Review individual eval runs visually with `eval-viewer/viewer.html`.

## License

MIT
