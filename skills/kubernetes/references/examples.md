# Kubernetes YAML Examples

Config-driven examples using `harumi.yaml` values. Replace placeholders with actual config values.

## Placeholder Reference

| Placeholder | Source in `harumi.yaml` |
|-------------|------------------------|
| `<app-namespace>` | The application namespace name (e.g., derived from `naming.namespace` and app name) |
| `<stage>` | `naming.stage` |
| `<naming-pattern>` | `naming.pattern` |
| `<context>` | `kubernetes.clusters[].context` |
| `<domain>` | `kubernetes.clusters[].domain` |
| `<registry>` | `kubernetes.clusters[].registry` |

## Complete Application Stack

A minimal but production-ready app deployment:

### Namespace + NetworkPolicy

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <app-namespace>
  labels:
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: <app-namespace>
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### Deployment + Service + Ingress

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
  namespace: <app-namespace>
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: <app-name>
  template:
    metadata:
      labels:
        app.kubernetes.io/name: <app-name>
    spec:
      serviceAccountName: <app-name>
      containers:
        - name: <app-name>
          image: <registry>/<app-name>:<tag>
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: <app-name>
  namespace: <app-namespace>
spec:
  selector:
    app.kubernetes.io/name: <app-name>
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app-name>
  namespace: <app-namespace>
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: <acm-cert-arn>
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
    - host: <app-name>.<domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <app-name>
                port:
                  number: 80
```

## ResourceQuota for Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
  namespace: <app-namespace>
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
```

## LimitRange for Namespace

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: <app-namespace>
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      type: Container
```

## Verification Commands

After applying any manifest, verify with:

```bash
# Resource created/updated
kubectl get <resource> -n <namespace> --context <context>

# No error events
kubectl get events -n <namespace> --sort-by='.lastTimestamp' --context <context> | head -10

# Pods running (for workloads)
kubectl get pods -n <namespace> --context <context>

# Endpoints populated (for services)
kubectl get endpoints -n <namespace> --context <context>
```
