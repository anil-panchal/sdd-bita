# Kubernetes Security Reference

## Overview

When reviewing Kubernetes manifests, focus on workload privilege, RBAC scope, secret exposure, network isolation, and trust boundaries between namespaces, service accounts, and nodes.

---

## Pod Security and Workload Hardening

### Safe Patterns (Do Not Flag)

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: registry.example.com/app@sha256:abc123
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
              add: ["NET_BIND_SERVICE"]
```

### Flag These

```yaml
spec:
  hostNetwork: true          # FLAG
  hostPID: true              # FLAG
  hostIPC: true              # FLAG
  containers:
    - name: app
      securityContext:
        privileged: true     # FLAG
        allowPrivilegeEscalation: true  # FLAG
        runAsUser: 0         # FLAG
        seccompProfile:
          type: Unconfined   # FLAG
        capabilities:
          add: ["SYS_ADMIN"] # FLAG
  volumes:
    - name: host
      hostPath:              # FLAG
          path: /
```

### Review Notes

- Kubernetes Pod Security Standards treat `Restricted` as the hardened baseline for most application workloads.
- `hostPath`, privileged containers, host namespaces, `Unconfined` seccomp, and root execution are strong signals for high-risk workloads.

---

## Service Accounts and RBAC

### Safe Patterns (Do Not Flag)

```yaml
kind: Role
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]

---

kind: RoleBinding
roleRef:
  kind: Role
  name: app-reader
subjects:
  - kind: ServiceAccount
    name: app
```

### Flag These

```yaml
kind: ClusterRole
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]   # FLAG

---

kind: ClusterRoleBinding
roleRef:
  kind: ClusterRole
  name: cluster-admin  # FLAG
```

### Check Context

```yaml
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["list", "watch"]   # FLAG in most app contexts

  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create"]          # CHECK: workload creation implies access to mounted secrets and service accounts

  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]          # CHECK
```

### Review Notes

- Prefer namespace-scoped `RoleBinding` over `ClusterRoleBinding`.
- Watch for privilege-escalation verbs such as `bind`, `escalate`, and `impersonate`.
- `list` or `watch` on Secrets effectively grants access to secret contents.

---

## Secrets Management

### Flag These

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
stringData:
  password: super-secret-password   # FLAG if committed to repo
```

```yaml
apiVersion: v1
kind: Secret
data:
  password: c3VwZXItc2VjcmV0  # FLAG if committed to repo; base64 is not encryption
```

### Safer Pattern

```yaml
spec:
  volumes:
    - name: app-secrets
      csi:
        driver: secrets-store.csi.k8s.io
```

### Review Notes

- Committed Secret manifests are high-confidence secret exposure findings.
- Base64 encoding does not provide confidentiality.
- Restrict Secret access to the specific container or workload that actually needs it.

---

## Network Isolation

### Safe Patterns (Do Not Flag)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
```

### Check Context

```yaml
kind: Service
spec:
  type: LoadBalancer  # CHECK
```

```yaml
kind: NetworkPolicy
spec:
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0   # CHECK
```

### Review Notes

- Missing or weak `NetworkPolicy` is usually a hardening gap rather than a direct exploit, but broad egress rules can increase SSRF blast radius.
- Public load balancers and ingress rules should be reviewed together with authentication, TLS, and application exposure.

---

## Admission and Namespace Controls

### Safe Patterns (Do Not Flag)

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
```

### Check Context

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: privileged   # CHECK
```

### Review Notes

- Namespace labels that weaken Pod Security Admission can materially change cluster trust boundaries.
- Review who can patch namespaces, webhook configurations, or policy resources.

---

## Images and Supply Chain

### Check Context

```yaml
containers:
  - image: nginx:latest   # CHECK
  - image: myrepo/app:v1  # CHECK
```

### Review Notes

- Mutable tags reduce traceability and can undermine repeatable deployments.
- Prefer immutable digests for production workloads when possible.

---

## rg Patterns

```bash
# Dangerous pod settings
rg -n 'privileged:\s*true|allowPrivilegeEscalation:\s*true|host(Network|PID|IPC):\s*true|runAsUser:\s*0|hostPath:' -g '*.{yaml,yml}'

# Weak securityContext
rg -n 'seccompProfile:\s*\n\s*type:\s*Unconfined|capabilities:\s*\n(?:.*\n)*\s*add:' -g '*.{yaml,yml}'

# RBAC wildcards and privileged bindings
rg -n 'cluster-admin|system:masters|verbs:\s*\["\*"\]|resources:\s*\["\*"\]|apiGroups:\s*\["\*"\]' -g '*.{yaml,yml}'

# Secrets in manifests
rg -n '^kind:\s*Secret|stringData:|data:' -g '*.{yaml,yml}'

# Service account token mounting
rg -n 'automountServiceAccountToken:\s*(true|false)|serviceAccount(Name)?:' -g '*.{yaml,yml}'

# Network exposure
rg -n 'type:\s*LoadBalancer|NodePort|cidr:\s*0\.0\.0\.0/0' -g '*.{yaml,yml}'
```

---

## References

- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Kubernetes Secrets Good Practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
