# Kubernetes Security Patterns

## RBAC

### Namespace-scoped Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: <role-name>
  namespace: <namespace>
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <binding-name>
  namespace: <namespace>
subjects:
  - kind: ServiceAccount
    name: <sa-name>
    namespace: <namespace>
roleRef:
  kind: Role
  name: <role-name>
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole (for cross-namespace access)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <role-name>
rules:
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch"]
```

## NetworkPolicy

### Default deny all ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### Allow specific ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-<source>-to-<target>
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: <target-app>
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: <source-namespace>
          podSelector:
            matchLabels:
              app.kubernetes.io/name: <source-app>
      ports:
        - port: <port>
          protocol: TCP
```

### Egress control

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: <app-name>
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
          protocol: TCP
```

## Pod Security Standards

### Restricted (recommended for workloads)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Baseline (for system components that need more privileges)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Secret Management

### External Secrets Operator (preferred)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: <app-name>-secrets
  namespace: <namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: <app-name>-secrets
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: <secrets-path>/<app-name>
        property: database_url
```

### ServiceAccount Token

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <app-name>
  namespace: <namespace>
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<role-name>
automountServiceAccountToken: false
```

Only mount tokens when the pod needs Kubernetes API access. Use IRSA (IAM Roles for Service Accounts) for AWS access.
