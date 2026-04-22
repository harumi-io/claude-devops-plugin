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

1. **Both `harumi.yaml` and `.devops.yaml` exist** — treat `harumi.yaml` as canonical; sync it. Ignore `.devops.yaml` as a sync target (it is a legacy alias).
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

### Step 2: Query Live Cluster State (Read-Only)

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
