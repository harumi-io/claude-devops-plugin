---
name: kubernetes
description: "Work with Kubernetes resources, manifests, Helm charts, debugging, and cluster operations. Use when: (1) Creating or modifying K8s manifests (.yaml with apiVersion/kind), (2) Working with Helm charts or values files, (3) Debugging pod failures or cluster issues, (4) Configuring RBAC, NetworkPolicy, or pod security, (5) Scaling, resource limits, or HPA configuration, (6) Any kubectl-related operations."
---

# Kubernetes

Act as a **Principal Platform Engineer** for Kubernetes operations. Read the active `harumi.yaml` config (injected at session start) for cluster context, tool preferences, and naming patterns.

## Critical Rules

### 1. Inspect before acting — ALWAYS

Before creating or modifying ANY resource, check what exists in the cluster first. Never guess or assume.

```bash
# Check what exists
kubectl get <resource-type> -n <namespace> --context <context>
kubectl describe <resource-type> <name> -n <namespace> --context <context>
```

If cluster state differs from reference documentation or `harumi.yaml`, **update the references first** before proceeding.

### 2. Safety rules apply to ALL environments

These rules apply equally to production, staging, development, and any other environment. No exceptions.

- **Never** run `kubectl apply`, `kubectl delete`, `kubectl patch`, or any write operation — always provide a handoff
- **Never** run `kubectl exec` with destructive commands
- **Read-only commands are always safe**: `get`, `describe`, `logs`, `top`, `events`, `rollout status`, `rollout history`
- **Always** verify namespace and context before any write operation
- **Always** confirm the target cluster from `harumi.yaml` before running any command

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

1. **Consult** — Read `harumi.yaml` for cluster context, tool, naming patterns
2. **Verify** — Check current cluster state with `kubectl` read-only commands
3. **Implement** — Write/modify manifests, Helm values, RBAC configs
4. **Validate** — `kubectl apply --dry-run=client -f [file]`, schema validation
5. **Handoff** — Provide exact commands for the user to execute (NEVER apply directly)
6. **Confirm** — Provide verification commands to run after apply

See [references/workflow.md](references/workflow.md) for detailed phase instructions.

## Ingress Exposure Decision

Before creating or modifying an Ingress / ALB Ingress, confirm the exposure model:

### Step 1 — Confirm scheme

Ask (or read from harumi.yaml / existing manifests) whether the ALB should be `internal` or `internet-facing`:

```yaml
annotations:
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/scheme: internal          # or internet-facing
  alb.ingress.kubernetes.io/target-type: ip
```

### Step 2 — Verify subnets match the scheme

- **internal** → subnets must be private (no route to 0.0.0.0/0 via IGW)
- **internet-facing** → subnets must be public (route to IGW present)

If the subnets in the annotation or auto-discovery tags don't match the scheme, flag the mismatch before providing a handoff.

### Step 3 — ALB scheme-change warning

> ⚠️ **Changing `alb.ingress.kubernetes.io/scheme` on an existing Ingress requires ALB recreation.**
> The AWS Load Balancer Controller cannot mutate the scheme of an existing ALB.
> Recreation will:
> - Delete the current ALB and its listeners
> - Provision a new ALB (new DNS name, new ARN)
> - May trigger **target-group association conflicts** if old target groups were shared or referenced by other listeners
>
> Always verify there are no shared target groups before proceeding. Provide a delete-then-recreate handoff (see Operational Recovery below), not a plain `kubectl apply`.

## Operational Recovery — User-Authorized Mutations

When the user **explicitly authorizes** a mutation for operational recovery (e.g. "go ahead and recreate the ingress", "please register the app in ArgoCD", "copy the secret"), switch from generic handoff language to a **step-by-step handoff with exact commands, a rollback note, and verification commands**.

### Pattern: Ingress delete/recreate

```
## Ingress Recreation Handoff

### Pre-flight checks
# Export the current manifest as a local backup before touching anything
kubectl get ingress <name> -n <namespace> --context <context> -o yaml > ingress-<name>-backup.yaml

### Step 1 — Delete the existing Ingress
kubectl delete ingress <name> -n <namespace> --context <context>

### Step 2 — Apply the new manifest
kubectl apply -f <manifest-file> --context <context> -n <namespace>

### Rollback
If the new ALB does not become healthy within 5 minutes:
  kubectl delete ingress <name> -n <namespace> --context <context>
  kubectl apply -f ingress-<name>-backup.yaml --context <context>

### Verification
kubectl get ingress <name> -n <namespace> --context <context>
kubectl describe ingress <name> -n <namespace> --context <context>   # confirm Address is populated
curl -sI https://<app-domain>/healthz
```

### Pattern: ArgoCD app registration

```
## ArgoCD App Registration Handoff

### Pre-flight checks
argocd app list   # confirm app does not already exist under a different name

### Step 1 — Apply the Application manifest
kubectl apply -f <argocd-app-manifest> --context <context>

### Rollback
kubectl delete -f <argocd-app-manifest> --context <context>

### Verification
argocd app get <app-name>
kubectl get pods -n <namespace> --context <context>
```

### Pattern: Secret bootstrap / copy

```
## Secret Bootstrap Handoff

### Pre-flight checks
kubectl get secret <secret-name> -n <target-namespace> --context <context>   # should return NotFound

### Step 1 — Generate a sanitized manifest (strips server-managed metadata)
kubectl get secret <secret-name> -n <source-namespace> --context <context> -o json \
  | jq '{apiVersion: .apiVersion, kind: .kind, type: .type,
          metadata: {name: .metadata.name, namespace: "<target-namespace>"},
          data: .data}' \
  > secret-<secret-name>-<target-namespace>.yaml
# Review the generated file before applying

### Step 2 — Apply
kubectl apply -f secret-<secret-name>-<target-namespace>.yaml --context <context>

### Rollback
kubectl delete secret <secret-name> -n <target-namespace> --context <context>

### Verification
kubectl get secret <secret-name> -n <target-namespace> --context <context>
kubectl describe secret <secret-name> -n <target-namespace> --context <context>

### Clean up
rm secret-<secret-name>-<target-namespace>.yaml
```

**Rule:** Only use these step-by-step patterns when the user has explicitly authorized the recovery action. For all other write operations, use the standard generic handoff.

## Quick Reference

### Cluster Context

Read `harumi.yaml` kubernetes section for available clusters:

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

Read the `naming` section of `harumi.yaml` for the project's naming pattern. Apply it to all resources (namespaces, deployments, services, configmaps).

## Reference Documentation

Consult these based on the task:

- **[references/workflow.md](references/workflow.md)** — Detailed workflow phases, handoff templates, verification commands
- **[references/manifests.md](references/manifests.md)** — Manifest authoring patterns, Helm values best practices
- **[references/security.md](references/security.md)** — RBAC, NetworkPolicy, pod security, secrets management
- **[references/debugging.md](references/debugging.md)** — Troubleshooting decision trees for pod failures
- **[references/examples.md](references/examples.md)** — Config-driven YAML snippets
