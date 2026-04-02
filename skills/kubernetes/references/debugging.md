# Kubernetes Debugging Guide

## Diagnostic Sequence

For any failing pod, run these commands in order:

```bash
# 1. Pod status overview
kubectl get pods -n <namespace> --context <context> | grep <pod-name>

# 2. Pod events
kubectl describe pod <pod-name> -n <namespace> --context <context>

# 3. Current logs
kubectl logs <pod-name> -n <namespace> --context <context> --tail=100

# 4. Previous container logs (if restarting)
kubectl logs <pod-name> -n <namespace> --context <context> --previous --tail=100

# 5. Node conditions (if pod is Pending)
kubectl describe node <node-name> --context <context>
```

## Decision Tree by Pod Phase

### Pending

Pod cannot be scheduled. Check in order:

1. **Insufficient resources**
   ```bash
   kubectl describe pod <pod-name> -n <namespace> --context <context> | grep -A5 "Events"
   kubectl top nodes --context <context>
   ```
   Look for: `Insufficient cpu`, `Insufficient memory`
   Fix: Adjust resource requests, add nodes, or use Cluster Autoscaler

2. **Node affinity/taints**
   ```bash
   kubectl get nodes --show-labels --context <context>
   kubectl describe node <node-name> --context <context> | grep -A5 "Taints"
   ```
   Fix: Adjust nodeSelector, affinity rules, or add tolerations

3. **PVC binding**
   ```bash
   kubectl get pvc -n <namespace> --context <context>
   kubectl describe pvc <pvc-name> -n <namespace> --context <context>
   ```
   Fix: Check StorageClass, available PVs, or zone constraints

### CrashLoopBackOff

Container starts but crashes repeatedly.

1. **Check logs**
   ```bash
   kubectl logs <pod-name> -n <namespace> --context <context> --previous --tail=200
   ```

2. **Check exit code**
   ```bash
   kubectl get pod <pod-name> -n <namespace> --context <context> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
   ```
   - Exit 1: Application error — check logs
   - Exit 137: OOMKilled — increase memory limit
   - Exit 143: SIGTERM — check preStop hooks
   - Exit 0 with restart: Check if command is wrong (one-shot vs long-running)

3. **Check resource limits**
   ```bash
   kubectl top pod <pod-name> -n <namespace> --context <context>
   kubectl describe pod <pod-name> -n <namespace> --context <context> | grep -A5 "Limits"
   ```

### ImagePullBackOff

Cannot pull container image.

1. **Check image name and tag**
   ```bash
   kubectl describe pod <pod-name> -n <namespace> --context <context> | grep "Image:"
   ```

2. **Check registry authentication**
   ```bash
   kubectl get secrets -n <namespace> --context <context> | grep regcred
   ```

3. **For ECR — check token expiry**
   ECR tokens expire every 12 hours. Node must have IAM permissions to pull.
   ```bash
   kubectl describe node <node-name> --context <context> | grep "iam"
   ```

### OOMKilled

Container exceeded memory limit.

```bash
# Check actual usage vs limits
kubectl top pod <pod-name> -n <namespace> --context <context>
kubectl describe pod <pod-name> -n <namespace> --context <context> | grep -A3 "Limits"

# Check if OOM events exist
kubectl get events -n <namespace> --context <context> --field-selector reason=OOMKilling
```

Fix: Increase memory limit, or investigate memory leak in application.

### Evicted

Node under disk or memory pressure.

```bash
kubectl describe node <node-name> --context <context> | grep -A10 "Conditions"
kubectl get events --context <context> --field-selector reason=Evicted
```

Fix: Clean up disk, adjust eviction thresholds, or add nodes.

### Terminating (stuck)

Pod won't terminate.

```bash
# Check for finalizers
kubectl get pod <pod-name> -n <namespace> --context <context> -o jsonpath='{.metadata.finalizers}'

# Check if node is healthy
kubectl get node <node-name> --context <context>
```

If stuck due to finalizers, provide handoff to user:
```
kubectl patch pod <pod-name> -n <namespace> --context <context> -p '{"metadata":{"finalizers":null}}' --type merge
```

Only suggest force delete as last resort:
```
kubectl delete pod <pod-name> -n <namespace> --context <context> --force --grace-period=0
```
