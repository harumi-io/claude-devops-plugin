---
name: debug-pod
description: "Guided troubleshooting for a failing or misbehaving Kubernetes pod. Runs diagnostic sequence and follows decision trees. Use when: user wants to debug, troubleshoot, or investigate a pod issue."
---

# Debug Pod

Guided troubleshooting for a failing pod.

## Inputs

Ask for these if not provided:

1. **Pod name or selector** (e.g., "my-api-7d9f8b6c5-x2k4j" or "app=my-api")
2. **Namespace**
3. **Target cluster** — from the active repo config `kubernetes.clusters[]` list

## Execution Steps

### Step 1: Run diagnostic sequence

Run these commands in order and analyze the output:

```bash
# 1. Pod status
kubectl get pods -n <namespace> --context <context> | grep <pod-name-or-selector>

# 2. Detailed pod description (events, conditions, container status)
kubectl describe pod <pod-name> -n <namespace> --context <context>

# 3. Current container logs
kubectl logs <pod-name> -n <namespace> --context <context> --tail=100

# 4. Previous container logs (if pod is restarting)
kubectl logs <pod-name> -n <namespace> --context <context> --previous --tail=100

# 5. Recent events in namespace
kubectl get events -n <namespace> --sort-by='.lastTimestamp' --context <context> | head -20
```

### Step 2: Follow decision tree

Based on the pod phase/condition, follow the appropriate troubleshooting path from the `kubernetes` skill's `references/debugging.md`:

- **Pending** → check node capacity, affinity/taints, PVC binding
- **CrashLoopBackOff** → check logs, exit codes, resource limits
- **ImagePullBackOff** → check image name, registry auth, ECR tokens
- **OOMKilled** → check memory limits vs actual usage
- **Evicted** → check node conditions, disk pressure
- **Terminating (stuck)** → check finalizers, node health

### Step 3: Check node conditions (if relevant)

If the issue appears node-related:

```bash
kubectl describe node <node-name> --context <context>
kubectl top nodes --context <context>
```

### Step 4: Suggest fixes

Based on findings, suggest specific fixes. If fixes require manifest changes, write the changes and provide a handoff. Never apply fixes directly.
