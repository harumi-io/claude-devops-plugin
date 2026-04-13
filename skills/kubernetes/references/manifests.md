# Manifest Authoring Patterns

Reference for writing Kubernetes manifests. Use `harumi.yaml` naming patterns for all resources.

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
  namespace: <namespace>
  labels:
    app.kubernetes.io/name: <app-name>
    app.kubernetes.io/part-of: <project>
    app.kubernetes.io/managed-by: <tool>
spec:
  replicas: <count>
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: <app-name>
  template:
    metadata:
      labels:
        app.kubernetes.io/name: <app-name>
    spec:
      containers:
        - name: <app-name>
          image: <registry>/<image>:<tag>
          ports:
            - containerPort: <port>
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
              port: <port>
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: <port>
            initialDelaySeconds: 5
            periodSeconds: 5
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: <app-name>
```

## Service

```yaml
# ClusterIP (default — internal traffic)
apiVersion: v1
kind: Service
metadata:
  name: <app-name>
  namespace: <namespace>
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: <app-name>
  ports:
    - port: 80
      targetPort: <port>
      protocol: TCP
```

## Ingress (ALB)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app-name>
  namespace: <namespace>
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <acm-cert-arn>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
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

Use `<domain>` from `harumi.yaml` `kubernetes.clusters[].domain`.

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <app-name>-config
  namespace: <namespace>
data:
  KEY: "value"
```

## PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <app-name>-data
  namespace: <namespace>
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 10Gi
```

## HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <app-name>
  namespace: <namespace>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <app-name>
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: <app-name>
  namespace: <namespace>
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: <app-name>
```

## Helm Values Best Practices

- Override only what differs from chart defaults — keep values files minimal
- Use `--set` for single CI-driven values (like image.tag), values files for everything else
- Pin chart versions in ArgoCD Applications or Helmfile
- Document non-obvious overrides with YAML comments
- Separate environment-specific values from shared values when managing multiple clusters
