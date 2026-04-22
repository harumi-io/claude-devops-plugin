---
name: sync-docs
description: "Maintain repo documentation accuracy by reading code, infrastructure state, and cluster state. Use when: (1) Drift detected at session start, (2) User asks to sync/update docs, (3) After infrastructure or K8s changes are applied."
---

# Sync Docs

Keep repo documentation accurate by reading actual code, infrastructure state, and live cluster state. This skill maintains two categories of docs:

- **Generated docs** — auto-updated silently (no prompt needed)
- **Human-authored docs** — proposed edits shown to user for approval

## Source of Truth

Each live source is queried **independently**. Partial reachability is normal — AWS may be reachable when Kubernetes is not, or vice versa. Apply this logic per value, not per session:

1. **Live state per source** (highest priority per surface)
   - **AWS**: when the `aws` CLI is available and credentials are valid, AWS API responses are the source of truth for account metadata, cluster names, registry URLs, and DNS/load-balancer facts.
   - **Kubernetes**: when `kubectl` is available and a context is configured, live cluster queries are the source of truth for context names, namespaces, ingress domains, and workload state.
   - Each source is authoritative only for the values it owns; a reachable source does not substitute for an unreachable one.
2. **Repository files** — Terraform source, K8s manifests, and CI/CD workflows are the source of truth for *intended configuration*. Use repo data for any surface whose live source is unavailable, or to detect divergence between intent and reality.
3. **Nothing invented** — if a value cannot be determined from either live state or the repo, omit it and note the gap. Never fabricate cloud or cluster state.

**Drift** means generated artifacts (e.g. `harumi.yaml`, architecture docs) must be refreshed to reflect observed reality. For human-authored docs, mismatches must be proposed to the user — never silently applied.

**When a live source is unavailable** — fall back to repo data for that surface, report "live drift could not be verified for [AWS / Kubernetes]", and continue. Missing access is reported per source, not as an all-or-nothing failure.

## Targets

| Target | Source of Truth | Classification |
|--------|----------------|----------------|
| `harumi.yaml` / `.devops.yaml` | Terraform files, K8s manifests, cluster state | Generated |
| `docs/architecture/clusters.md` | Live cluster state across all contexts | Generated |
| `docs/architecture/services.md` | ArgoCD apps, Helm releases, deployments | Generated |
| `docs/architecture/infrastructure.md` | Terraform state, modules, outputs | Generated |
| `docs/architecture/networking.md` | Ingresses, services, domains | Generated |
| `docs/architecture/observability.md` | Monitoring stack state in monitoring namespace | Generated |
| `README.md` | All of the above | Human-authored |
| `CLAUDE.md` | Repo structure, commands, conventions | Human-authored |
| `AGENTS.md` | Infrastructure modules, operational context | Human-authored |
| `docs/runbooks/*` | Operational state changes | Human-authored |

### Config File Resolution

When syncing the repo config target, apply this resolution order:

1. **Both `harumi.yaml` and `.devops.yaml` exist** — treat `harumi.yaml` as canonical; sync it. Flag `.devops.yaml` as migration debt in the summary:

   ```
   ⚠ Migration debt: both `harumi.yaml` and `.devops.yaml` are present.
   `.devops.yaml` is a legacy alias and should be removed:
     rm .devops.yaml
   ```
2. **Only `harumi.yaml` exists** — sync it normally.
3. **Only `.devops.yaml` exists** — sync against it as the active config. After syncing, append a migration notice to the summary:

   ```
   ⚠ Migration recommended: this repo uses `.devops.yaml` (legacy name).
   Rename it to `harumi.yaml` so the plugin hook loads it as the canonical config:
     mv .devops.yaml harumi.yaml
   ```

4. **Neither file exists** — emit a BLOCKING warning; the plugin cannot give accurate project-specific guidance without a repo config.

## Workflow

Follow these steps in order.

### Step 1: Scan Repos

Read the codebase to understand current state:

- **Terraform**: Read all `.tf` files. Extract modules, resources, outputs, variables. Note state backend paths.
- **K8s manifests**: Read manifests in the harumi-k8s repo (if accessible). Extract namespaces, deployments, services, ingresses, ArgoCD Applications.
- **CI/CD**: Read `.github/workflows/*.yml`. Extract pipeline structure, deployment targets.
- **Config**: Detect the active config file: prefer `harumi.yaml` if it exists; fall back to `.devops.yaml`. Read whichever is present and note any values that may need updating.

### Step 2: Query Live State (Read-Only)

Query each live source independently. A failure in one does not block the other.

#### Step 2a: Query Live AWS State

Run these commands when the `aws` CLI is present and credentials are available:

```bash
# Account identity — confirms account ID (not region; use aws configure get region for that)
aws sts get-caller-identity

# EKS clusters — live cluster names, endpoints, and Kubernetes versions
aws eks list-clusters --region <region>
aws eks describe-cluster --name <cluster-name> --region <region> \
  --query 'cluster.{name:name,endpoint:endpoint,version:version,status:status}'

# ECR registries — live registry URLs and repository names
aws ecr describe-repositories --region <region> \
  --query 'repositories[*].{name:repositoryName,uri:repositoryUri}'

# Route53 hosted zones — live domain names attached to this account
aws route53 list-hosted-zones --query 'HostedZones[*].{name:Name,id:Id}'

# ELB/ALB — load balancer DNS names (useful for domain and ingress verification)
aws elbv2 describe-load-balancers --region <region> \
  --query 'LoadBalancers[*].{name:LoadBalancerName,dns:DNSName,state:State.Code}'
```

**If the `aws` CLI is unavailable or credentials are missing:**
- Log: "Live AWS access unavailable — live drift could not be verified for AWS. Falling back to repo data for AWS-sourced fields (account ID, EKS cluster names, ECR registries, domains)."
- Continue — do NOT fail or stop.
- Do NOT invent AWS resource attributes.

#### Step 2b: Query Live Kubernetes State

Use locally configured kubectl contexts for read-only access. Run these commands for each cluster context defined in the active repo config:

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
- Log: "Live Kubernetes access unavailable — live drift could not be verified for Kubernetes. Falling back to repo data for Kubernetes-sourced fields (cluster contexts, ingress domains, workload state)."
- Continue — do NOT fail or stop.
- Do NOT invent cluster contexts, domain names, or workload state.

**NEVER run write commands.** Allowed: `get`, `describe`, `logs`, `top`. Forbidden: `apply`, `delete`, `edit`, `patch`, `create`.

### Step 3: Update Generated Docs

For each generated target:

1. Build the new content from scanned + live data
2. If the target file exists, diff against current content
3. If changes detected, write the file silently (no user prompt)
4. If no changes, skip

**`harumi.yaml` is a generated projection.** It must be rewritten whenever any of the following drift from the checked-in file:
- Terraform outputs (e.g. cluster endpoint, state bucket)
- Live AWS resources: account ID, region, EKS cluster names, ECR registry URIs (from Step 2a)
- Live Kubernetes cluster contexts or ingress domains (from Step 2b)
- ArgoCD gitops repo or app-of-apps paths

When regenerating `harumi.yaml`, use the best available reading per field:
- Fields sourced from AWS: use live AWS data if Step 2a succeeded; otherwise use repo data and note "AWS: repo-only".
- Fields sourced from Kubernetes: use live cluster data if Step 2b succeeded; otherwise use repo data and note "Kubernetes: repo-only".
- Record which source was used for each surface in the Step 5 summary.

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
4. For each stale section, present the drift-review decision block:

```
[filename] line [N]: stale claim detected

  Stale claim:  "[current text in the doc]"
  Live fact:    "[observed value from AWS or Kubernetes]"   ← use when live access succeeded for this source
  Repo fact:    "[value from Terraform or manifests]"       ← use when that source's live access is unavailable
  Proposed edit: [show the diff]

Apply? (yes / skip)
```

5. Wait for user response before proceeding to the next proposed edit
6. If user says "skip", move to the next edit without changing the file

**Never change a human-authored file without showing this block and receiving explicit approval.**

### Step 5: Summary

After all targets are processed, print a summary:

```
Sync complete.

Updated: [list of generated files that changed]
Unchanged: [list of generated files with no drift]
Proposed: [N] edits to [human-authored files] ([M] applied, [K] skipped)
Skipped: [human-authored files with no drift detected]
Live access: AWS=[available/unavailable], Kubernetes=[available/unavailable]
harumi.yaml source: AWS=[live/repo-only], Kubernetes=[live/repo-only]
```

When a source is unavailable, append: "live drift could not be verified for [AWS / Kubernetes]".

## What NOT to Do

- Do not run any write commands against clusters (`kubectl apply`, `helm install`, `argocd app sync`)
- Do not modify human-authored docs without showing the diff and getting explicit approval
- Do not fail if cluster access is unavailable — degrade gracefully
- Do not invent information — if a value cannot be determined from code or cluster, omit it and note the gap
