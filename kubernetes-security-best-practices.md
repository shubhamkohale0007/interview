# Kubernetes Security Best Practices — Complete Guide

> **Reference Video:** [Kubernetes Security Best Practices](https://www.youtube.com/watch?v=oBf5lrmquYI)
> **Last Updated:** June 2026
> **Applies To:** Kubernetes 1.25+, OpenShift 4.x, EKS, AKS, GKE

---

## Table of Contents

1. [Why Kubernetes Security Matters](#why-kubernetes-security-matters)
2. [Attack Surface Overview](#attack-surface-overview)
3. [Defense-in-Depth Architecture](#defense-in-depth-architecture)
4. [Best Practice #1 — Secure Container Image Building](#best-practice-1--secure-container-image-building)
5. [Best Practice #2 — Image Scanning](#best-practice-2--image-scanning)
6. [Best Practice #3 — Avoid Root & Privileged Containers](#best-practice-3--avoid-root--privileged-containers)
7. [Best Practice #4 — RBAC & Least Privilege](#best-practice-4--rbac--least-privilege)
8. [Best Practice #5 — Network Policies](#best-practice-5--network-policies)
9. [Best Practice #6 — Service Mesh & mTLS](#best-practice-6--service-mesh--mtls)
10. [Best Practice #7 — Secrets Management](#best-practice-7--secrets-management)
11. [Best Practice #8 — etcd Security](#best-practice-8--etcd-security)
12. [Best Practice #9 — Backup & Disaster Recovery](#best-practice-9--backup--disaster-recovery)
13. [Best Practice #10 — Policy Enforcement (OPA / Kyverno)](#best-practice-10--policy-enforcement-opa--kyverno)
14. [Best Practice #11 — Automated Disaster Recovery](#best-practice-11--automated-disaster-recovery)
15. [Additional Best Practices](#additional-best-practices)
16. [Security Checklist](#security-checklist)
17. [Tools Reference](#tools-reference)

---

## Why Kubernetes Security Matters

```
Cloud ≠ Secure by Default
Kubernetes ≠ Secure by Default
```

- Default Kubernetes configuration contains multiple exploitable vulnerabilities
- Cloud-native apps are high-value targets — attackers follow where workloads go
- Attackers need **one** weak link; defenders must protect **every** layer
- A compromised pod can escalate to OS-level compromise of the entire node
- Security must be built-in from the start — not bolted on later

> **The Golden Rule:** Redundant, layered defenses. No single control is sufficient.

---

## Attack Surface Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ATTACK SURFACE MAP                               │
│                                                                         │
│   External Attacker                                                     │
│        │                                                                │
│        ▼                                                                │
│   ┌────────────┐    ┌──────────────────────────────────────────────┐    │
│   │  Exposed   │    │              Kubernetes Cluster              │    │
│   │  Services  │───▶│                                              │    │
│   │  :443/:80  │    │  ┌─────────────────────────────────────┐    │    │
│   └────────────┘    │  │         Workload Layer               │    │    │
│                     │  │  ┌──────────┐  ┌──────────────────┐ │    │    │
│   Malicious Image   │  │  │   Pod    │  │  Privileged Pod  │ │    │    │
│   (Supply Chain) ───┼─▶│  │ (App)   │  │  (Root User)     │ │    │    │
│                     │  │  └────┬─────┘  └────────┬─────────┘ │    │    │
│   Stolen Creds  ────┼─▶│       │ Lateral Movement│           │    │    │
│   (API Access)      │  └───────┼─────────────────┼───────────┘    │    │
│                     │          │                 │                 │    │
│   etcd Direct   ────┼─▶│  ┌────▼────────────────▼──────────────┐ │    │
│   Access            │  │  │       Platform Layer                │ │    │
│                     │  │  │  API Server | etcd | kubelet        │ │    │
│   Insider Threat ───┼─▶│  └────────────────────────────────────┘ │    │
│                     │  │                                          │    │
│                     │  │  ┌──────────────────────────────────────┐│    │
│                     │  │  │     Infrastructure Layer             ││    │
│                     │  │  │  OS | Hypervisor | Cloud Provider   ││    │
│                     │  │  └──────────────────────────────────────┘│    │
│                     │  └──────────────────────────────────────────┘    │
│                     └──────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Defense-in-Depth Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LAYERED SECURITY MODEL                               │
│                                                                         │
│   Layer 7: Disaster Recovery & Backup (Kasten K10)                     │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  Layer 6: Policy Enforcement (OPA / Kyverno Admission Control)  │  │
│   │  ┌───────────────────────────────────────────────────────────┐  │  │
│   │  │  Layer 5: Secrets Encryption (Vault / KMS)               │  │  │
│   │  │  ┌─────────────────────────────────────────────────────┐  │  │  │
│   │  │  │  Layer 4: Service Mesh mTLS (Istio / Linkerd)      │  │  │  │
│   │  │  │  ┌───────────────────────────────────────────────┐  │  │  │  │
│   │  │  │  │  Layer 3: Network Policies (Calico / Cilium) │  │  │  │  │
│   │  │  │  │  ┌─────────────────────────────────────────┐  │  │  │  │  │
│   │  │  │  │  │  Layer 2: RBAC & AuthN/AuthZ           │  │  │  │  │  │
│   │  │  │  │  │  ┌───────────────────────────────────┐  │  │  │  │  │  │
│   │  │  │  │  │  │  Layer 1: Secure Images & Runtime│  │  │  │  │  │  │
│   │  │  │  │  │  │  (Non-root, Scan, Minimal base)  │  │  │  │  │  │  │
│   │  │  │  │  │  └───────────────────────────────────┘  │  │  │  │  │  │
│   │  │  │  │  └─────────────────────────────────────────┘  │  │  │  │  │
│   │  │  │  └───────────────────────────────────────────────┘  │  │  │  │
│   │  │  └─────────────────────────────────────────────────────┘  │  │  │
│   │  └───────────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Best Practice #1 — Secure Container Image Building

> **Video Reference:** [05:04] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=304](https://www.youtube.com/watch?v=oBf5lrmquYI&t=304)

### Key Risks

- Malicious or backdoored packages pulled from public registries
- Vulnerabilities in base OS layers (Ubuntu, Debian, Alpine)
- Bloated images with unnecessary tools (curl, bash, package managers) that attackers can abuse

### Image Build Security Model

```
  Source Code
      │
      ▼
  ┌──────────────────────────────────────────┐
  │            CI/CD Pipeline                │
  │                                          │
  │  ┌────────────────────────────────────┐  │
  │  │  Step 1: Dependency Audit          │  │
  │  │  npm audit / pip check / go mod    │  │
  │  └────────────────────────────────────┘  │
  │                    │                     │
  │  ┌─────────────────▼──────────────────┐  │
  │  │  Step 2: Minimal Base Image        │  │
  │  │  FROM scratch / distroless / alpine│  │
  │  └────────────────────────────────────┘  │
  │                    │                     │
  │  ┌─────────────────▼──────────────────┐  │
  │  │  Step 3: Multi-Stage Build         │  │
  │  │  Build stage → Runtime stage only  │  │
  │  └────────────────────────────────────┘  │
  │                    │                     │
  │  ┌─────────────────▼──────────────────┐  │
  │  │  Step 4: Non-Root User             │  │
  │  │  USER 1001 (never root)            │  │
  │  └────────────────────────────────────┘  │
  │                    │                     │
  │  ┌─────────────────▼──────────────────┐  │
  │  │  Step 5: Read-Only Filesystem      │  │
  │  │  No writeable layers at runtime    │  │
  │  └────────────────────────────────────┘  │
  └──────────────────────────────────────────┘
```

### Secure Dockerfile Example

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

# Stage 2: Minimal runtime image
FROM gcr.io/distroless/static:nonroot    # No shell, no package manager, no root
WORKDIR /app
COPY --from=builder /app/myapp .
USER nonroot:nonroot                      # Never run as root
EXPOSE 8080
ENTRYPOINT ["/app/myapp"]
```

### Base Image Comparison

| Base Image         | Size    | Shell | Package Manager | Attack Surface |
|--------------------|---------|-------|-----------------|----------------|
| ubuntu:latest      | ~77 MB  | Yes   | apt             | HIGH           |
| debian:slim        | ~74 MB  | Yes   | apt             | HIGH           |
| alpine:3.19        | ~7 MB   | sh    | apk             | MEDIUM         |
| distroless/static  | ~2 MB   | No    | None            | LOW            |
| scratch            | 0 MB    | No    | None            | LOWEST         |

---

## Best Practice #2 — Image Scanning

> **Video Reference:** [07:26] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=446](https://www.youtube.com/watch?v=oBf5lrmquYI&t=446)

### Scanning Pipeline

```
  Image Build
      │
      ▼
  ┌──────────────────────────────────────────────────────────┐
  │                   Scanning Gates                         │
  │                                                          │
  │  ┌──────────────────────┐   ┌──────────────────────────┐ │
  │  │   Build-Time Scan    │   │   Registry-Level Scan    │ │
  │  │  (CI/CD — Block PR) │   │  (Ongoing — Alert+Block) │ │
  │  │  Trivy / Snyk / Grype│   │  ECR / Docker Hub / Quay │ │
  │  └──────────┬───────────┘   └────────────┬─────────────┘ │
  │             │                            │               │
  │  ┌──────────▼────────────────────────────▼─────────────┐ │
  │  │          Runtime Admission Scan                     │ │
  │  │  (Reject deploy if vulnerabilities exceed threshold) │ │
  │  │  Sysdig / Anchore / Aqua Security                   │ │
  │  └─────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────┘
```

### Trivy Scanning (CI/CD Integration)

```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# Scan image — fail build on HIGH/CRITICAL CVEs
trivy image \
  --exit-code 1 \
  --severity HIGH,CRITICAL \
  --no-progress \
  myapp:latest

# Scan as part of Dockerfile build (without pushing)
trivy image \
  --format json \
  --output trivy-report.json \
  myapp:latest

# Scan Kubernetes manifests
trivy config ./k8s-manifests/

# Scan running cluster
trivy k8s --report summary cluster
```

### GitHub Actions Integration

```yaml
# .github/workflows/security-scan.yaml
name: Container Security Scan
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: 1
      - name: Upload SARIF results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

---

## Best Practice #3 — Avoid Root & Privileged Containers

> **Video Reference:** [09:39] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=579](https://www.youtube.com/watch?v=oBf5lrmquYI&t=579)

### Risk Visualization

```
  Without Security Context          With Security Context
  ─────────────────────             ──────────────────────
  ┌─────────────────┐               ┌─────────────────┐
  │   Container     │               │   Container     │
  │   (root/uid=0)  │               │   (uid=1001)    │
  └────────┬────────┘               └────────┬────────┘
           │ breakout!                       │ contained
           ▼                                ▼
  ┌─────────────────┐               ┌─────────────────┐
  │   Host OS       │               │   Host OS       │
  │   root access   │               │  no privileges  │
  │   (FULL PWNED)  │               │  (safe)         │
  └─────────────────┘               └─────────────────┘
```

### Secure Pod Specification

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true               # Pod-level: enforce non-root
    runAsUser: 1001                  # Specific UID
    runAsGroup: 1001
    fsGroup: 2000                    # Volume ownership
    seccompProfile:
      type: RuntimeDefault           # Restrict syscalls
  containers:
    - name: app
      image: myapp:latest
      securityContext:
        allowPrivilegeEscalation: false   # Cannot gain more privs
        readOnlyRootFilesystem: true      # Read-only FS
        runAsNonRoot: true
        runAsUser: 1001
        capabilities:
          drop:
            - ALL                        # Drop ALL Linux capabilities
          add:
            - NET_BIND_SERVICE           # Only add what's needed
        seccompProfile:
          type: RuntimeDefault
      volumeMounts:
        - name: tmp                      # Writable tmp mount if needed
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
  hostNetwork: false                 # Never share host network
  hostPID: false                     # Never share host PID namespace
  hostIPC: false                     # Never share host IPC namespace
  automountServiceAccountToken: false # Disable unless needed
```

### Linux Capabilities Reference

| Capability      | Risk Level | Description                        |
|-----------------|------------|------------------------------------|
| `SYS_ADMIN`     | CRITICAL   | Equivalent to root — never grant   |
| `NET_ADMIN`     | HIGH       | Modify network interfaces          |
| `SYS_PTRACE`    | HIGH       | Debug other processes              |
| `DAC_OVERRIDE`  | MEDIUM     | Bypass file permission checks      |
| `NET_BIND_SERVICE` | LOW     | Bind ports < 1024 (acceptable)     |

---

## Best Practice #4 — RBAC & Least Privilege

> **Video Reference:** [11:24] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=684](https://www.youtube.com/watch?v=oBf5lrmquYI&t=684)

### RBAC Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        RBAC Model                               │
  │                                                                 │
  │   Subject (Who)          Binding            Role (What)         │
  │  ┌──────────────┐                       ┌──────────────────┐    │
  │  │ Human User   │──┐                 ┌──│  Role            │    │
  │  │ (cert-based) │  │                 │  │  (namespace)     │    │
  │  └──────────────┘  │  ┌──────────┐  │  └──────────────────┘    │
  │                    ├─▶│RoleBinding│─┤                           │
  │  ┌──────────────┐  │  └──────────┘  │  ┌──────────────────┐    │
  │  │ Service Acct │──┘                └──│  ClusterRole     │    │
  │  │ (token-based)│                      │  (cluster-wide)  │    │
  │  └──────────────┘                      └──────────────────┘    │
  │                                                                 │
  │   ┌────────────────────────────────────────────────────────┐   │
  │   │   Role defines: verbs × resources                     │   │
  │   │   verbs: get, list, watch, create, update, patch, delete│  │
  │   │   resources: pods, services, configmaps, secrets, etc. │   │
  │   └────────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────────┘
```

### RBAC Examples

```yaml
# Read-only Role for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-readonly
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-readonly-binding
  namespace: dev
subjects:
  - kind: User
    name: jane.developer
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-readonly
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# Minimal Service Account for CI pipeline
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production
automountServiceAccountToken: false      # Disable auto-mount
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deploy-role
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "patch", "update"]    # Only what CI needs
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deploy-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: ci-deploy-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Audit who has cluster-admin
kubectl get clusterrolebindings -o json | jq '
  .items[] | select(.roleRef.name == "cluster-admin") |
  {binding: .metadata.name, subjects: .subjects}'

# Check effective permissions for a user
kubectl auth can-i --list --as=jane.developer -n dev

# Audit all RBAC rules
kubectl get clusterroles,clusterrolebindings,roles,rolebindings --all-namespaces
```

---

## Best Practice #5 — Network Policies

> **Video Reference:** [16:19] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=979](https://www.youtube.com/watch?v=oBf5lrmquYI&t=979)

### Default vs Secured Network Model

```
  DEFAULT (Insecure)                   WITH NETWORK POLICIES
  ──────────────────                   ─────────────────────
  ┌─────────────────────────┐          ┌──────────────────────────────┐
  │  Frontend  ←──────────────────────▶  Frontend                    │
  │     │                   │          │     │ (only → Backend:8080)  │
  │     ▼                   │          │     ▼                        │
  │  Backend   ◀──────────────────────▶  Backend                     │
  │     │      (all can talk)│          │     │ (only → DB:5432)      │
  │     ▼                   │          │     ▼                        │
  │  Database  ◀──────────────────────▶  Database                    │
  │  (exposed to all!)      │          │  (only from Backend)  BLOCK  │
  └─────────────────────────┘          └──────────────────────────────┘
```

### Network Policy Examples

```yaml
# Default deny-all policy (apply FIRST in every namespace)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}              # Selects ALL pods
  policyTypes:
    - Ingress
    - Egress
---
# Allow frontend to receive traffic from ingress only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-ingress
      ports:
        - protocol: TCP
          port: 8080
---
# Allow backend to receive from frontend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
---
# Allow backend egress to database only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:                        # Allow DNS resolution
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

---

## Best Practice #6 — Service Mesh & mTLS

> **Video Reference:** [17:30] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=1050](https://www.youtube.com/watch?v=oBf5lrmquYI&t=1050)

### Service Mesh Architecture

```
  Without mTLS                         With Istio mTLS
  ────────────                         ──────────────
  ┌──────────┐                         ┌───────────────────────┐
  │ Service A│──── plain HTTP ────────▶│ Service A             │
  │          │     (eavesdropped!)     │  └─ Envoy Sidecar ────┼─── ENCRYPTED ───▶ ┌───────────────────────┐
  └──────────┘                         └───────────────────────┘  mTLS (verified)   │ Service B             │
                                                                                    │  └─ Envoy Sidecar     │
                                                                                    └───────────────────────┘
                                       ┌─────────────────────────────────────────┐
                                       │           Istio Control Plane            │
                                       │   istiod: cert mgmt, policy, telemetry  │
                                       └─────────────────────────────────────────┘
```

### Istio mTLS Configuration

```yaml
# Enable strict mTLS cluster-wide
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system          # Applies cluster-wide
spec:
  mtls:
    mode: STRICT                   # Reject any non-mTLS traffic
---
# Allow specific service to accept plaintext (migration period)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-service-permissive
  namespace: production
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: PERMISSIVE               # Accept both TLS and plaintext
---
# AuthorizationPolicy: only frontend can call backend
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/frontend-sa"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

---

## Best Practice #7 — Secrets Management

> **Video Reference:** [19:21] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=1161](https://www.youtube.com/watch?v=oBf5lrmquYI&t=1161)

### Secret Storage Hierarchy (Most to Least Secure)

```
  ┌────────────────────────────────────────────────────────┐
  │  LEVEL 4: External Secret Manager (Most Secure)        │
  │  HashiCorp Vault / AWS Secrets Manager / Azure KV      │
  │  • Centralized, audited, dynamic secrets               │
  │  • Auto-rotation, lease management                     │
  └────────────────────────────────────────────────────────┘
              ▲ preferred
  ┌────────────────────────────────────────────────────────┐
  │  LEVEL 3: Kubernetes Secrets + Encryption at Rest      │
  │  • EncryptionConfiguration with KMS provider           │
  │  • Secrets encrypted in etcd                           │
  └────────────────────────────────────────────────────────┘
  ┌────────────────────────────────────────────────────────┐
  │  LEVEL 2: Sealed Secrets / SOPS                        │
  │  • GitOps-safe encrypted secrets                       │
  │  • Decrypted only inside cluster by controller         │
  └────────────────────────────────────────────────────────┘
  ┌────────────────────────────────────────────────────────┐
  │  LEVEL 1: Plain Kubernetes Secrets (Avoid)             │
  │  • Only base64-encoded — not encrypted!                │
  │  • Anyone with get/list secret RBAC can decode         │
  └────────────────────────────────────────────────────────┘
```

### Enable Encryption at Rest

```yaml
# /etc/kubernetes/encryption-config.yaml (on API server nodes)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}               # Fallback for unencrypted data
```

```bash
# Generate a strong encryption key
head -c 32 /dev/urandom | base64

# Apply on kube-apiserver (add flag):
# --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# Verify secrets are encrypted in etcd
ETCDCTL_API=3 etcdctl get \
  /registry/secrets/default/my-secret \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  | hexdump -C | head    # Should show k8s:enc:aescbc: prefix
```

### HashiCorp Vault Integration

```yaml
# External Secrets Operator + Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "production-role"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

---

## Best Practice #8 — etcd Security

> **Video Reference:** [20:33] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=1233](https://www.youtube.com/watch?v=oBf5lrmquYI&t=1233)

### etcd Threat Model

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        etcd Security                            │
  │                                                                 │
  │   WHAT etcd STORES:                                             │
  │   • All Kubernetes resources (pods, services, secrets)          │
  │   • Cluster configuration and state                             │
  │   • RBAC policies                                               │
  │                                                                 │
  │   RISK: Direct etcd access = bypass Kubernetes RBAC entirely    │
  │                                                                 │
  │  ┌──────────────┐       ┌──────────────┐      ┌────────────┐   │
  │  │ API Server   │──────▶│    etcd      │      │  Attacker  │   │
  │  │ (port 6443)  │ mTLS  │ (port 2379)  │◀─ ─ ─│ (BLOCKED!) │   │
  │  └──────────────┘       │              │      └────────────┘   │
  │                         │  Encrypted   │                        │
  │                         │  at rest     │                        │
  │                         └──────────────┘                        │
  │                               │                                 │
  │                         ┌─────▼──────┐                          │
  │                         │  Firewall  │                          │
  │                         │  Allow:    │                          │
  │                         │  API→2379  │                          │
  │                         │  Peer:2380 │                          │
  │                         │  DENY all  │                          │
  │                         └────────────┘                          │
  └─────────────────────────────────────────────────────────────────┘
```

### etcd Hardening Checklist

```bash
# 1. Verify etcd peer TLS
etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd.crt \
  --key=/etc/etcd/etcd.key

# 2. Verify etcd client TLS flags are set
grep -E "trusted-ca-file|cert-file|key-file|client-cert-auth|peer-client-cert-auth" \
  /etc/etcd/etcd.conf

# 3. Verify etcd is not listening on 0.0.0.0
ss -tlnp | grep 2379    # Should show only internal IPs

# 4. Check etcd encryption configuration
etcdctl get /registry/secrets/default/test-secret \
  --endpoints=https://127.0.0.1:2379 | hexdump | head
# Should start with: k8s:enc:aescbc

# 5. Backup etcd regularly
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

### Required etcd API Server Flags

```bash
# kube-apiserver must have:
--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
--etcd-servers=https://127.0.0.1:2379          # Localhost only

# etcd must have:
--client-cert-auth=true
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--peer-client-cert-auth=true
--peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

---

## Best Practice #9 — Backup & Disaster Recovery

> **Video Reference:** [22:25] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=1345](https://www.youtube.com/watch?v=oBf5lrmquYI&t=1345)

### Backup Strategy

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    Backup Layers                             │
  │                                                              │
  │   Layer 1: etcd Snapshot (Cluster State)                    │
  │   ┌──────────────────────────────────────────────────────┐  │
  │   │  etcdctl snapshot save → encrypted S3/NFS            │  │
  │   │  Frequency: Every 15 min / before upgrades           │  │
  │   └──────────────────────────────────────────────────────┘  │
  │                                                              │
  │   Layer 2: Kubernetes Manifests (GitOps)                    │
  │   ┌──────────────────────────────────────────────────────┐  │
  │   │  All resources in Git (ArgoCD / Flux)                │  │
  │   │  Frequency: Every commit                             │  │
  │   └──────────────────────────────────────────────────────┘  │
  │                                                              │
  │   Layer 3: Persistent Volume Data (Application Data)       │
  │   ┌──────────────────────────────────────────────────────┐  │
  │   │  Velero / Kasten K10 → S3-compatible storage         │  │
  │   │  Frequency: Daily / hourly for critical data         │  │
  │   └──────────────────────────────────────────────────────┘  │
  │                                                              │
  │   Layer 4: Immutable Backups (Ransomware Protection)        │
  │   ┌──────────────────────────────────────────────────────┐  │
  │   │  S3 Object Lock / Immutable storage target           │  │
  │   │  Frequency: Replicated from Layer 3                  │  │
  │   └──────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────┘
```

### Velero Backup Setup

```bash
# Install Velero with AWS S3 backend
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket my-k8s-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-node-agent

# Create scheduled backup
velero schedule create daily-full-backup \
  --schedule="0 2 * * *" \
  --ttl 720h \
  --include-namespaces production,staging \
  --storage-location default

# Manual backup before upgrades
velero backup create pre-upgrade-backup \
  --include-cluster-resources=true \
  --wait

# Restore from backup
velero restore create --from-backup pre-upgrade-backup

# Verify backup status
velero backup describe daily-full-backup-20260630 --details
```

---

## Best Practice #10 — Policy Enforcement (OPA / Kyverno)

> **Video Reference:** [25:18] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=1518](https://www.youtube.com/watch?v=oBf5lrmquYI&t=1518)

### Admission Control Flow

```
  kubectl apply
       │
       ▼
  API Server
       │
       ▼
  ┌─────────────────────────────────────────┐
  │     Admission Controllers               │
  │                                         │
  │  ┌─────────────────────────────────┐    │
  │  │  Mutating Webhooks              │    │
  │  │  (Modify / add defaults)        │    │
  │  │  Kyverno Mutate / OPA Mutate    │    │
  │  └─────────────────────────────────┘    │
  │                  │                      │
  │  ┌───────────────▼─────────────────┐    │
  │  │  Validating Webhooks            │    │
  │  │  (Allow / Deny)                 │    │
  │  │  Kyverno Validate / OPA/Gatekeeper  │
  │  └─────────────────────────────────┘    │
  └─────────────────────────────────────────┘
       │                       │
       ▼                       ▼
   ADMITTED               REJECTED (403)
```

### Kyverno Policies

```yaml
# Policy 1: Disallow privileged containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
spec:
  validationFailureAction: Enforce      # Block (use Audit to test first)
  rules:
    - name: no-privileged
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
---
# Policy 2: Require non-root user
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-non-root
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Containers must not run as root (uid=0)."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
---
# Policy 3: Require image from trusted registry
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: trusted-registries-only
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Images must come from registry.company.com or gcr.io/distroless"
        pattern:
          spec:
            containers:
              - image: "registry.company.com/* | gcr.io/distroless/*"
---
# Policy 4: Auto-add security context (Mutating)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-security-context
spec:
  rules:
    - name: add-securitycontext
      match:
        any:
          - resources:
              kinds: [Pod]
      mutate:
        patchStrategicMerge:
          spec:
            securityContext:
              +(runAsNonRoot): true
              +(runAsUser): 1001
            containers:
              - (name): "*"
                securityContext:
                  +(allowPrivilegeEscalation): false
                  +(readOnlyRootFilesystem): true
```

### OPA Gatekeeper Example

```yaml
# ConstraintTemplate: No hostPath volumes
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: nohostpath
spec:
  crd:
    spec:
      names:
        kind: NoHostPath
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package nohostpath
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          has_field(volume, "hostPath")
          msg := sprintf("HostPath volume '%v' is not allowed", [volume.name])
        }
        has_field(object, field) {
          object[field]
        }
---
# Apply the constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: NoHostPath
metadata:
  name: no-hostpath-volumes
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  enforcementAction: deny
```

---

## Best Practice #11 — Automated Disaster Recovery

> **Video Reference:** [27:03] — [https://www.youtube.com/watch?v=oBf5lrmquYI&t=1623](https://www.youtube.com/watch?v=oBf5lrmquYI&t=1623)

### Recovery Time Objectives

```
  ┌─────────────────────────────────────────────────────────────────┐
  │              Disaster Recovery Tiers                            │
  │                                                                 │
  │   Tier 1: Pod/Deployment failure                               │
  │   RTO: Seconds — Kubernetes self-healing restarts pods         │
  │                                                                 │
  │   Tier 2: Node failure                                          │
  │   RTO: Minutes — Scheduler reschedules to healthy nodes        │
  │                                                                 │
  │   Tier 3: Namespace / App data corruption                      │
  │   RTO: Minutes — Velero/K10 namespace restore                  │
  │                                                                 │
  │   Tier 4: Full cluster failure                                  │
  │   RTO: 30–60 min — etcd restore + cluster rebuild              │
  │                                                                 │
  │   Tier 5: Region failure                                        │
  │   RTO: Hours — Cross-region cluster failover                   │
  └─────────────────────────────────────────────────────────────────┘
```

### Automated DR Testing

```bash
# Schedule monthly DR drill with Velero
velero schedule create monthly-dr-test \
  --schedule="0 4 1 * *" \
  --include-namespaces staging

# Test restore to isolated namespace
velero restore create dr-test-restore \
  --from-backup monthly-dr-test-backup \
  --namespace-mappings production:dr-validation \
  --wait

# Validate restored resources
kubectl get all -n dr-validation
kubectl get pvc -n dr-validation

# Cleanup test namespace after validation
kubectl delete namespace dr-validation
```

---

## Additional Best Practices

These practices extend beyond the video and reflect current security standards.

---

### A. Runtime Security Monitoring (Falco)

```
  ┌────────────────────────────────────────────────────────────────┐
  │                    Falco Runtime Security                      │
  │                                                                │
  │   Kernel (eBPF / syscall intercept)                           │
  │         │                                                      │
  │         ▼                                                      │
  │   ┌─────────────────────────────────────────┐                 │
  │   │  Falco Rules Engine                     │                 │
  │   │  • Shell spawned in container → ALERT   │                 │
  │   │  • /etc/passwd write → ALERT            │                 │
  │   │  • Unexpected outbound conn → ALERT     │                 │
  │   │  • Privilege escalation → ALERT         │                 │
  │   └────────────────────┬────────────────────┘                 │
  │                        │                                       │
  │        ┌───────────────┼────────────────┐                     │
  │        ▼               ▼                ▼                     │
  │   Slack Alert     SIEM/Splunk      PagerDuty                  │
  └────────────────────────────────────────────────────────────────┘
```

```bash
# Install Falco with Helm
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/..."
```

```yaml
# Custom Falco rule: detect crypto miners
- rule: Detect Crypto Mining
  desc: Detects known crypto mining binaries or network connections
  condition: >
    spawned_process and
    (proc.name in (xmrig, minerd, cpuminer) or
     (proc.name = curl and proc.args contains "stratum+tcp"))
  output: >
    Possible crypto mining detected (user=%user.name cmd=%proc.cmdline
    container=%container.name image=%container.image.repository)
  priority: CRITICAL
  tags: [cryptomining, malware]
```

---

### B. Pod Security Admission (PSA) — Kubernetes 1.25+

```bash
# Replace deprecated PodSecurityPolicy with Pod Security Admission
# Label namespaces to enforce security standards

# Baseline: prevent known privilege escalations
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# Restricted: most secure (deny hostPath, require non-root, etc.)
kubectl label namespace payment-service \
  pod-security.kubernetes.io/enforce=restricted
```

| Level        | What It Blocks                                                       |
|--------------|----------------------------------------------------------------------|
| `privileged` | Nothing blocked (fully open — avoid)                                 |
| `baseline`   | Privileged containers, hostPath, hostNetwork, most capabilities      |
| `restricted` | Everything above + requires non-root, seccomp, drops all caps        |

---

### C. Supply Chain Security (SLSA / Cosign)

```bash
# Sign container images with Cosign (Sigstore)
# Generate key pair
cosign generate-key-pair

# Sign image after build & push
cosign sign --key cosign.key registry.company.com/myapp:latest

# Verify signature before deploy
cosign verify \
  --key cosign.pub \
  registry.company.com/myapp:latest

# Kyverno policy to enforce image signatures
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-image-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - image: "registry.company.com/*"
          key: |-
            -----BEGIN PUBLIC KEY-----
            <your-cosign-public-key>
            -----END PUBLIC KEY-----
EOF
```

---

### D. API Server Hardening

```bash
# Required kube-apiserver flags for hardening
--anonymous-auth=false                           # No anonymous access
--authorization-mode=Node,RBAC                  # No AlwaysAllow
--enable-admission-plugins=NodeRestriction,PodSecurity,EventRateLimit
--audit-log-path=/var/log/kube-audit.log        # Enable audit logging
--audit-log-maxage=30                           # Keep 30 days
--audit-log-maxbackup=10
--audit-log-maxsize=100
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--tls-min-version=VersionTLS12                  # No TLS 1.0/1.1
--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,...
--disable-admission-plugins=AlwaysAdmit
--profiling=false                               # Disable profiling endpoint
--request-timeout=300s
```

```yaml
# Audit Policy (log secrets access, auth failures)
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: None
    resources:
      - group: ""
        resources: [events]
  - level: Metadata
    resources:
      - group: ""
        resources: [secrets, configmaps]
  - level: Request
    users: [system:anonymous]
  - level: RequestResponse
    verbs: [create, update, patch, delete]
    resources:
      - group: rbac.authorization.k8s.io
        resources: [clusterroles, clusterrolebindings]
  - level: Metadata
    omitStages: [RequestReceived]
```

---

### E. Node Security Hardening

```bash
# CIS Kubernetes Benchmark audit
docker run --rm --pid=host \
  -v /etc:/etc:ro \
  -v /var:/var:ro \
  aquasec/kube-bench:latest \
  --benchmark cis-1.8

# Key node hardening actions:
# 1. Disable unused kernel modules
echo "install cramfs /bin/true" >> /etc/modprobe.d/disable.conf
echo "install freevxfs /bin/true" >> /etc/modprobe.d/disable.conf

# 2. Enable kernel hardening params
cat >> /etc/sysctl.d/99-kubernetes.conf <<EOF
kernel.dmesg_restrict = 1
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.ip_forward = 1
kernel.randomize_va_space = 2
EOF
sysctl --system

# 3. Restrict kubelet anonymous auth
cat > /etc/kubernetes/kubelet-config.yaml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
readOnlyPort: 0
protectKernelDefaults: true
EOF
```

---

### F. Multi-Tenancy Isolation

```yaml
# Namespace isolation with quotas and limit ranges
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    secrets: "20"
    services.loadbalancers: "2"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 8Gi
```

---

### G. Vulnerability Management Workflow

```
  ┌──────────────────────────────────────────────────────────────────┐
  │               Continuous Vulnerability Management                │
  │                                                                  │
  │   Discover                                                       │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │  Trivy / Grype → Scan all images in registry weekly     │  │
  │   └──────────────────────────────────────────────────────────┘  │
  │                           │                                      │
  │   Triage                  ▼                                      │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │  CVSS ≥ 9.0 → Critical: patch within 24h               │  │
  │   │  CVSS 7.0-8.9 → High: patch within 7 days              │  │
  │   │  CVSS 4.0-6.9 → Medium: patch within 30 days           │  │
  │   │  CVSS < 4.0 → Low: schedule in next sprint             │  │
  │   └──────────────────────────────────────────────────────────┘  │
  │                           │                                      │
  │   Remediate               ▼                                      │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │  Update base image → rebuild → scan → deploy            │  │
  │   └──────────────────────────────────────────────────────────┘  │
  │                           │                                      │
  │   Verify                  ▼                                      │
  │   ┌──────────────────────────────────────────────────────────┐  │
  │   │  Re-scan → confirm CVE resolved → close ticket          │  │
  │   └──────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Security Checklist

### Image & Build

- [ ] Use distroless or minimal base images
- [ ] Multi-stage builds — no build tools in runtime image
- [ ] No secrets in Dockerfile or image layers
- [ ] All images scanned before push (Trivy / Snyk)
- [ ] Images signed with Cosign
- [ ] Only pull from trusted, internal registries

### Runtime

- [ ] All containers run as non-root (`runAsNonRoot: true`)
- [ ] No privileged containers (`privileged: false`)
- [ ] `allowPrivilegeEscalation: false` on all containers
- [ ] `readOnlyRootFilesystem: true` where possible
- [ ] All Linux capabilities dropped, only needed ones added
- [ ] `hostNetwork`, `hostPID`, `hostIPC` all false
- [ ] `automountServiceAccountToken: false` unless required
- [ ] Pod Security Admission labels on all namespaces

### Network

- [ ] Default-deny NetworkPolicy in every namespace
- [ ] Only required pod-to-pod communication allowed
- [ ] Service mesh (Istio) with strict mTLS enabled
- [ ] No NodePort services without firewall rules
- [ ] Ingress TLS enforced (no HTTP routes)

### Access Control

- [ ] RBAC enabled (no AlwaysAllow)
- [ ] No `cluster-admin` for regular users
- [ ] Service accounts follow least privilege
- [ ] `kubeadmin` / default admin credentials changed
- [ ] Anonymous authentication disabled
- [ ] Audit logging enabled and shipped to SIEM

### Secrets & Data

- [ ] Secrets encryption at rest enabled
- [ ] Secrets managed via Vault or external secret store
- [ ] No secrets in ConfigMaps, environment variables (direct), or Git
- [ ] Sealed Secrets or SOPS for GitOps secret storage

### Platform

- [ ] etcd behind firewall — API server only
- [ ] etcd TLS enabled (client + peer)
- [ ] etcd data encrypted at rest
- [ ] API server hardening flags applied
- [ ] Kubelet anonymous auth disabled
- [ ] CIS benchmark score reviewed (kube-bench)
- [ ] Falco runtime security monitoring active

### Policy & Compliance

- [ ] OPA Gatekeeper or Kyverno deployed
- [ ] Policies: no privileged, no hostPath, no root
- [ ] Policies in Enforce mode (not just Audit)
- [ ] Policy violations trigger alerts

### Backup & Recovery

- [ ] etcd snapshots automated (every 15 min)
- [ ] Application PV backups automated (Velero / K10)
- [ ] Immutable backup storage configured
- [ ] DR restore tested monthly
- [ ] RTO/RPO documented and validated

---

## Tools Reference

| Category              | Tool                    | Purpose                                  |
|-----------------------|-------------------------|------------------------------------------|
| Image Scanning        | Trivy                   | CVE scanning for images, manifests, IaC  |
| Image Scanning        | Snyk                    | Developer-centric vulnerability scanning |
| Image Scanning        | Grype                   | Fast CVE scanner (Anchore OSS)           |
| Image Scanning        | Sysdig Secure           | Runtime + image scanning platform        |
| Runtime Security      | Falco                   | Syscall-based runtime threat detection   |
| Runtime Security      | Aqua Security           | Full lifecycle container security        |
| Policy Enforcement    | Kyverno                 | Native K8s policy engine (YAML-based)    |
| Policy Enforcement    | OPA Gatekeeper          | Rego-based policy enforcement            |
| Secrets Management    | HashiCorp Vault         | Enterprise secrets management            |
| Secrets Management    | External Secrets Op.    | Sync external secrets to K8s             |
| Secrets Management    | Sealed Secrets          | GitOps-safe encrypted secrets            |
| Service Mesh          | Istio                   | mTLS, traffic management, AuthZ policies |
| Service Mesh          | Linkerd                 | Lightweight mTLS service mesh            |
| Network Policy        | Calico                  | CNI with advanced NetworkPolicy support  |
| Network Policy        | Cilium                  | eBPF-based CNI with L7 policies          |
| Backup & Recovery     | Velero                  | K8s backup and restore                   |
| Backup & Recovery     | Kasten K10              | Enterprise K8s data management           |
| Supply Chain          | Cosign (Sigstore)       | Container image signing and verification |
| Compliance Audit      | kube-bench              | CIS Kubernetes Benchmark checks          |
| Compliance Audit      | Polaris                 | K8s config best practices auditor        |
| Threat Detection      | KubeArmor               | eBPF-based workload protection           |
| SBOM                  | Syft                    | Generate Software Bill of Materials      |

---

## Key Takeaways

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  1.  Build security IN, not ON — start at the image, not after      │
│  2.  Kubernetes is NOT secure by default — every default is open    │
│  3.  Defense-in-depth: if one layer fails, others hold             │
│  4.  Least privilege everywhere: users, pods, service accounts      │
│  5.  Encrypt everything: secrets at rest, traffic in flight         │
│  6.  Automate compliance — humans forget, policies don't            │
│  7.  Assume breach — monitor runtime behavior continuously          │
│  8.  Backup + test restore — a backup never tested is not a backup  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

*Reference: [Kubernetes Security Best Practices — YouTube](https://www.youtube.com/watch?v=oBf5lrmquYI)*
*Additional standards: CIS Kubernetes Benchmark v1.8, NIST SP 800-190, NSA/CISA Kubernetes Hardening Guide*
