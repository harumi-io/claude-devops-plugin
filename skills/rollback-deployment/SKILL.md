---
name: rollback-deployment
description: "Roll back a Kubernetes deployment to a previous revision with safety checks. Use when: user wants to rollback, revert, or undo a deployment."
---

# Rollback Deployment

Roll back a deployment to a previous revision.

## Inputs

Ask for these if not provided:

1. **Deployment name** (e.g., "my-api")
2. **Namespace**
3. **Target cluster** — from the active repo config `kubernetes.clusters[]` list
4. **Target revision** — optional, defaults to previous revision

## Execution Steps

### Step 1: Inspect current state

```bash
# Current rollout status
kubectl rollout status deployment/<deployment-name> -n <namespace> --context <context>

# Current replica set
kubectl get replicaset -n <namespace> --context <context> -l app.kubernetes.io/name=<deployment-name>

# Revision history
kubectl rollout history deployment/<deployment-name> -n <namespace> --context <context>
```

### Step 2: Show revision details

If the user hasn't specified a target revision, show the last 5 revisions with their details:

```bash
kubectl rollout history deployment/<deployment-name> -n <namespace> --context <context> --revision=<N>
```

Ask the user which revision to roll back to.

### Step 3: Show what will change

```bash
# Compare current vs target revision
kubectl rollout history deployment/<deployment-name> -n <namespace> --context <context> --revision=<current>
kubectl rollout history deployment/<deployment-name> -n <namespace> --context <context> --revision=<target>
```

Present a clear diff of what changes (image tag, env vars, resource limits, etc.).

### Step 4: Hand off

```
Rollback ready!

Execute:
kubectl rollout undo deployment/<deployment-name> -n <namespace> --context <context> --to-revision=<target>

What this will do:
- Roll back <deployment-name> from revision <current> to revision <target>
- [Specific changes: image tag, env vars, etc.]

Verification:
kubectl rollout status deployment/<deployment-name> -n <namespace> --context <context>
kubectl get pods -n <namespace> --context <context> -l app.kubernetes.io/name=<deployment-name>
```
