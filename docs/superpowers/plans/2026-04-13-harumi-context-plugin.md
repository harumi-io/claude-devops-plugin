# Harumi Context Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Transform harumi-devops-plugin from a generic multi-provider DevOps plugin into a harumi-specific context-aware plugin with docs sync and drift detection.

**Architecture:** Keep domain expertise in skills, remove generic multi-provider abstractions, replace `.devops.yaml` config system with `harumi.yaml`. Add `sync-docs` skill for documentation maintenance. Add drift detection to session-start hook.

**Tech Stack:** Bash (hooks), Markdown (skills), YAML (config)

**Spec:** `docs/superpowers/specs/2026-04-13-harumi-context-plugin-design.md`

---

### Task 1: Delete Generic Config and Setup Skill

**Files:**
- Delete: `config/default.devops.yaml`
- Delete: `skills/setup-devops-config/` (entire directory — SKILL.md + evals/)

- [ ] **Step 1: Delete the default config file**

```bash
rm config/default.devops.yaml
```

- [ ] **Step 2: Delete the setup-devops-config skill directory**

```bash
rm -rf skills/setup-devops-config/
```

- [ ] **Step 3: Remove the empty config directory if nothing else is in it**

```bash
rmdir config/ 2>/dev/null || true
```

- [ ] **Step 4: Verify deletions**

```bash
ls config/ 2>/dev/null && echo "config/ still has files" || echo "config/ gone — OK"
ls skills/setup-devops-config/ 2>/dev/null && echo "setup-devops-config still exists" || echo "setup-devops-config gone — OK"
```

Expected: Both directories gone.

- [ ] **Step 5: Add .harumi-last-sync to .gitignore**

In `.gitignore`, add a new line:

```
.harumi-last-sync
```

The file currently contains only `.claude/worktrees/`. After edit:

```
.claude/worktrees/
.harumi-last-sync
```

- [ ] **Step 6: Create docs/architecture/ and docs/runbooks/ directories**

```bash
mkdir -p docs/architecture docs/runbooks
touch docs/architecture/.gitkeep docs/runbooks/.gitkeep
```

- [ ] **Step 7: Commit**

```bash
git add -A config/ skills/setup-devops-config/ .gitignore docs/architecture/.gitkeep docs/runbooks/.gitkeep
git commit -m "chore: remove generic config system and setup-devops-config skill

Delete config/default.devops.yaml and skills/setup-devops-config/.
Add .harumi-last-sync to .gitignore. Create docs/architecture/ and
docs/runbooks/ directories for sync-docs skill."
```

---

### Task 2: Create sync-docs Skill

**Files:**
- Create: `skills/sync-docs/SKILL.md`

- [ ] **Step 1: Create the sync-docs skill file**

Write `skills/sync-docs/SKILL.md` with this content:

```markdown
---
name: sync-docs
description: "Maintain repo documentation accuracy by reading code, infrastructure state, and cluster state. Use when: (1) Drift detected at session start, (2) User asks to sync/update docs, (3) After infrastructure or K8s changes are applied."
---

# Sync Docs

Keep repo documentation accurate by reading actual code, infrastructure state, and live cluster state. This skill maintains two categories of docs:

- **Generated docs** — auto-updated silently (no prompt needed)
- **Human-authored docs** — proposed edits shown to user for approval

## Targets

| Target | Source of Truth | Classification |
|--------|----------------|----------------|
| `harumi.yaml` | Terraform files, K8s manifests, cluster state | Generated |
| `docs/architecture/clusters.md` | Live cluster state across all contexts | Generated |
| `docs/architecture/services.md` | ArgoCD apps, Helm releases, deployments | Generated |
| `docs/architecture/infrastructure.md` | Terraform state, modules, outputs | Generated |
| `docs/architecture/networking.md` | Ingresses, services, domains | Generated |
| `docs/architecture/observability.md` | Monitoring stack state in monitoring namespace | Generated |
| `README.md` | All of the above | Human-authored |
| `CLAUDE.md` | Repo structure, commands, conventions | Human-authored |
| `AGENTS.md` | Infrastructure modules, operational context | Human-authored |
| `docs/runbooks/*` | Operational state changes | Human-authored |

## Workflow

Follow these steps in order.

### Step 1: Scan Repos

Read the codebase to understand current state:

- **Terraform**: Read all `.tf` files. Extract modules, resources, outputs, variables. Note state backend paths.
- **K8s manifests**: Read manifests in the harumi-k8s repo (if accessible). Extract namespaces, deployments, services, ingresses, ArgoCD Applications.
- **CI/CD**: Read `.github/workflows/*.yml`. Extract pipeline structure, deployment targets.
- **Config**: Read current `harumi.yaml` if it exists. Note any values that may need updating.

### Step 2: Query Live Cluster State (Read-Only)

Use locally configured kubectl contexts for read-only access. Run these commands for each cluster context defined in `harumi.yaml`:

```bash
# Namespaces and workloads
kubectl get namespaces --context <context>
kubectl get deployments --all-namespaces --context <context> -o wide
kubectl get services --all-namespaces --context <context>
kubectl get ingress --all-namespaces --context <context>

# ArgoCD apps (if argocd CLI available)
argocd app list --output json 2>/dev/null || echo "argocd CLI not available"

# Helm releases (if helm CLI available)
helm list --all-namespaces 2>/dev/null || echo "helm CLI not available"

# Observability stack health
kubectl get pods -n monitoring --context <context> 2>/dev/null || echo "monitoring namespace not found"
```

**If kubectl is unavailable or contexts are not configured:**
- Log: "Cluster access unavailable — proceeding with repo-only data. Cluster-derived docs (clusters.md, services.md, networking.md) will be incomplete."
- Continue with Steps 3 and 4 using only repo-scanned data.
- Do NOT fail or stop.

**NEVER run write commands.** Allowed: `get`, `describe`, `logs`, `top`. Forbidden: `apply`, `delete`, `edit`, `patch`, `create`.

### Step 3: Update Generated Docs

For each generated target:

1. Build the new content from scanned + live data
2. If the target file exists, diff against current content
3. If changes detected, write the file silently (no user prompt)
4. If no changes, skip

After all generated docs are updated, write the current git HEAD to `.harumi-last-sync`:

```
commit=<current HEAD SHA>
timestamp=<current UTC time>
```

### Step 4: Check Human-Authored Docs

For each human-authored target:

1. Read the current file content
2. Compare claims in the doc against the current state gathered in Steps 1-2
3. Identify stale or inaccurate sections (e.g., wrong cluster count, outdated commands, missing services)
4. For each stale section, present the proposed edit:

```
[filename] line [N] says "[current text]" but the current state is [new state].
Proposed edit: [show the diff]
Apply? (yes / skip)
```

5. Wait for user response before proceeding to the next proposed edit
6. If user says "skip", move to the next edit without changing the file

### Step 5: Summary

After all targets are processed, print a summary:

```
Sync complete.

Updated: [list of generated files that changed]
Unchanged: [list of generated files with no drift]
Proposed: [N] edits to [human-authored files] ([M] applied, [K] skipped)
Skipped: [human-authored files with no drift detected]
Cluster access: [available / unavailable]
```

## What NOT to Do

- Do not run any write commands against clusters (`kubectl apply`, `helm install`, `argocd app sync`)
- Do not modify human-authored docs without showing the diff and getting explicit approval
- Do not fail if cluster access is unavailable — degrade gracefully
- Do not invent information — if a value cannot be determined from code or cluster, omit it and note the gap
```

- [ ] **Step 2: Verify the file is valid**

```bash
head -5 skills/sync-docs/SKILL.md
```

Expected: The YAML frontmatter with `name: sync-docs`.

- [ ] **Step 3: Commit**

```bash
git add skills/sync-docs/SKILL.md
git commit -m "feat(sync-docs): add docs sync skill

Replaces setup-devops-config. Maintains generated docs (architecture/,
harumi.yaml) automatically and proposes edits to human-authored docs
(README, CLAUDE.md, AGENTS.md, runbooks) with user approval."
```

---

### Task 3: Update Session-Start Hook with Drift Detection

**Files:**
- Modify: `hooks/session-start`

- [ ] **Step 1: Rewrite the session-start hook**

Replace the entire content of `hooks/session-start` with:

```bash
#!/usr/bin/env bash
# SessionStart hook for harumi-devops-plugin
# Loads the bootstrap skill, reads harumi.yaml config, and detects drift

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PLUGIN_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

# --- Read bootstrap skill ---
using_devops_content=$(cat "${PLUGIN_ROOT}/skills/using-devops/SKILL.md" 2>&1 || echo "Error reading using-devops skill")

# --- Read harumi.yaml config ---
user_config=""
repo_config_path="${PWD}/harumi.yaml"

if [ -f "$repo_config_path" ]; then
    user_config=$(cat "$repo_config_path" 2>/dev/null || true)
    config_source="harumi.yaml"
else
    config_source="not found"
    user_config="# No harumi.yaml found. Run the sync-docs skill to generate one."
fi

# --- Drift detection ---
drift_summary=""
last_sync_path="${PWD}/.harumi-last-sync"

if [ -f "$repo_config_path" ] && command -v git &>/dev/null && git rev-parse --git-dir &>/dev/null; then
    current_head=$(git rev-parse --short HEAD 2>/dev/null || echo "")

    if [ -f "$last_sync_path" ] && [ -n "$current_head" ]; then
        saved_commit=$(grep '^commit=' "$last_sync_path" 2>/dev/null | cut -d= -f2 || echo "")

        if [ -n "$saved_commit" ] && [ "$saved_commit" != "$current_head" ]; then
            # Drift detected — summarize changes
            commit_count=$(git rev-list --count "${saved_commit}..HEAD" 2>/dev/null || echo "?")
            changed_dirs=$(git diff --name-only "${saved_commit}..HEAD" 2>/dev/null | sed 's|/.*||' | sort -u | tr '\n' ', ' | sed 's/,$//' || echo "unknown")

            # Classify changed files
            generated_changed=""
            human_changed=""
            changed_files=$(git diff --name-only "${saved_commit}..HEAD" 2>/dev/null || echo "")

            while IFS= read -r file; do
                case "$file" in
                    docs/architecture/*|harumi.yaml)
                        generated_changed="${generated_changed:+$generated_changed, }$file"
                        ;;
                    README.md|CLAUDE.md|AGENTS.md|docs/runbooks/*)
                        human_changed="${human_changed:+$human_changed, }$file"
                        ;;
                esac
            done <<< "$changed_files"

            drift_summary="DRIFT DETECTED: ${commit_count} commits since last session. Changes in: ${changed_dirs}."
            [ -n "$generated_changed" ] && drift_summary="${drift_summary} Auto-update needed: ${generated_changed}."
            [ -n "$human_changed" ] && drift_summary="${drift_summary} May need review: ${human_changed}."
            drift_summary="${drift_summary} Run sync-docs skill before proceeding with other work."
        fi

        # Update last-sync
        printf 'commit=%s\ntimestamp=%s\n' "$current_head" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" > "$last_sync_path"

    elif [ ! -f "$last_sync_path" ] && [ -n "$current_head" ]; then
        # First run — write initial state, no drift check
        printf 'commit=%s\ntimestamp=%s\n' "$current_head" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" > "$last_sync_path"
    fi
fi

# --- Build context ---
config_block="## Active Configuration (source: ${config_source})\n\n\`\`\`yaml\n${user_config}\n\`\`\`"

if [ -n "$drift_summary" ]; then
    config_block="${config_block}\n\n## Drift Detection\n\n${drift_summary}"
fi

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

- [ ] **Step 2: Verify the hook is executable and has valid syntax**

```bash
chmod +x hooks/session-start
bash -n hooks/session-start && echo "Syntax OK" || echo "Syntax error"
```

Expected: `Syntax OK`

- [ ] **Step 3: Commit**

```bash
git add hooks/session-start
git commit -m "feat(hooks): add drift detection to session-start

Read harumi.yaml instead of .devops.yaml. On session start, compare
.harumi-last-sync commit SHA with HEAD. If drift detected, classify
changed files as generated vs human-authored and inject a summary
telling the AI to run sync-docs before other work."
```

---

### Task 4: Update Bootstrap Skill (using-devops)

**Files:**
- Modify: `skills/using-devops/SKILL.md`

This task rewrites the bootstrap skill to remove generic references, add multi-repo awareness, cluster read-access rules, drift trigger, and observability endpoints.

- [ ] **Step 1: Rewrite the bootstrap skill**

Replace the entire content of `skills/using-devops/SKILL.md` with:

```markdown
---
name: using-devops
description: "Bootstrap skill for harumi-devops-plugin. Injected at session start. Announces available DevOps skills, loads harumi.yaml config, defines trigger rules, enforces safety rules, and detects drift."
---

# DevOps Plugin

You have the **harumi-devops-plugin** installed. This plugin provides DevOps skills for harumi's infrastructure and Kubernetes operations.

## Managed Repositories

This plugin manages two repositories:

| Repo | Purpose |
|------|---------|
| `harumi-io/infrastructure` | Terraform IaC (AWS) — VPC, ECS, EKS, RDS, IAM, DNS |
| `harumi-io/harumi-k8s` | Kubernetes manifests, ArgoCD apps, Helm values, Grafana dashboards |

Read `harumi.yaml` (injected at session start) for cluster names, contexts, endpoints, and naming conventions.

## Drift Detection

If the drift detection section below reports drift, invoke `harumi-devops-plugin:sync-docs` **before** proceeding with any other work. This ensures documentation is up to date before making further changes.

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

Skills use locally configured kubectl contexts for **read-only** cluster access.

**Allowed commands:**
- `kubectl get`, `kubectl describe`, `kubectl logs`, `kubectl top`
- `argocd app get`, `argocd app list`
- `helm list`, `helm get values`
- `curl` against observability endpoints (from `harumi.yaml` observability.endpoints)

**Forbidden commands (require handoff to user):**
- `kubectl apply`, `kubectl delete`, `kubectl edit`, `kubectl patch`, `kubectl create`
- `helm install`, `helm upgrade`, `helm uninstall`
- `argocd app sync`, `argocd app delete`

## Universal Safety Rules (NON-NEGOTIABLE)

These apply to ALL DevOps skills:

1. **Never run `terraform apply` or `terraform destroy`** — Always provide a handoff with the exact command for the user to execute
2. **Never `kubectl delete` or any write operation without explicit user confirmation** — this applies to ALL environments (production, staging, development). No exceptions.
3. **Never push images to production registries without confirmation**
4. **Always verify current state before making changes** — Use `aws` CLI and `kubectl` to confirm resource existence and configuration
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

The active `harumi.yaml` config (loaded at session start) tells you:
- **AWS account** — account ID, region, alias
- **Terraform settings** — version, state backend, state bucket, var file, module paths
- **Clusters** — names, contexts, environments, domains, registries
- **ArgoCD** — gitops repo, app-of-apps paths
- **Observability endpoints** — Prometheus, Grafana, Loki, Tempo, Alertmanager URLs
- **Naming pattern** — how resources are named

Read config values to adapt your guidance to the specific context.
```

- [ ] **Step 2: Verify the frontmatter is valid**

```bash
head -4 skills/using-devops/SKILL.md
```

Expected: YAML frontmatter with `name: using-devops`.

- [ ] **Step 3: Commit**

```bash
git add skills/using-devops/SKILL.md
git commit -m "feat(using-devops): replace generic bootstrap with harumi-specific context

Remove multi-provider references, setup-devops-config trigger, and
generic config guidance. Add multi-repo awareness, cluster read-access
rules, drift detection trigger, sync-docs trigger, and observability
endpoint guidance."
```

---

### Task 5: Clean Up Infrastructure Skill

**Files:**
- Modify: `skills/infrastructure/SKILL.md`

- [ ] **Step 1: Update the frontmatter description**

Replace:
```
description: "Write and manage Terraform/IaC infrastructure code following project conventions. Use when: (1) Creating, modifying, or reviewing Terraform configurations, (2) Working with cloud infrastructure (AWS, GCP, Azure), (3) Managing IaC changes across modules, (4) Planning infrastructure migrations or zero-downtime changes, (5) Reviewing security patterns, cost implications, or naming conventions."
```

With:
```
description: "Write and manage Terraform/IaC infrastructure code following project conventions. Use when: (1) Creating, modifying, or reviewing Terraform configurations, (2) Working with AWS infrastructure, (3) Managing IaC changes across modules, (4) Planning infrastructure migrations or zero-downtime changes, (5) Reviewing security patterns, cost implications, or naming conventions."
```

- [ ] **Step 2: Update the intro paragraph**

Replace:
```
Act as a **Principal Platform Engineer** for the project's cloud infrastructure. Read the active `.devops.yaml` config (injected at session start) for provider, region, naming, and state backend details.
```

With:
```
Act as a **Principal Platform Engineer** for harumi's AWS infrastructure. Read the active `harumi.yaml` config (injected at session start) for region, naming, state backend, and module paths.
```

- [ ] **Step 3: Remove GCP and Azure CLI blocks and the provider note**

Replace the entire CLI verification section (lines 16-40):
```
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
```

With:
```
```bash
aws ec2 describe-vpcs --vpc-ids vpc-xxxxx
aws ecs describe-services --cluster [cluster] --services [name]
aws rds describe-db-instances --db-instance-identifier [name]
aws iam get-role --role-name [name]
aws eks describe-cluster --name [cluster-name]
```
```

- [ ] **Step 4: Update naming quick reference**

Replace:
```
Read the `naming` section of `.devops.yaml` for the project's naming pattern. Common patterns:

- CloudPosse: `{namespace}-{stage}-{name}` (e.g., harumi-production-data-lake)
- GCP: `{project}-{env}-{name}` (e.g., myproject-prod-vpc)
- Azure: `{prefix}-{env}-{name}` (e.g., app-prod-rg)
```

With:
```
Read the `naming` section of `harumi.yaml` for the naming pattern:

- CloudPosse: `{namespace}-{stage}-{name}` (e.g., harumi-production-data-lake)
```

- [ ] **Step 5: Update cross-module references**

Replace:
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

With:
```hcl
data "terraform_remote_state" "core" {
  backend = "s3"
  config = {
    bucket = "[state-bucket from harumi.yaml terraform.state_bucket]"
    key    = "[module]/terraform.tfstate"
    region = "[region from harumi.yaml aws.region]"
  }
}
```

- [ ] **Step 6: Update `.devops.yaml` references in Naming section (line 57)**

Replace:
```
Read the `naming` section of `.devops.yaml`
```

With (wherever it appears in the file):
```
Read the `naming` section of `harumi.yaml`
```

- [ ] **Step 7: Verify no `.devops.yaml`, GCP, Azure, or `gcloud` references remain**

```bash
grep -in 'devops\.yaml\|gcloud\|azurerm\|azure\|GCP' skills/infrastructure/SKILL.md
```

Expected: No matches.

- [ ] **Step 8: Commit**

```bash
git add skills/infrastructure/SKILL.md
git commit -m "refactor(infrastructure): remove multi-provider abstractions

Remove GCP and Azure CLI blocks, naming patterns, and config references.
Replace .devops.yaml references with harumi.yaml."
```

---

### Task 6: Clean Up Infrastructure References

**Files:**
- Modify: `skills/infrastructure/references/examples.md`
- Modify: `skills/infrastructure/references/naming.md`
- Modify: `skills/infrastructure/references/security.md`
- Modify: `skills/infrastructure/references/workflow.md`

- [ ] **Step 1: Clean examples.md — remove GCS, Azure Storage, Cloud Run, GKE, and non-S3 remote state**

In `skills/infrastructure/references/examples.md`:

Remove the entire `## GCS Bucket (GCP)` section (lines 26-41).

Remove the entire `## Azure Storage Account` section (lines 43-56).

Remove the entire `## Cloud Run Service (GCP)` section (lines 126-163).

Remove the entire `## GKE Cluster (GCP)` section (lines 200-237).

In the `## Remote State Reference` section, remove the GCP GCS and Azure azurerm blocks (lines 252-270), keeping only the AWS S3 block.

- [ ] **Step 2: Clean naming.md — remove GCP and Azure from provider table**

In `skills/infrastructure/references/naming.md`:

Replace the `### Common patterns by provider` table:
```
| Provider | Pattern | Example |
|----------|---------|---------|
| AWS (CloudPosse) | {namespace}-{stage}-{name} | harumi-production-data-lake |
| AWS (native) | {app}-{env}-{resource} | myapp-prod-vpc |
| GCP | {project}-{env}-{name} | myproject-prod-vpc |
| Azure | {prefix}-{env}-{name} | app-prod-rg |
```

With:
```
| Pattern | Example |
|---------|---------|
| CloudPosse: {namespace}-{stage}-{name} | harumi-production-data-lake |
| Native: {app}-{env}-{resource} | myapp-prod-vpc |
```

Also replace the reference to `.devops.yaml`:
```
The naming pattern is configured in `.devops.yaml` under `naming`. Defaults:
```

With:
```
The naming pattern is configured in `harumi.yaml` under `naming`. Defaults:
```

- [ ] **Step 3: Clean security.md — remove GCP and Azure secrets and storage blocks**

In `skills/infrastructure/references/security.md`:

Replace line 10:
```
4. **No secrets in code** — All credentials in secrets manager (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault)
```

With:
```
4. **No secrets in code** — All credentials in AWS Secrets Manager
```

Remove the `### GCP` secrets section (lines 31-38).

Remove the `### Azure` secrets section (lines 40-45).

Remove the `### GCP GCS` storage section (lines 66-77).

Remove the `### Azure Storage` storage section (lines 79-86).

- [ ] **Step 4: Clean workflow.md — remove GCP and Azure CLI, pricing, and multi-provider references**

In `skills/infrastructure/references/workflow.md`:

Remove the entire `### GCP` CLI section (lines 28-37).

Remove the entire `### Azure` CLI section (lines 39-50).

In the downtime risk table, simplify multi-provider resource names:
- `RDS / Cloud SQL / Azure SQL` → `RDS`
- `Redis / Memorystore / Azure Cache` → `ElastiCache Redis`
- `ECS / Cloud Run / ACI` → `ECS Fargate`
- `EKS / GKE / AKS Node Group` → `EKS Node Group`
- `ALB / Cloud LB / App Gateway` → `ALB`
- `S3 / GCS / Azure Storage` → `S3`

In zero-downtime patterns:
- `ECS/Cloud Run` → `ECS`
- `EKS/GKE/AKS Node Groups` → `EKS Node Groups`

Replace `.devops.yaml` reference on line 58:
```
- **Naming**: What naming pattern does this project use? (Check `.devops.yaml` naming section)
```

With:
```
- **Naming**: What naming pattern does this project use? (Check `harumi.yaml` naming section)
```

Remove the entire `### GCP Pricing Reference` section (lines 169-186).

Remove the entire `### Azure Pricing Reference` section (lines 188-202).

Replace `terraform plan -var-file=[var_file from .devops.yaml]` on line 220:
```
terraform plan -var-file=[var_file from .devops.yaml]
```

With:
```
terraform plan -var-file=[var_file from harumi.yaml]
```

In Multi-Module Changes section line 277, simplify:
```
4. **Applications** last (ECS, Lambda, Cloud Run, etc.)
```

With:
```
4. **Applications** last (ECS, Lambda, etc.)
```

- [ ] **Step 5: Verify no generic provider references remain in any reference file**

```bash
grep -rn 'gcloud\|azurerm\|azure\|GCP\|Cloud Run\|GKE\|Cloud SQL\|App Gateway\|ACI\|AKS\|Memorystore\|Cloud LB\|\.devops\.yaml' skills/infrastructure/references/
```

Expected: No matches (or only false positives in non-provider contexts).

- [ ] **Step 6: Commit**

```bash
git add skills/infrastructure/references/
git commit -m "refactor(infrastructure): remove GCP/Azure from reference docs

Remove GCP and Azure examples, pricing tables, CLI blocks, and naming
patterns from examples.md, naming.md, security.md, and workflow.md.
Simplify multi-provider resource names to AWS-only equivalents."
```

---

### Task 7: Clean Up Observability Skill

**Files:**
- Modify: `skills/observability/SKILL.md`

- [ ] **Step 1: Update the intro paragraph**

Replace:
```
Act as a **Senior SRE / Observability Engineer**. Read the active `.devops.yaml` config (injected at session start) for the observability stack: metrics, dashboards, logs, traces.
```

With:
```
Act as a **Senior SRE / Observability Engineer**. Read the active `harumi.yaml` config (injected at session start) for observability endpoints: Prometheus, Grafana, Loki, Tempo, Alertmanager.
```

- [ ] **Step 2: Update the CLI tools priority list**

Replace:
```
1. Dedicated CLI if available (`promtool`, `logcli`, `amtool`)
2. Cloud provider CLI (`aws`, `gcloud`, `az`)
3. `kubectl` for K8s-level investigation
4. `curl` against HTTP APIs as fallback
```

With:
```
1. Dedicated CLI if available (`promtool`, `logcli`, `amtool`)
2. `aws` CLI for AWS-level investigation
3. `kubectl` for K8s-level investigation
4. `curl` against observability endpoints (from `harumi.yaml` observability.endpoints)
```

- [ ] **Step 3: Update the `.devops.yaml` reference in Author mode workflow**

Replace:
```
1. **Read context** — Check `.devops.yaml` for stack, scan the project's `docs/references/` for service topology and runbooks
```

With:
```
1. **Read context** — Check `harumi.yaml` for endpoints, scan the project's `docs/references/` for service topology and runbooks
```

- [ ] **Step 4: Update the `.devops.yaml` reference in Investigate mode workflow**

Replace:
```
2. **Read context** — Check `.devops.yaml` for stack, scan the project's `docs/references/` for service topology and runbooks
```

With:
```
2. **Read context** — Check `harumi.yaml` for endpoints, scan the project's `docs/references/` for service topology and runbooks
```

- [ ] **Step 5: Update the Grafana note**

Replace:
```
> **Note:** The UID pattern and datasource UID above are Harumi defaults from `.devops.yaml`. Read the `observability` section of `.devops.yaml` for overrides, or confirm with the user if working on a different stack.
```

With:
```
> **Note:** The UID pattern and datasource UID above are Harumi defaults from `harumi.yaml`.
```

- [ ] **Step 6: Verify no `.devops.yaml`, `gcloud`, or `az` references remain**

```bash
grep -in 'devops\.yaml\|gcloud\|azure\| az ' skills/observability/SKILL.md
```

Expected: No matches.

- [ ] **Step 7: Commit**

```bash
git add skills/observability/SKILL.md
git commit -m "refactor(observability): replace generic config refs with harumi.yaml

Remove multi-provider CLI references. Point to harumi.yaml endpoints
instead of .devops.yaml stack config."
```

---

### Task 8: Update README and Package Version

**Files:**
- Modify: `README.md`
- Modify: `package.json`
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Update README.md**

The README was recently updated but still references `.devops.yaml`. Rewrite the Configuration section and update the "How It Works" section.

In the Configuration section, replace everything from `Create a .devops.yaml` through the end of the config YAML block and the two lines after it with the new `harumi.yaml` schema and description. Key changes:

- Replace all `.devops.yaml` references with `harumi.yaml`
- Update the example YAML to match the `harumi.yaml` schema from the spec (including `repos`, `aws`, `argocd`, `observability.endpoints`, `docs` sections)
- Replace `See config/default.devops.yaml for all available options.` with a note that the sync-docs skill can generate it
- Replace `setup-devops-config` mention with `sync-docs`

In the Skills tables:
- Replace `setup-devops-config` row with `sync-docs` row: `| sync-docs | Keep repo docs in sync with actual infrastructure and cluster state |`

In the "How It Works" section, update:
- Step 1: mention `harumi.yaml` instead of `.devops.yaml`
- Add drift detection as a step
- Reference sync-docs instead of setup-devops-config

- [ ] **Step 2: Bump version in package.json**

Replace:
```json
"version": "0.7.0"
```

With:
```json
"version": "0.8.0"
```

- [ ] **Step 3: Add CHANGELOG entry**

Add a new entry at the top of the CHANGELOG (after any existing header):

```markdown
## 0.8.0

### Breaking Changes

- Removed `.devops.yaml` config system and `config/default.devops.yaml` — replaced by `harumi.yaml` per-repo manifest
- Removed `setup-devops-config` skill — replaced by `sync-docs`
- Removed generic multi-provider abstractions (GCP, Azure) from all skills — plugin is now AWS-only

### Added

- `sync-docs` skill — maintains generated docs (`docs/architecture/*`, `harumi.yaml`) automatically and proposes edits to human-authored docs (`README.md`, `CLAUDE.md`, `AGENTS.md`, `docs/runbooks/*`) with user approval
- Drift detection on session start — compares `.harumi-last-sync` commit SHA with HEAD, classifies changed files, triggers sync-docs when drift is detected
- Multi-repo awareness — bootstrap skill declares `harumi-io/infrastructure` and `harumi-io/harumi-k8s` as managed repos
- Cluster read-access rules — explicit allowed/forbidden kubectl command lists in bootstrap skill
- Observability endpoints in `harumi.yaml` — skills can query Prometheus, Grafana, Loki, Tempo, Alertmanager directly
- `docs/architecture/` directory for generated architecture docs
- `docs/runbooks/` directory for operational runbooks

### Changed

- Session-start hook reads `harumi.yaml` instead of `.devops.yaml`
- Bootstrap skill (`using-devops`) updated with harumi-specific context
- Infrastructure skill and references cleaned of GCP/Azure content
- Observability skill updated to reference `harumi.yaml` endpoints
```

- [ ] **Step 4: Commit**

```bash
git add README.md package.json CHANGELOG.md
git commit -m "chore: bump version to 0.8.0 and update README/CHANGELOG

Update README for harumi.yaml config, sync-docs skill, and drift
detection. Remove .devops.yaml references. Add CHANGELOG entry for
0.8.0 breaking changes and new features."
```

---

### Task 9: Final Verification

- [ ] **Step 1: Verify no `.devops.yaml` references remain anywhere in skills/**

```bash
grep -rn '\.devops\.yaml' skills/ hooks/
```

Expected: No matches.

- [ ] **Step 2: Verify no GCP/Azure provider references remain in skills/**

```bash
grep -rn 'gcloud\|azurerm\| az \|GCP\|Azure' skills/ | grep -v 'evals/'
```

Expected: No matches (evals may still have fixtures but those are test data).

- [ ] **Step 3: Verify `config/` directory is gone**

```bash
ls config/ 2>/dev/null && echo "FAIL: config/ still exists" || echo "OK: config/ removed"
```

Expected: `OK: config/ removed`

- [ ] **Step 4: Verify `setup-devops-config` is gone**

```bash
ls skills/setup-devops-config/ 2>/dev/null && echo "FAIL: still exists" || echo "OK: removed"
```

Expected: `OK: removed`

- [ ] **Step 5: Verify new directories exist**

```bash
ls docs/architecture/.gitkeep docs/runbooks/.gitkeep skills/sync-docs/SKILL.md
```

Expected: All three files listed.

- [ ] **Step 6: Verify session-start hook syntax**

```bash
bash -n hooks/session-start && echo "Syntax OK" || echo "Syntax error"
```

Expected: `Syntax OK`

- [ ] **Step 7: Verify all skill frontmatter is valid YAML**

```bash
for f in skills/*/SKILL.md; do
  head -4 "$f" | grep -q '^name:' && echo "OK: $f" || echo "FAIL: $f"
done
```

Expected: All OK.
