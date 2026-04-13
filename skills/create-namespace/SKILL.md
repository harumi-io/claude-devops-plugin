---
name: create-namespace
description: "Create a Kubernetes namespace with RBAC, quotas, network policies, and optional ArgoCD registration. Use when: user wants to create a new namespace."
---

# Create Namespace

Create a namespace with associated security and resource controls.

## Inputs

Ask for these if not provided:

1. **Namespace name** (e.g., "my-app")
2. **Target cluster** — from `harumi.yaml` `kubernetes.clusters[]` list
3. **Resource quotas** — ask if custom limits are needed, otherwise use defaults

## Execution Steps

### Step 1: Check if namespace exists

```bash
kubectl get namespace <namespace-name> --context <context>
kubectl get namespaces --context <context>
```

If namespace already exists, report it and ask the user how to proceed.

### Step 2: Generate namespace resources

Create the following manifests:

1. **Namespace** with pod security labels
2. **ResourceQuota** with sensible defaults (4 CPU, 8Gi memory, 20 pods)
3. **LimitRange** with default container limits
4. **NetworkPolicy** — default deny ingress
5. **RBAC RoleBinding** — if the user specifies who should have access

Use patterns from the `kubernetes` skill's `references/security.md` and `references/examples.md`.

### Step 3: Register in ArgoCD (if configured)

If `harumi.yaml` has `kubernetes.gitops: argocd`:
- Create an ArgoCD Application manifest for the namespace resources
- Place it in the gitops repo's `<cluster-dir>/argocd/` directory
- Use patterns from the `argocd` skill's `references/examples.md`

### Step 4: Hand off

```
Namespace '<namespace-name>' resources ready!

Execute:
kubectl apply -f <namespace-manifest-path> --context <context>

What this will do:
- Create namespace <namespace-name>
- Set ResourceQuota (CPU: 4, Memory: 8Gi, Pods: 20)
- Set LimitRange (default: 500m CPU, 512Mi memory)
- Apply default-deny NetworkPolicy
- [RBAC if configured]

Verification:
kubectl get namespace <namespace-name> --context <context>
kubectl get resourcequota -n <namespace-name> --context <context>
kubectl get networkpolicy -n <namespace-name> --context <context>
```
