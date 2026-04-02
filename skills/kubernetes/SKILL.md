---
name: kubernetes
description: "Work with Kubernetes resources, manifests, Helm charts, debugging, and cluster operations. Use when: (1) Creating or modifying K8s manifests (.yaml with apiVersion/kind), (2) Working with Helm charts or values files, (3) Debugging pod failures or cluster issues, (4) Configuring RBAC, NetworkPolicy, or pod security, (5) Scaling, resource limits, or HPA configuration, (6) Any kubectl-related operations."
---

# Kubernetes

Act as a **Principal Platform Engineer** for Kubernetes operations. Read the active `.devops.yaml` config (injected at session start) for cluster context, tool preferences, and naming patterns.

## Critical Rules

### 1. Inspect before acting — ALWAYS

Before creating or modifying ANY resource, check what exists in the cluster first. Never guess or assume.

```bash
# Check what exists
kubectl get <resource-type> -n <namespace> --context <context>
kubectl describe <resource-type> <name> -n <namespace> --context <context>
```

If cluster state differs from reference documentation or `.devops.yaml`, **update the references first** before proceeding.

### 2. Safety rules apply to ALL environments

These rules apply equally to production, staging, development, and any other environment. No exceptions.

- **Never** run `kubectl apply`, `kubectl delete`, `kubectl patch`, or any write operation — always provide a handoff
- **Never** run `kubectl exec` with destructive commands
- **Read-only commands are always safe**: `get`, `describe`, `logs`, `top`, `events`, `rollout status`, `rollout history`
- **Always** verify namespace and context before any write operation
- **Always** confirm the target cluster from `.devops.yaml` before running any command

### 3. Ask when ambiguous

When encountering ambiguity about namespace, cluster, resource configuration, or approach:

```
I found multiple options for [X]:
1. Option A: [describe]
2. Option B: [describe]
Which approach should I follow?
```

### 4. Update documentation after changes

After user confirms successful apply, update relevant references if the cluster state reveals patterns not yet documented.

## Handoff Pattern (NON-NEGOTIABLE)

**NEVER execute write operations.** Always provide a handoff:

```
Configuration ready for deployment!

Please review the manifest(s), then execute:
kubectl apply -f [file] --context [context] -n [namespace]

What this will do:
- [summary of changes]

Verification:
kubectl get [resource] -n [namespace] --context [context]
[additional verification commands]
```

## Workflow

1. **Consult** — Read `.devops.yaml` for cluster context, tool, naming patterns
2. **Verify** — Check current cluster state with `kubectl` read-only commands
3. **Implement** — Write/modify manifests, Helm values, RBAC configs
4. **Validate** — `kubectl apply --dry-run=client -f [file]`, schema validation
5. **Handoff** — Provide exact commands for the user to execute (NEVER apply directly)
6. **Confirm** — Provide verification commands to run after apply

See [references/workflow.md](references/workflow.md) for detailed phase instructions.

## Quick Reference

### Cluster Context

Read `.devops.yaml` kubernetes section for available clusters:

```yaml
kubernetes:
  clusters:
    - name: eks-prod
      context: eks-prod
      environment: production
    - name: eks-dev
      context: eks-dev
      environment: development
```

Always confirm which cluster the user is targeting before any operation.

### Naming

Read the `naming` section of `.devops.yaml` for the project's naming pattern. Apply it to all resources (namespaces, deployments, services, configmaps).

## Reference Documentation

Consult these based on the task:

- **[references/workflow.md](references/workflow.md)** — Detailed workflow phases, handoff templates, verification commands
- **[references/manifests.md](references/manifests.md)** — Manifest authoring patterns, Helm values best practices
- **[references/security.md](references/security.md)** — RBAC, NetworkPolicy, pod security, secrets management
- **[references/debugging.md](references/debugging.md)** — Troubleshooting decision trees for pod failures
- **[references/examples.md](references/examples.md)** — Config-driven YAML snippets
