# Kubernetes Workflow and Handoff

Detailed workflow phases for Kubernetes changes. Read this when following the full change workflow.

## Phase 1: Consult Config

Read `.devops.yaml` to determine:
- Which clusters are available and their contexts
- The naming pattern for resources
- The gitops tool (argocd or none)
- The gitops repo (where manifests should live)

## Phase 2: Verify Current State

Always verify current state before changes. Never assume.

```bash
# Cluster health
kubectl cluster-info --context <context>
kubectl get nodes --context <context>

# Namespace resources
kubectl get all -n <namespace> --context <context>

# Specific resource
kubectl describe <resource-type> <name> -n <namespace> --context <context>

# Events (recent issues)
kubectl get events -n <namespace> --sort-by='.lastTimestamp' --context <context>

# Resource usage
kubectl top pods -n <namespace> --context <context>
kubectl top nodes --context <context>
```

## Phase 3: Implement

Write manifests following the project's naming conventions from `.devops.yaml`.

Key checks before writing:
- Does the namespace exist?
- Are there existing resources with similar names?
- Does the resource conflict with existing HPA, PDB, or NetworkPolicy?

## Phase 4: Validate

```bash
# Dry run against the cluster
kubectl apply --dry-run=client -f <file> --context <context>

# Server-side dry run (more thorough)
kubectl apply --dry-run=server -f <file> --context <context>

# Diff against current state
kubectl diff -f <file> --context <context>
```

## Phase 5: Handoff

Template for all mutations:

```
Configuration ready for deployment!

Please review the manifest(s), then execute:
kubectl apply -f [file] --context [context] -n [namespace]

What this will do:
- [Create/Modify/Delete] [resource count] [resource types]
- [Specific changes summary]

Verification:
kubectl get [resource] -n [namespace] --context [context]
kubectl describe [resource] [name] -n [namespace] --context [context]
kubectl rollout status deployment/[name] -n [namespace] --context [context]  # if Deployment
```

## Phase 6: Post-Apply Verification

After user confirms apply, verify:

```bash
# Check resource exists and is healthy
kubectl get <resource> <name> -n <namespace> --context <context>

# For Deployments — check rollout
kubectl rollout status deployment/<name> -n <namespace> --context <context>

# For Services — check endpoints
kubectl get endpoints <name> -n <namespace> --context <context>

# For Ingress — check address
kubectl get ingress <name> -n <namespace> --context <context>

# Check events for errors
kubectl get events -n <namespace> --sort-by='.lastTimestamp' --context <context> | head -20
```
