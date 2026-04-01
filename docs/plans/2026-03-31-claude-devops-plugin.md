# Harumi DevOps Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that provides DevOps-oriented skills for infrastructure management, mirroring the superpowers plugin architecture, with an MVP of the devops skill ported from harumi-io/infrastructure.

**Architecture:** Plugin follows the superpowers pattern — `.claude-plugin/plugin.json` manifest, session-start hook that injects a bootstrap skill, and individual skills under `skills/`. Configuration is driven by a `.devops.yaml` file in the user's repo root, merged with plugin defaults at session start. The devops skill is generalized from the existing Harumi-specific devops skill to support AWS/GCP/Azure via config.

**Tech Stack:** Bash (session-start hook), Markdown (SKILL.md files), JSON (plugin manifests), YAML (config)

---

## File Structure

```
harumi-devops-plugin/
├── .claude-plugin/
│   └── plugin.json                          # Claude Code manifest
├── .cursor-plugin/
│   └── plugin.json                          # Cursor manifest
├── hooks/
│   ├── session-start                        # Bash bootstrap hook
│   ├── hooks.json                           # Claude Code hooks config
│   └── hooks-cursor.json                    # Cursor hooks config
├── skills/
│   └── using-devops/
│       └── SKILL.md                         # Bootstrap meta skill
│   └── infrastructure/
│       ├── SKILL.md                         # Infrastructure skill
│       └── references/
│           ├── architecture.md              # Architecture documentation guide
│           ├── workflow.md                  # 10-phase workflow (provider-aware)
│           ├── examples.md                  # Terraform examples (provider-aware)
│           ├── modules.md                   # Module patterns
│           ├── naming.md                    # Naming conventions (config-driven)
│           └── security.md                  # Security patterns (provider-aware)
├── config/
│   └── default.devops.yaml                  # Default config (Harumi as example)
├── agents/                                  # Empty for MVP
├── package.json
├── README.md
├── LICENSE
└── CHANGELOG.md
```

---

### Task 1: Initialize Repository and Plugin Manifests

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.cursor-plugin/plugin.json`
- Create: `package.json`
- Create: `LICENSE`
- Create: `CHANGELOG.md`

- [ ] **Step 1: Create `.claude-plugin/plugin.json`**

```json
{
  "name": "harumi-devops-plugin",
  "description": "DevOps skills for infrastructure, Kubernetes, CI/CD, and cloud operations",
  "version": "0.1.0",
  "author": {
    "name": "Harumi.io",
    "email": "devops@harumi.io"
  },
  "homepage": "https://github.com/harumi-io/harumi-devops-plugin",
  "repository": "https://github.com/harumi-io/harumi-devops-plugin",
  "license": "MIT",
  "keywords": [
    "devops",
    "infrastructure",
    "terraform",
    "kubernetes",
    "aws",
    "gcp",
    "azure",
    "cicd"
  ]
}
```

- [ ] **Step 2: Create `.cursor-plugin/plugin.json`**

```json
{
  "name": "harumi-devops-plugin",
  "displayName": "Harumi DevOps Plugin",
  "description": "DevOps skills for infrastructure, Kubernetes, CI/CD, and cloud operations",
  "version": "0.1.0",
  "author": {
    "name": "Harumi.io",
    "email": "devops@harumi.io"
  },
  "homepage": "https://github.com/harumi-io/harumi-devops-plugin",
  "repository": "https://github.com/harumi-io/harumi-devops-plugin",
  "license": "MIT",
  "keywords": [
    "devops",
    "infrastructure",
    "terraform",
    "kubernetes",
    "aws",
    "gcp",
    "azure",
    "cicd"
  ],
  "skills": "./skills/",
  "agents": "./agents/",
  "hooks": "./hooks/hooks-cursor.json"
}
```

- [ ] **Step 3: Create `package.json`**

```json
{
  "name": "harumi-devops-plugin",
  "version": "0.1.0",
  "type": "module"
}
```

- [ ] **Step 4: Create `LICENSE`**

MIT license file with `Copyright (c) 2026 Harumi.io`.

- [ ] **Step 5: Create `CHANGELOG.md`**

```markdown
# Changelog

## 0.1.0

- Initial release
- Plugin scaffold with Claude Code and Cursor manifests
- Session-start hook with `.devops.yaml` config loading
- `using-devops` bootstrap skill
- `infrastructure` skill (generalized from Harumi.io devops skill)
- Default `.devops.yaml` config
```

- [ ] **Step 6: Create empty `agents/` directory**

```bash
mkdir -p agents && touch agents/.gitkeep
```

- [ ] **Step 7: Commit**

```bash
git add .claude-plugin/ .cursor-plugin/ package.json LICENSE CHANGELOG.md agents/
git commit -m "feat: initialize plugin scaffold with manifests"
```

---

### Task 2: Default Configuration File

**Files:**
- Create: `config/default.devops.yaml`

- [ ] **Step 1: Create `config/default.devops.yaml`**

```yaml
# Default DevOps plugin configuration
# Override per-repo by placing a .devops.yaml in your repository root

cloud:
  provider: aws                    # aws | gcp | azure
  region: us-east-2
  account_alias: harumi

terraform:
  version: "1.5.7"
  state_backend: s3
  var_file: prod.tfvars

kubernetes:
  tool: kubectl                    # kubectl | oc (openshift)
  gitops: argocd                   # argocd | flux | none
  clusters:
    - name: eks-dev
      context: eks-dev
    - name: eks-prod
      context: eks-prod

cicd:
  platform: github-actions         # github-actions | gitlab-ci | circleci

observability:
  metrics: prometheus              # prometheus | datadog | cloudwatch
  dashboards: grafana
  logs: loki                       # loki | cloudwatch | datadog
  traces: tempo                    # tempo | jaeger | xray

containers:
  runtime: docker                  # docker | podman | nerdctl
  registry: ecr                    # ecr | gcr | dockerhub | ghcr

naming:
  pattern: "{namespace}-{stage}-{name}"
  namespace: harumi
  stage: production
```

- [ ] **Step 2: Commit**

```bash
git add config/
git commit -m "feat: add default .devops.yaml configuration"
```

---

### Task 3: Hooks System (Session-Start Bootstrap)

**Files:**
- Create: `hooks/session-start`
- Create: `hooks/hooks.json`
- Create: `hooks/hooks-cursor.json`

- [ ] **Step 1: Create `hooks/hooks.json` (Claude Code format)**

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/session-start\"",
            "async": false
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 2: Create `hooks/hooks-cursor.json` (Cursor format)**

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "command": "./hooks/session-start"
      }
    ]
  }
}
```

- [ ] **Step 3: Create `hooks/session-start`**

```bash
#!/usr/bin/env bash
# SessionStart hook for harumi-devops-plugin
# Loads the bootstrap skill and merges .devops.yaml config

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PLUGIN_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

# --- Read bootstrap skill ---
using_devops_content=$(cat "${PLUGIN_ROOT}/skills/using-devops/SKILL.md" 2>&1 || echo "Error reading using-devops skill")

# --- Merge .devops.yaml config ---
# Look for .devops.yaml in the current working directory, fall back to plugin defaults
user_config=""
default_config_path="${PLUGIN_ROOT}/config/default.devops.yaml"
repo_config_path="${PWD}/.devops.yaml"

if [ -f "$repo_config_path" ]; then
    user_config=$(cat "$repo_config_path" 2>/dev/null || true)
    config_source="repo (.devops.yaml)"
else
    user_config=$(cat "$default_config_path" 2>/dev/null || true)
    config_source="plugin defaults"
fi

# --- Build context ---
config_block="## Active Configuration (source: ${config_source})\n\n\`\`\`yaml\n${user_config}\n\`\`\`"

# --- Escape for JSON ---
escape_for_json() {
    local s="$1"
    s="${s//\\/\\\\}"
    s="${s//\"/\\\"}"
    s="${s//$'\n'/\\n}"
    s="${s//$'\r'/\\r}"
    s="${s//$'\t'/\\t}"
    printf '%s' "$s"
}

using_devops_escaped=$(escape_for_json "$using_devops_content")
config_escaped=$(escape_for_json "$config_block")
session_context="<IMPORTANT>\nYou have the harumi-devops-plugin installed.\n\n**Below is the bootstrap skill. For all other skills, use the 'Skill' tool:**\n\n${using_devops_escaped}\n\n${config_escaped}\n</IMPORTANT>"

# --- Platform-specific output ---
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
    printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ]; then
    printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$session_context"
else
    printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
fi

exit 0
```

- [ ] **Step 4: Make hook executable**

```bash
chmod +x hooks/session-start
```

- [ ] **Step 5: Verify hook runs without errors**

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops && CLAUDE_PLUGIN_ROOT="$(pwd)" ./hooks/session-start | python3 -m json.tool
```

Expected: Valid JSON output with `hookSpecificOutput.additionalContext` containing the bootstrap skill content. Will show "Error reading using-devops skill" in the content until we create the skill in Task 4.

- [ ] **Step 6: Commit**

```bash
git add hooks/
git commit -m "feat: add session-start hook with config loading and platform detection"
```

---

### Task 4: Bootstrap Skill (`using-devops`)

**Files:**
- Create: `skills/using-devops/SKILL.md`

- [ ] **Step 1: Create `skills/using-devops/SKILL.md`**

```markdown
---
name: using-devops
description: "Bootstrap skill for harumi-devops-plugin. Injected at session start. Announces available DevOps skills, loads repo config, defines trigger rules, and enforces safety rules for infrastructure operations."
---

# DevOps Plugin

You have the **harumi-devops-plugin** installed. This plugin provides DevOps-oriented skills for infrastructure, Kubernetes, CI/CD, and cloud operations.

## Available Skills

Use the Skill tool to invoke these when triggered:

| Skill | Trigger | Use When |
|-------|---------|----------|
| `harumi-devops-plugin:devops` | `.tf` files, Terraform, AWS/GCP/Azure infra | Creating, modifying, or reviewing Terraform/IaC configurations |

**Future skills** (not yet available):
- `kubernetes` — K8s manifests, Helm charts, ArgoCD/Flux
- `cicd` — CI/CD pipeline configs, deployment workflows
- `cost-optimization` — Resource sizing, cost analysis
- `observability` — Monitoring, alerting, dashboards
- `security-operations` — IAM, secrets, compliance
- `containers` — Dockerfiles, image builds, registries

## Trigger Rules

Invoke `harumi-devops-plugin:devops` when you encounter ANY of:
- `.tf` files or Terraform discussions
- AWS, GCP, or Azure infrastructure tasks
- IaC changes, module creation, state management
- Infrastructure migrations or zero-downtime changes
- Cost or security review of cloud resources

## Universal Safety Rules (NON-NEGOTIABLE)

These apply to ALL DevOps skills:

1. **Never run `terraform apply` or `terraform destroy`** — Always provide a handoff with the exact command for the user to execute
2. **Never `kubectl delete` in production without explicit user confirmation**
3. **Never push images to production registries without confirmation**
4. **Always verify current state before making changes** — Use CLI commands (aws, gcloud, az, kubectl) to confirm resource existence and configuration
5. **Always present the handoff pattern for destructive actions:**

```
Configuration ready for apply!

Execute: cd [path] && terraform apply -var-file=[tfvars]
Changes: [summary]
Verification: [CLI commands to confirm]
```

## Configuration

The active `.devops.yaml` config (loaded at session start) tells you:
- **Cloud provider** — which CLI commands to use (aws, gcloud, az)
- **Terraform settings** — version, state backend, var file
- **Naming pattern** — how resources are named
- **Stack details** — K8s tool, CI/CD platform, observability stack, container runtime

Read config values to adapt your guidance to the user's specific stack.

## Relationship to Superpowers

This plugin is **domain-oriented** (how to do DevOps). Superpowers is **process-oriented** (how to work: TDD, debugging, planning). They complement each other. Use superpowers skills for workflow (brainstorming, planning, debugging) and devops skills for domain knowledge (Terraform patterns, cloud architecture, security).
```

- [ ] **Step 2: Re-test hook now that skill exists**

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops && CLAUDE_PLUGIN_ROOT="$(pwd)" ./hooks/session-start | python3 -c "import sys,json; d=json.load(sys.stdin); print('OK' if 'DevOps Plugin' in d['hookSpecificOutput']['additionalContext'] else 'FAIL')"
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add skills/using-devops/
git commit -m "feat: add using-devops bootstrap skill"
```

---

### Task 5: Infrastructure Skill — Main SKILL.md

**Files:**
- Create: `skills/infrastructure/SKILL.md`

- [ ] **Step 1: Create `skills/infrastructure/SKILL.md`**

This is the generalized version of the existing Harumi devops skill. It reads cloud provider, naming, and state config from `.devops.yaml` instead of hardcoding Harumi values.

```markdown
---
name: infrastructure
description: "Write and manage Terraform/IaC infrastructure code following project conventions. Use when: (1) Creating, modifying, or reviewing Terraform configurations, (2) Working with cloud infrastructure (AWS, GCP, Azure), (3) Managing IaC changes across modules, (4) Planning infrastructure migrations or zero-downtime changes, (5) Reviewing security patterns, cost implications, or naming conventions."
---

# Infrastructure

Act as a **Principal Platform Engineer** for the project's cloud infrastructure. Read the active `.devops.yaml` config (injected at session start) for provider, region, naming, and state backend details.

## Critical Rules

### 1. Always verify with CLI before changes

Never rely solely on Terraform state or code. Confirm resource existence, current configuration, security settings, and dependencies.

**AWS:**
```bash
aws ec2 describe-vpcs --vpc-ids vpc-xxxxx
aws ecs describe-services --cluster [cluster] --services [name]
aws rds describe-db-instances --db-instance-identifier [name]
aws iam get-role --role-name [name]
```

**GCP:**
```bash
gcloud compute networks describe [name]
gcloud container clusters describe [name] --region [region]
gcloud sql instances describe [name]
gcloud iam roles describe [name]
```

**Azure:**
```bash
az network vnet show --name [name] --resource-group [rg]
az aks show --name [name] --resource-group [rg]
az sql server show --name [name] --resource-group [rg]
az role definition list --name [name]
```

Use the provider from `.devops.yaml` to determine which CLI commands to suggest.

### 2. Ask when ambiguous

When encountering ambiguity about resource location, naming pattern, or approach, ALWAYS ask the user:

```
I found multiple patterns for [X]:
1. Pattern A: [describe]
2. Pattern B: [describe]
Which approach should I follow?
```

### 3. Present downtime alternatives

When a change may cause downtime, present options: zero-downtime migration, in-place update, or maintenance window recreation. Include complexity, risk level, and expected downtime for each.

See [references/workflow.md](references/workflow.md) for downtime assessment templates and zero-downtime patterns.

### 4. Present cost options with estimates

When choosing instance sizes, storage, or any cost-impacting decision, present at least 3 options (cost-optimized, balanced, performance) with monthly cost estimates.

See [references/workflow.md](references/workflow.md) for pricing reference tables.

### 5. Update documentation after changes

After user confirms successful apply, update relevant docs: module README, architecture docs, and any AI assistant guidance files.

### 6. Always use the correct backend-config for terraform init

When the project uses multiple state files (common with modular Terraform), NEVER run bare `terraform init` for non-root modules. Always check the project's state configuration and use explicit `-backend-config` when required.

```bash
# Pattern for multiple state files:
cd [module-path]
terraform init -backend-config="key=[module]/terraform.tfstate"

# WRONG — may switch to wrong state:
cd [module-path]
terraform init                    # MAY USE WRONG STATE
```

## Apply Safety (NON-NEGOTIABLE)

**NEVER execute `terraform apply` or `terraform destroy`.** Provide a handoff:

```
Configuration ready for apply!

Execute:
cd [MODULE_PATH]
terraform apply -var-file=[var_file from config]

Changes: [Summary]
Verification: [CLI commands to confirm]
```

## Workflow

1. **Consult** — Read project docs and module-specific guidance
2. **Verify** — Check current state with cloud CLI
3. **Clarify** — Ask user about ambiguities (module location, naming, approach)
4. **Assess** — Evaluate downtime risk and cost implications; present alternatives
5. **Implement** — Write Terraform code following project patterns
6. **Validate** — `terraform fmt -check -recursive` and `terraform validate`
7. **Plan** — `terraform plan -var-file=[var_file]`
8. **Handoff** — Provide apply command to user (NEVER apply directly)
9. **Verify** — After user reports back, confirm with cloud CLI
10. **Document** — Update relevant documentation

See [references/workflow.md](references/workflow.md) for detailed phase instructions.

## Quick Reference

### Naming

Read the `naming` section of `.devops.yaml` for the project's naming pattern. Common patterns:

- CloudPosse: `{namespace}-{stage}-{name}` (e.g., harumi-production-data-lake)
- GCP: `{project}-{env}-{name}` (e.g., myproject-prod-vpc)
- Azure: `{prefix}-{env}-{name}` (e.g., app-prod-rg)

See [references/naming.md](references/naming.md) for detailed conventions.

### Cross-module references

```hcl
data "terraform_remote_state" "core" {
  backend = "s3"  # or "gcs" or "azurerm" — match your state_backend
  config = {
    bucket = "[state-bucket]"
    key    = "[module]/terraform.tfstate"
    region = "[region from config]"
  }
}
```

## Reference Documentation

Consult these based on the task:

- **[references/architecture.md](references/architecture.md)** — How to document and understand infrastructure architecture
- **[references/workflow.md](references/workflow.md)** — Detailed workflow phases, downtime templates, cost tables, handoff templates
- **[references/examples.md](references/examples.md)** — Terraform code examples for common resources
- **[references/modules.md](references/modules.md)** — Module patterns, cross-module references, conditional resources
- **[references/naming.md](references/naming.md)** — Resource naming conventions (config-driven)
- **[references/security.md](references/security.md)** — Encryption, secrets, IAM, network security patterns
```

- [ ] **Step 2: Commit**

```bash
git add skills/infrastructure/SKILL.md
git commit -m "feat: add devops skill main SKILL.md"
```

---

### Task 6: Infrastructure Skill — references/workflow.md

**Files:**
- Create: `skills/infrastructure/references/workflow.md`

- [ ] **Step 1: Create `skills/infrastructure/references/workflow.md`**

Generalized from the Harumi workflow.md — removes hardcoded Harumi resource names, makes CLI commands provider-conditional, keeps the universal 10-phase process, downtime tables, and cost tables.

```markdown
# Infrastructure Workflow and Handoff

Detailed workflow phases for infrastructure changes. Read this when following the full change workflow or need downtime/cost guidance.

## Phase 2: Verify with CLI

Verify current state before changes. Terraform state may be outdated or resources may have been modified outside Terraform.

### AWS

```bash
aws ec2 describe-vpcs --vpc-ids [vpc-id]
aws ec2 describe-subnets --filters "Name=vpc-id,Values=[vpc-id]"
aws ecs describe-services --cluster [cluster-name] --services [name]
aws ecs describe-task-definition --task-definition [name]
aws rds describe-db-instances --db-instance-identifier [name]
aws elasticache describe-replication-groups --replication-group-id [name]
aws s3api get-bucket-encryption --bucket [bucket-name]
aws s3api get-public-access-block --bucket [bucket-name]
aws iam get-user --user-name [username]
aws iam get-role --role-name [role-name]
aws eks describe-cluster --name [cluster-name]
aws route53 list-resource-record-sets --hosted-zone-id [zone-id]
```

### GCP

```bash
gcloud compute networks describe [name] --project [project]
gcloud compute networks subnets list --network [name]
gcloud container clusters describe [name] --region [region]
gcloud sql instances describe [name]
gcloud redis instances describe [name] --region [region]
gcloud storage buckets describe gs://[bucket-name]
gcloud iam service-accounts describe [email]
gcloud dns record-sets list --zone [zone-name]
```

### Azure

```bash
az network vnet show --name [name] --resource-group [rg]
az network vnet subnet list --vnet-name [name] --resource-group [rg]
az aks show --name [name] --resource-group [rg]
az sql server show --name [name] --resource-group [rg]
az redis show --name [name] --resource-group [rg]
az storage account show --name [name] --resource-group [rg]
az ad sp show --id [id]
az network dns record-set list --zone-name [zone] --resource-group [rg]
```

## Phase 3: Clarify Ambiguities

Common questions to ask the user:

- **Resource location**: Which module or directory should this resource live in?
- **Pattern**: Follow existing patterns or introduce a new convention?
- **Naming**: What naming pattern does this project use? (Check `.devops.yaml` naming section)

## Phase 4a: Downtime Assessment

### Downtime risk by resource type

| Resource | Downtime Risk | Data Loss Risk |
|----------|---------------|----------------|
| RDS / Cloud SQL / Azure SQL | HIGH (10-30 min) | YES |
| Redis / Memorystore / Azure Cache | HIGH (5-15 min) | YES (cache) |
| ECS / Cloud Run / ACI | LOW (1-5 min) | NO |
| EKS / GKE / AKS Node Group | MEDIUM (5-15 min) | NO |
| ALB / Cloud LB / App Gateway | MEDIUM (5-10 min) | NO |
| VPC / Network | CRITICAL | YES |
| S3 / GCS / Azure Storage | CRITICAL | YES |
| Security Group / Firewall Rule | LOW | NO |
| IAM Role / Service Account | LOW | NO |
| DNS Record | LOW (TTL dependent) | NO |

### Present alternatives template

```
This change will recreate [resource], which may cause downtime.

Option 1: Zero-downtime migration (Recommended for production)
- Create new resource -> migrate data/traffic -> verify -> remove old
- Complexity: HIGH | Risk: LOW | Downtime: ZERO

Option 2: In-place modification (If provider supports it)
- Apply change directly, provider handles update
- Complexity: LOW | Risk: MEDIUM | Downtime: Brief (~5 min)

Option 3: Maintenance window recreation
- Schedule window -> destroy + create -> restore data
- Complexity: LOW | Risk: HIGH | Downtime: 10-30 minutes

Which approach do you prefer?
```

### Zero-downtime patterns

**ECS/Cloud Run**: Rolling updates via deployment configuration (maximum_percent = 200, minimum_healthy_percent = 100).

**EKS/GKE/AKS Node Groups**: Create new group first, cordon/drain old, then remove old using feature flags.

**Load Balancers**: Use weighted routing to shift traffic gradually.

**Databases**: Create read replica, promote, switch traffic, remove old.

## Phase 4b: Cost Assessment

### Present cost options template

```
This resource has cost implications:

Option 1: Cost-optimized (~$X/month)
- Specs: [instance type, storage]
- Best for: Dev/test, low traffic

Option 2: Balanced (~$Y/month) - Recommended
- Specs: [instance type, storage]
- Best for: Production, moderate load

Option 3: Performance (~$Z/month)
- Specs: [instance type, storage]
- Best for: High traffic, critical workloads

Which option fits your needs?
```

### AWS Pricing Reference (us-east-2, approximate)

**RDS PostgreSQL** (add ~20% for Multi-AZ):

| Instance | vCPU | Memory | Monthly |
|----------|------|--------|---------|
| db.t4g.micro | 2 | 1 GB | ~$12 |
| db.t4g.small | 2 | 2 GB | ~$24 |
| db.t4g.medium | 2 | 4 GB | ~$48 |
| db.t4g.large | 2 | 8 GB | ~$96 |
| db.m6g.large | 2 | 8 GB | ~$120 |

**ElastiCache Redis**:

| Instance | Memory | Monthly |
|----------|--------|---------|
| cache.t4g.micro | 0.5 GB | ~$12 |
| cache.t4g.small | 1.4 GB | ~$24 |
| cache.t4g.medium | 3.1 GB | ~$48 |
| cache.m6g.large | 6.4 GB | ~$110 |

**ECS Fargate** (Spot is ~70% cheaper):

| CPU | Memory | Monthly (24/7) |
|-----|--------|----------------|
| 256 | 512 MB | ~$9 |
| 512 | 2 GB | ~$27 |
| 1024 | 4 GB | ~$54 |
| 2048 | 4 GB | ~$72 |
| 4096 | 8 GB | ~$144 |

**EKS** (Spot saves 50-70%):

| Component | Monthly |
|-----------|---------|
| Control Plane | ~$73 |
| t3.large node | ~$60 |
| m5.large node | ~$70 |
| NAT Gateway (per AZ) | ~$32 + data |

### GCP Pricing Reference (approximate)

**Cloud SQL PostgreSQL**:

| Instance | vCPU | Memory | Monthly |
|----------|------|--------|---------|
| db-f1-micro | shared | 0.6 GB | ~$8 |
| db-g1-small | shared | 1.7 GB | ~$26 |
| db-custom-2-4096 | 2 | 4 GB | ~$50 |
| db-custom-2-8192 | 2 | 8 GB | ~$95 |

**GKE**:

| Component | Monthly |
|-----------|---------|
| Autopilot (per vCPU) | ~$25 |
| Standard cluster fee | ~$73 |
| e2-standard-4 node | ~$97 |

### Azure Pricing Reference (approximate)

**Azure Database for PostgreSQL**:

| Instance | vCPU | Memory | Monthly |
|----------|------|--------|---------|
| B1ms | 1 | 2 GB | ~$25 |
| GP_Gen5_2 | 2 | 10 GB | ~$125 |

**AKS**:

| Component | Monthly |
|-----------|---------|
| Control Plane (free tier) | $0 |
| Standard_D2s_v3 node | ~$70 |

## Phase 5-6: Implement and Validate

```bash
# Format check
terraform fmt -check -recursive
terraform fmt  # Fix formatting

# Validation
terraform validate
```

Common validation errors: `Undeclared resource` (check typo), `Missing required argument` (add field), `Invalid reference` (check provider docs), `Type mismatch` (fix variable type).

## Phase 7: Plan

```bash
terraform plan -var-file=[var_file from .devops.yaml]
```

Red flags in plan output:
- **Unexpected destroys**: Why is this being destroyed?
- **Force replacement (-/+)**: Will data be lost?
- **Many changes**: Does scope match intent?
- **Changes to core resources**: VPC, cluster, etc.

## Phase 8: Handoff

```
Configuration ready for apply!

Please review the plan, then execute:
cd [MODULE_PATH]
terraform apply -var-file=[var_file]

What this will do:
- Create X new resources
- Modify Y existing resources
- Destroy Z resources (if any)

Verification after apply:
terraform state list | grep [resource]
[cloud CLI verification command]

Do NOT proceed if you see unexpected deletions.
```

Risk indicators: Creates only = Safe, Modifications = Caution, Deletions/state changes = High Risk.

## Phase 9: Verify After Apply

After user confirms success, verify with cloud CLI (use service-specific commands from Phase 2).

```
Verification complete!
- [resource 1]: Created successfully
- [resource 2]: Configuration matches expected
- [resource 3]: Security settings correct
```

## Phase 10: Update Documentation

| File | Update When |
|------|-------------|
| Module README/CLAUDE.md | AI guidance or usage changed |
| Architecture docs | Structure changed |
| Naming conventions | New patterns established |

## Multi-Module Changes

Order of operations:
1. **Core Infrastructure** first (VPC, DNS, security)
2. **IAM** second (roles and policies)
3. **Databases** third (RDS, Redis, etc.)
4. **Applications** last (ECS, Lambda, Cloud Run, etc.)

When changes span modules, apply core infrastructure first, then refresh dependent modules.

## State Management

Safe operations:
```bash
terraform state list
terraform state show '<address>'
terraform state mv '<old>' '<new>'
terraform import -var-file=[var_file] '<address>' '<id>'
```

Dangerous operations (require explicit user approval):
```bash
terraform state rm '<address>'
terraform force-unlock LOCK_ID
```

## Troubleshooting

| Error | Solution |
|-------|----------|
| `Error acquiring the state lock` | Wait for other process, or `terraform force-unlock LOCK_ID` |
| `Failed to instantiate provider` | `terraform init -upgrade` |
| `Unable to find remote state` | Check backend config key path |
```

- [ ] **Step 2: Commit**

```bash
git add skills/infrastructure/references/workflow.md
git commit -m "feat: add infrastructure workflow reference (provider-aware)"
```

---

### Task 7: Infrastructure Skill — references/architecture.md

**Files:**
- Create: `skills/infrastructure/references/architecture.md`

- [ ] **Step 1: Create `skills/infrastructure/references/architecture.md`**

This is a guide for documenting architecture, not the Harumi-specific architecture (which stays in the Harumi repo).

```markdown
# Infrastructure Architecture Guide

How to document and understand infrastructure architecture. Read this when working on changes that span modules or need to understand dependencies.

## What to Document

Every infrastructure project should have a clear picture of:

### State File Organization

Map which root modules manage which resources, and where their state lives:

| Root Module | State Path | Status |
|-------------|------------|--------|
| Core Infrastructure | `[backend]://[bucket]/[key]` | Active |
| IAM | `[backend]://[bucket]/[key]` | Active |
| Applications | `[backend]://[bucket]/[key]` | Active |

### Module Inventory

List active submodules with their status and purpose:

| Submodule | Path | Status | Description |
|-----------|------|--------|-------------|
| network | `./network` | ACTIVE | VPC, subnets, NAT, IGW |
| dns | `./dns` | ACTIVE | DNS zones, certificates |
| security | `./security` | ACTIVE | KMS keys, WAF rules |

Use `CONDITIONAL` status for feature-flagged modules.

### Network Configuration

Document CIDRs and subnet layout:

| Component | CIDR | Location |
|-----------|------|----------|
| VPC | 10.0.0.0/16 | core/network |
| Public Subnet A | 10.0.100.0/20 | core/network |
| Private Subnet A | 10.0.116.0/20 | core/network |

### Domain Configuration

| Domain | Purpose | Managed By |
|--------|---------|------------|
| example.com | Primary domain | core/dns |
| *.example.com | Wildcard cert | core/dns |
| api.example.com | Backend API | applications |

### Conditional Feature Flags

```hcl
# prod.tfvars
enable_feature_x = true
enable_feature_y = false
remove_legacy_z  = false  # ALWAYS default false for removal flags
```

## Initialization Commands

Always document the exact init commands for each module, especially backend-config requirements:

```bash
# Module with explicit backend-config
cd [module-path]
terraform init -backend-config="key=[module]/terraform.tfstate"
terraform plan -var-file=[var_file]

# Module with default backend
cd [root-path]
terraform init
terraform plan -var-file=[var_file]
```

## Cross-Module Dependencies

Document which modules reference other modules' state:

```
core-infrastructure (VPC, DNS, Security)
    |
    +--> IAM (roles reference VPC)
    |
    +--> Applications (reference VPC, DNS, IAM outputs)
```

## Example: Harumi.io Architecture

The Harumi.io project uses three independent state files:

```
infrastructure/
+-- main.tf                    # Legacy: ECS, RDS, Redis, S3, Glue, CDN
+-- core-infrastructure/       # Modern: Network, DNS, Security, ECS cluster, EKS
+-- iam/                       # Users, groups, roles, service accounts
+-- modules/                   # 27 reusable child modules
```

This demonstrates a common migration pattern from monolithic to modular Terraform.
```

- [ ] **Step 2: Commit**

```bash
git add skills/infrastructure/references/architecture.md
git commit -m "feat: add infrastructure architecture reference guide"
```

---

### Task 8: Infrastructure Skill — references/examples.md

**Files:**
- Create: `skills/infrastructure/references/examples.md`

- [ ] **Step 1: Create `skills/infrastructure/references/examples.md`**

```markdown
# Infrastructure Code Examples

Terraform code examples for common resources. Read this when implementing new resources to match existing patterns.

## S3 Bucket (AWS)

```hcl
module "s3_bucket" {
  source  = "cloudposse/s3-bucket/aws"
  version = "4.5.0"

  namespace = var.naming_namespace   # from .devops.yaml naming.namespace
  stage     = var.environment
  name      = var.name

  acl                = "private"
  versioning_enabled = true

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## GCS Bucket (GCP)

```hcl
resource "google_storage_bucket" "this" {
  name          = "${var.project}-${var.environment}-${var.name}"
  location      = var.region
  storage_class = "STANDARD"

  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"

  versioning {
    enabled = true
  }
}
```

## Azure Storage Account

```hcl
resource "azurerm_storage_account" "this" {
  name                     = "${var.prefix}${var.environment}${var.name}"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
}
```

## ECS Fargate Service (AWS)

```hcl
module "container_definition" {
  source         = "cloudposse/ecs-container-definition/aws"
  version        = "0.60.0"
  container_name = "${var.container_name}-${var.environment}"
  container_image = "${data.aws_caller_identity.current.account_id}.dkr.ecr.${var.region}.amazonaws.com/${var.image}:latest"
  essential      = true
  environment    = var.container_environment

  log_configuration = {
    logDriver = "awslogs"
    options = {
      "awslogs-group"         = "${var.container_name}-container"
      "awslogs-region"        = var.region
      "awslogs-create-group"  = "true"
      "awslogs-stream-prefix" = "logs"
    }
  }

  port_mappings = [for port in var.container_ports : {
    containerPort = port
    hostPort      = port
    protocol      = "tcp"
  }]
}

resource "aws_ecs_task_definition" "this" {
  execution_role_arn       = var.ecs_task_execution_role
  task_role_arn            = var.task_arn
  container_definitions    = module.container_definition.json_map_encoded_list
  cpu                      = var.task_cpu
  family                   = "${var.container_name}-${var.environment}-ecs-task"
  memory                   = var.task_memory
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
}

resource "aws_ecs_service" "this" {
  cluster              = var.ecs_cluster_id
  desired_count        = var.service_count
  launch_type          = var.use_spot_instances ? null : "FARGATE"
  name                 = "${var.container_name}-${var.environment}-ecs-task"
  task_definition      = aws_ecs_task_definition.this.arn
  force_new_deployment = true
  enable_execute_command = true

  lifecycle {
    ignore_changes = [desired_count]
  }

  dynamic "capacity_provider_strategy" {
    for_each = var.use_spot_instances ? [1] : []
    content {
      capacity_provider = "FARGATE_SPOT"
      weight            = 100
    }
  }

  network_configuration {
    security_groups  = var.security_group_ids
    subnets          = var.private_subnets
    assign_public_ip = false
  }
}
```

## Cloud Run Service (GCP)

```hcl
resource "google_cloud_run_v2_service" "this" {
  name     = "${var.project}-${var.name}"
  location = var.region

  template {
    containers {
      image = "${var.region}-docker.pkg.dev/${var.project}/${var.repository}/${var.image}:latest"

      resources {
        limits = {
          cpu    = var.cpu
          memory = var.memory
        }
      }

      dynamic "env" {
        for_each = var.environment_variables
        content {
          name  = env.value.name
          value = env.value.value
        }
      }
    }

    scaling {
      min_instance_count = var.min_instances
      max_instance_count = var.max_instances
    }

    vpc_access {
      connector = var.vpc_connector_id
      egress    = "PRIVATE_RANGES_ONLY"
    }
  }
}
```

## EKS Cluster (AWS)

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "${var.naming_namespace}-${var.environment}-eks"
  cluster_version = var.kubernetes_version

  cluster_endpoint_public_access = true

  vpc_id     = var.vpc_id
  subnet_ids = var.private_subnet_ids

  eks_managed_node_groups = {
    spot = {
      instance_types = ["t3.large", "t3a.large", "m5.large"]
      capacity_type  = "SPOT"
      min_size       = var.spot_min_size
      max_size       = var.spot_max_size
      desired_size   = var.spot_desired_size
    }
    on_demand = {
      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"
      min_size       = 1
      max_size       = var.on_demand_max_size
      desired_size   = 1
    }
  }
}
```

## GKE Cluster (GCP)

```hcl
resource "google_container_cluster" "this" {
  name     = "${var.project}-${var.environment}-gke"
  location = var.region

  initial_node_count       = 1
  remove_default_node_pool = true

  network    = var.network
  subnetwork = var.subnetwork

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = var.master_cidr
  }
}

resource "google_container_node_pool" "primary" {
  name       = "primary"
  cluster    = google_container_cluster.this.name
  location   = var.region
  node_count = var.node_count

  node_config {
    machine_type = var.machine_type
    spot         = var.use_spot
    disk_size_gb = var.disk_size_gb
  }

  autoscaling {
    min_node_count = var.min_nodes
    max_node_count = var.max_nodes
  }
}
```

## Remote State Reference

```hcl
# AWS S3 backend
data "terraform_remote_state" "core" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "core-infrastructure/terraform.tfstate"
    region = var.region
  }
}

# GCP GCS backend
data "terraform_remote_state" "core" {
  backend = "gcs"
  config = {
    bucket = var.state_bucket
    prefix = "core-infrastructure"
  }
}

# Azure azurerm backend
data "terraform_remote_state" "core" {
  backend = "azurerm"
  config = {
    resource_group_name  = var.state_rg
    storage_account_name = var.state_account
    container_name       = var.state_container
    key                  = "core-infrastructure.tfstate"
  }
}
```

## Feature Flags Pattern

```hcl
# tfvars
enable_spot_instances    = true
remove_legacy_nodegroup  = false  # ALWAYS default false for removal flags

# Conditional creation
resource "aws_eks_node_group" "spot" {
  count         = var.enable_spot_instances ? 1 : 0
  capacity_type = "SPOT"
  # ...
}

resource "aws_eks_node_group" "legacy" {
  count         = var.remove_legacy_nodegroup ? 0 : 1
  capacity_type = "ON_DEMAND"
  # ...
}
```

## Locals Pattern

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.naming_namespace
    ManagedBy   = "terraform"
  }
}
```
```

- [ ] **Step 2: Commit**

```bash
git add skills/infrastructure/references/examples.md
git commit -m "feat: add infrastructure code examples reference (multi-provider)"
```

---

### Task 9: Infrastructure Skill — references/modules.md

**Files:**
- Create: `skills/infrastructure/references/modules.md`

- [ ] **Step 1: Create `skills/infrastructure/references/modules.md`**

```markdown
# Module Patterns

Module patterns and architecture conventions. Read this when creating new modules or refactoring existing ones.

## Root Module Pattern

```
<root-module>/
+-- main.tf         # Module calls and orchestration
+-- variables.tf    # Input variables
+-- outputs.tf      # Outputs for cross-module references
+-- locals.tf       # Local values (optional)
+-- backend.tf      # State backend configuration
+-- providers.tf    # Provider configuration
+-- versions.tf     # Provider/Terraform version constraints
+-- prod.tfvars     # Production values
+-- dev.tfvars      # Development values (optional)
+-- README.md       # Module documentation
```

Each root module should use a different state key when sharing a backend:

```bash
terraform init -backend-config="key=core-infrastructure/terraform.tfstate"
terraform init -backend-config="key=iam/terraform.tfstate"
terraform init  # Root/default module
```

## Child Module Pattern

### Community modules (CloudPosse, HashiCorp, etc.)

```hcl
module "s3_bucket" {
  source  = "cloudposse/s3-bucket/aws"
  version = "4.5.0"

  namespace = var.naming_namespace
  stage     = var.environment
  name      = var.name
}
```

### Custom modules

```hcl
# modules/fargate/main.tf
resource "aws_ecs_task_definition" "this" {
  family = "${var.container_name}-${var.environment}-ecs-task"
  # ...
}
```

### Module call pattern

```hcl
module "backend_api" {
  source = "./modules/fargate"

  container_name  = "${var.naming_namespace}-api"
  environment     = var.environment
  # ...
}
```

## Cross-Module References

### Remote state pattern

```hcl
data "terraform_remote_state" "core" {
  backend = "s3"  # Match your state_backend
  config = {
    bucket = var.state_bucket
    key    = "core-infrastructure/terraform.tfstate"
    region = var.region
  }
}

# Usage
vpc_id = data.terraform_remote_state.core.outputs.vpc_id
```

### Expected outputs from core modules

```hcl
# Core infrastructure should expose:
output "vpc_id" {}
output "vpc_cidr_block" {}
output "public_subnet_ids" {}
output "private_subnet_ids" {}
output "route53_zone_id" {}
output "acm_certificate_arn" {}
output "kms_key_arn" {}
```

## Conditional Resource Creation

### Feature flags

```hcl
# tfvars
enable_feature_x        = true
remove_legacy_resource  = false  # ALWAYS default false for removal flags

# main.tf
module "feature_x" {
  count  = var.enable_feature_x ? 1 : 0
  source = "./feature-x"
}
```

### Zero-downtime migration pattern

```hcl
# Phase 1: Create new (enable_new=true, remove_old=false)
# Phase 2: Validate new resources
# Phase 3: Remove old (remove_old=true)

variable "enable_new_nodegroup" {
  type    = bool
  default = false
}

variable "remove_legacy_nodegroup" {
  type    = bool
  default = false  # NEVER default to true
}
```

## When to Create a New Module

**Create in `modules/`** when the component is reusable, has 3+ related resources, and follows a consistent pattern.

**Create as root module** when resources need isolated state, different teams manage them, or changes should not affect other infrastructure.

**Add to existing root module** when resources are tightly coupled, state isolation is not needed, or quick iteration matters more.

When in doubt, ask the user.
```

- [ ] **Step 2: Commit**

```bash
git add skills/infrastructure/references/modules.md
git commit -m "feat: add infrastructure module patterns reference"
```

---

### Task 10: Infrastructure Skill — references/naming.md

**Files:**
- Create: `skills/infrastructure/references/naming.md`

- [ ] **Step 1: Create `skills/infrastructure/references/naming.md`**

```markdown
# Naming Conventions

Naming standards for infrastructure resources. Read this when naming new resources or resolving naming ambiguities.

The naming pattern is configured in `.devops.yaml` under `naming`. Defaults:

```yaml
naming:
  pattern: "{namespace}-{stage}-{name}"
  namespace: harumi
  stage: production
```

## Resource Naming

### CloudPosse pattern: namespace-stage-name

```hcl
module "s3_bucket" {
  source    = "cloudposse/s3-bucket/aws"
  namespace = var.naming_namespace    # from config: naming.namespace
  stage     = var.environment         # from config: naming.stage
  name      = var.name                # resource purpose
}
# Result: harumi-production-data-lake
```

### Common patterns by provider

| Provider | Pattern | Example |
|----------|---------|---------|
| AWS (CloudPosse) | {namespace}-{stage}-{name} | harumi-production-data-lake |
| AWS (native) | {app}-{env}-{resource} | myapp-prod-vpc |
| GCP | {project}-{env}-{name} | myproject-prod-vpc |
| Azure | {prefix}-{env}-{name} | app-prod-rg |

## IAM Naming

### Users

| Element | Pattern | Example |
|---------|---------|---------|
| Directory | firstname-lastname | italo-rocha |
| Module name | firstname_lastname | italo_rocha |
| Username | firstname.lastname | italo.rocha |

### Service accounts

```hcl
user_name = "airbyte-service"    # purpose-specific
path      = "/service-accounts/"
```

### Groups

```hcl
group_name = "developers-production"  # {purpose}-{environment}
group_name = "admin"                  # Global groups: no environment suffix
```

### Roles

```hcl
name = "developer-role"       # {purpose}-role
name = "ecs-task-role"
name = "ecs-execution-role"
```

## Module and Directory Naming

```
modules/
+-- fargate/              # Lowercase, hyphenated
+-- s3/                   # Simple names for simple resources
+-- iam-developer-user/   # Hyphenated for compound names
+-- data-pipeline-glue/   # Hyphenated for features
```

### Module call naming

```hcl
# Pattern: {layer}_{purpose} or {resource_type}_{name}
module "backend_api" {}
module "frontend_platform" {}
```

## File Naming

### Standard module files

| File | Purpose |
|------|---------|
| main.tf | Primary resource definitions |
| variables.tf | Input variables |
| outputs.tf | Output values |
| versions.tf | Provider/Terraform versions |
| locals.tf | Local values (optional) |
| data.tf | Data sources (optional) |

## Variable Naming

```hcl
variable "environment" {
  description = "Deployment environment (production, staging, development)"
  type        = string
  default     = "production"
}

variable "region" {
  type    = string
}

# Boolean enable flags
variable "enable_encryption" { type = bool; default = true }

# Removal flags: ALWAYS default false
variable "remove_legacy_resource" { type = bool; default = false }
```

## Tag Naming

All resources MUST have:

```hcl
tags = {
  Environment = var.environment
  Project     = var.naming_namespace
  ManagedBy   = "terraform"
}
```

## Output Naming

Pattern: `resource_attribute`

```hcl
output "vpc_id" {}
output "vpc_cidr_block" {}
output "private_subnet_ids" {}
```
```

- [ ] **Step 2: Commit**

```bash
git add skills/infrastructure/references/naming.md
git commit -m "feat: add infrastructure naming conventions reference"
```

---

### Task 11: Infrastructure Skill — references/security.md

**Files:**
- Create: `skills/infrastructure/references/security.md`

- [ ] **Step 1: Create `skills/infrastructure/references/security.md`**

```markdown
# Security Patterns

Security patterns for cloud infrastructure. Read this when configuring security for any resource.

## Core Principles

1. **Encryption by default** — All resources with encryption support MUST enable it
2. **Private by default** — Resources in private subnets/networks unless justified
3. **Least privilege** — IAM/RBAC policies grant minimum required permissions
4. **No secrets in code** — All credentials in secrets manager (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault)
5. **Defense in depth** — Multiple security layers

## Secrets Management

### AWS

```hcl
# NEVER hardcode secrets
# Use managed credentials
resource "aws_db_instance" "this" {
  manage_master_user_password   = true
  master_user_secret_kms_key_id = var.kms_key_arn
}

# Reference existing secret
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "app-db-credentials"
}
```

### GCP

```hcl
data "google_secret_manager_secret_version" "db" {
  secret  = "db-password"
  project = var.project
}
```

### Azure

```hcl
data "azurerm_key_vault_secret" "db" {
  name         = "db-password"
  key_vault_id = var.key_vault_id
}
```

## Storage Security

### AWS S3

```hcl
module "s3_bucket" {
  source  = "cloudposse/s3-bucket/aws"

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true

  versioning_enabled = true
}
```

### GCP GCS

```hcl
resource "google_storage_bucket" "this" {
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"

  versioning {
    enabled = true
  }
}
```

### Azure Storage

```hcl
resource "azurerm_storage_account" "this" {
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
}
```

## Database Security

```hcl
# Universal patterns:
# - Always encrypt storage
# - Never make publicly accessible
# - Enable deletion protection in production
# - Use secrets manager for credentials
# - Restrict network access to application security groups only

# AWS RDS
resource "aws_db_instance" "this" {
  storage_encrypted             = true
  publicly_accessible           = false
  deletion_protection           = var.environment == "production"
  manage_master_user_password   = true
  backup_retention_period       = 7
  vpc_security_group_ids        = [aws_security_group.rds.id]
}

# Security group: only allow from application
resource "aws_security_group" "rds" {
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "PostgreSQL from application"
  }
}
```

## IAM Security

### Least privilege pattern

```hcl
# GOOD: Specific permissions and resources
data "aws_iam_policy_document" "s3_read" {
  statement {
    sid     = "ReadBucket"
    effect  = "Allow"
    actions = ["s3:GetObject", "s3:ListBucket"]
    resources = [
      "arn:aws:s3:::${var.bucket_name}",
      "arn:aws:s3:::${var.bucket_name}/*",
    ]
  }
}

# BAD: Overly permissive
actions   = ["s3:*"]
resources = ["*"]
```

### Service role pattern

Separate execution roles (pulls images, writes logs) from task roles (application permissions):

```hcl
# Execution role — platform permissions
resource "aws_iam_role" "execution" {
  name = "${var.name}-execution-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# Task role — application permissions
resource "aws_iam_role" "task" {
  name = "${var.name}-task-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

## Network Security

- **Public subnets** — Load balancers, NAT Gateways only
- **Private subnets** — Applications, databases, caches

### Security group patterns

```hcl
# Load balancer: Public-facing
resource "aws_security_group" "alb" {
  ingress { from_port = 443; to_port = 443; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 80; to_port = 80; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}

# Application: Internal only (traffic from LB)
resource "aws_security_group" "app" {
  ingress { from_port = 0; to_port = 65535; protocol = "tcp"; security_groups = [aws_security_group.alb.id] }
  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}
```

## Encryption

### KMS (AWS)

```hcl
resource "aws_kms_key" "this" {
  description             = "KMS key for ${var.naming_namespace} ${var.environment}"
  deletion_window_in_days = 30
  enable_key_rotation     = true  # ALWAYS enable
}
```

Used for: EBS, S3, RDS storage, Secrets Manager, CloudWatch Logs.

## Security Checklist

### Storage
- [ ] Public access blocked
- [ ] Encryption enabled
- [ ] Versioning enabled (data buckets)

### Databases
- [ ] Private subnet, not publicly accessible
- [ ] Storage encrypted
- [ ] Credentials in secrets manager
- [ ] Deletion protection (production)

### Cache
- [ ] Private subnet
- [ ] Encryption at rest and in transit
- [ ] Auth configured

### IAM
- [ ] Least privilege policies
- [ ] No wildcard resources unless justified
- [ ] Service accounts have specific permissions

### Network
- [ ] Databases in private subnets
- [ ] Load balancers with HTTPS
- [ ] Security groups use specific source references
```

- [ ] **Step 2: Commit**

```bash
git add skills/infrastructure/references/security.md
git commit -m "feat: add infrastructure security patterns reference (multi-provider)"
```

---

### Task 12: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Create `README.md`**

```markdown
# Harumi DevOps Plugin

DevOps skills for [Claude Code](https://claude.ai/code), [Cursor](https://cursor.com), and GitHub Copilot. Provides infrastructure, Kubernetes, CI/CD, and cloud operations guidance through an extensible skill system.

## Installation

### Claude Code

```bash
claude plugin add harumi-io/harumi-devops-plugin
```

### Cursor

Clone the repository and add the plugin path in Cursor settings.

### GitHub Copilot

Clone the repository. The session-start hook auto-detects the Copilot environment.

## Configuration

Create a `.devops.yaml` in your repository root to configure the plugin for your stack:

```yaml
cloud:
  provider: aws          # aws | gcp | azure
  region: us-east-1

terraform:
  version: "1.5.7"
  state_backend: s3
  var_file: prod.tfvars

naming:
  pattern: "{namespace}-{stage}-{name}"
  namespace: mycompany
  stage: production
```

See `config/default.devops.yaml` for all available options.

If no `.devops.yaml` is found, the plugin uses its built-in defaults.

## Skills

### Available (MVP)

| Skill | Description |
|-------|-------------|
| `infrastructure` | Terraform/IaC management with multi-provider support (AWS, GCP, Azure) |

### Planned

| Skill | Description |
|-------|-------------|
| `kubernetes` | K8s manifest management, Helm, ArgoCD/Flux |
| `cicd` | CI/CD pipeline authoring and deployment patterns |
| `cost-optimization` | Resource right-sizing and cost analysis |
| `observability` | Monitoring, alerting, and dashboard management |
| `security-operations` | IAM audit, secrets rotation, compliance |
| `containers` | Dockerfile optimization, image management |

## How It Works

1. **Session start** — The hook loads the bootstrap skill and merges your `.devops.yaml` config
2. **Skill triggering** — The bootstrap skill tells Claude when to invoke domain-specific skills
3. **Safety rules** — Destructive operations (apply, destroy, delete) always require user confirmation via handoff

## Relationship to Superpowers

This plugin is **domain-oriented** (DevOps knowledge). [Superpowers](https://github.com/obra/superpowers) is **process-oriented** (TDD, debugging, planning). They work together — use superpowers for workflow, devops-plugin for domain expertise.

## License

MIT
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "feat: add README with installation and usage instructions"
```

---

### Task 13: Final Verification

- [ ] **Step 1: Verify complete file structure**

```bash
find . -not -path './.git/*' -not -name '.git' | sort
```

Expected output:
```
.
./.claude-plugin
./.claude-plugin/plugin.json
./.cursor-plugin
./.cursor-plugin/plugin.json
./agents
./agents/.gitkeep
./CHANGELOG.md
./config
./config/default.devops.yaml
./hooks
./hooks/hooks-cursor.json
./hooks/hooks.json
./hooks/session-start
./LICENSE
./package.json
./README.md
./skills
./skills/infrastructure
./skills/infrastructure/references
./skills/infrastructure/references/architecture.md
./skills/infrastructure/references/examples.md
./skills/infrastructure/references/modules.md
./skills/infrastructure/references/naming.md
./skills/infrastructure/references/security.md
./skills/infrastructure/references/workflow.md
./skills/infrastructure/SKILL.md
./skills/using-devops
./skills/using-devops/SKILL.md
```

- [ ] **Step 2: Test session-start hook end-to-end**

```bash
CLAUDE_PLUGIN_ROOT="$(pwd)" ./hooks/session-start | python3 -c "
import sys, json
data = json.load(sys.stdin)
ctx = data['hookSpecificOutput']['additionalContext']
assert 'DevOps Plugin' in ctx, 'Missing bootstrap skill'
assert 'devops.yaml' in ctx, 'Missing config'
print('ALL CHECKS PASSED')
"
```

Expected: `ALL CHECKS PASSED`

- [ ] **Step 3: Verify all files are committed**

```bash
git status
```

Expected: clean working tree.
