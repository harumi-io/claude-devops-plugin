---
name: scale-deployment
description: "Scale a Kubernetes deployment's replicas with safety checks for HPA conflicts and node capacity. Use when: user wants to scale up, scale down, or change replica count."
---

# Scale Deployment

Scale a deployment's replica count with safety checks.

## Inputs

Ask for these if not provided:

1. **Deployment name** (e.g., "my-api")
2. **Namespace**
3. **Target cluster** — from `harumi.yaml` `kubernetes.clusters[]` list
4. **Target replica count**

## Execution Steps

### Step 1: Inspect current state

```bash
# Current deployment state
kubectl get deployment <deployment-name> -n <namespace> --context <context>

# Check if HPA exists
kubectl get hpa -n <namespace> --context <context> | grep <deployment-name>

# Current pod distribution
kubectl get pods -n <namespace> --context <context> -l app.kubernetes.io/name=<deployment-name> -o wide

# Available node capacity
kubectl top nodes --context <context>
```

### Step 2: Check for HPA conflict

If an HPA is configured for this deployment:

```bash
kubectl describe hpa <hpa-name> -n <namespace> --context <context>
```

**Warn the user:**
```
⚠ An HPA is managing this deployment (min: X, max: Y, current: Z).
Manual scaling will be overridden by the HPA on the next evaluation cycle.

To change the scale permanently, modify the HPA instead:
kubectl patch hpa <hpa-name> -n <namespace> --context <context> -p '{"spec":{"minReplicas":<new-min>,"maxReplicas":<new-max>}}'

To scale temporarily (will revert), use the handoff below.
```

Ask the user how to proceed.

### Step 3: Check capacity (if scaling up)

If scaling up, verify node capacity:

```bash
kubectl top nodes --context <context>
kubectl describe nodes --context <context> | grep -A5 "Allocated resources"
```

If insufficient capacity, warn the user and suggest enabling Cluster Autoscaler or adding nodes first.

### Step 4: Hand off

```
Scale ready!

Execute:
kubectl scale deployment/<deployment-name> -n <namespace> --context <context> --replicas=<target>

What this will do:
- Change replicas from <current> to <target>
- [HPA warning if applicable]

Verification:
kubectl get deployment <deployment-name> -n <namespace> --context <context>
kubectl get pods -n <namespace> --context <context> -l app.kubernetes.io/name=<deployment-name>
kubectl rollout status deployment/<deployment-name> -n <namespace> --context <context>
```
