# EKS Cluster Upgrade User Manual
### 3-Node Cluster — Control Plane + Managed Node Group

---

## Table of Contents

1. [Overview](#overview)
2. [Important Warnings](#important-warnings)
3. [Prerequisites](#prerequisites)
4. [Release Notes & Version Lifecycle](#release-notes--version-lifecycle)
5. [Phase 1 — Pre-Upgrade Backup](#phase-1--pre-upgrade-backup)
6. [Phase 2 — Pre-Upgrade Health Check](#phase-2--pre-upgrade-health-check)
7. [Phase 3 — Upgrade Control Plane](#phase-3--upgrade-control-plane)
8. [Phase 4 — Upgrade the 3 Worker Nodes](#phase-4--upgrade-the-3-worker-nodes)
9. [Phase 5 — Upgrade Core Add-ons](#phase-5--upgrade-core-add-ons)
10. [Phase 6 — Post-Upgrade Validation](#phase-6--post-upgrade-validation)
11. [Rollback / Recovery](#rollback--recovery)
12. [Quick Reference Cheatsheet](#quick-reference-cheatsheet)

---

## Overview

This manual covers upgrading an Amazon EKS cluster (3-node setup) from one minor Kubernetes version to the next. The upgrade touches:

| Component | Description |
|---|---|
| Control Plane | AWS-managed API server, etcd, scheduler, controller-manager |
| Worker Nodes (×3) | EC2 instances running `kubelet` + `kube-proxy` |
| Core Add-ons | VPC CNI, CoreDNS, kube-proxy |
| kubectl (client) | Local CLI must stay within ±1 minor version of control plane |

**Upgrade order is mandatory:**
```
Backup → Pre-checks → Control Plane → Nodes → Add-ons → Validate
```

---

## Important Warnings

> **You cannot downgrade a cluster.** Once the control plane is upgraded, it cannot be rolled back. The only recovery path is restoring from backup or creating a new cluster.

> **One minor version at a time only.** EKS does not allow skipping versions. To go from 1.28 → 1.30 you must do 1.28 → 1.29 → 1.30.

> **Do not upgrade nodes before the control plane.** Nodes must always be at the same version as or one minor version below the control plane.

> **The upgrade cannot be paused or stopped** once initiated on the control plane. EKS will automatically rollback the control plane only if its own health checks fail — it will NOT roll back node changes you have already made.

---

## Prerequisites

### Tools Required

| Tool | Minimum Version | Install / Check |
|---|---|---|
| `kubectl` | Within ±1 minor of target version | `kubectl version --client` |
| `eksctl` | 0.215.0+ | `eksctl version` |
| `aws` CLI | Latest | `aws --version` |
| AWS Backup | Enabled | AWS Console → Backup |

### Access Requirements

- IAM permissions: `eks:UpdateClusterVersion`, `eks:DescribeUpdate`, `ec2:DescribeSubnets`, `ec2:DescribeSecurityGroups`
- `kubectl` configured with cluster credentials: `aws eks update-kubeconfig --name <cluster-name> --region <region>`
- At least **5 available IP addresses** in each subnet used by the cluster

### Cluster State Requirements

- All 3 nodes must be in `Ready` state
- No critical pending `PodDisruptionBudget` violations
- Control plane and all nodes on the same current minor version

---

## Release Notes & Version Lifecycle

### How to Find the Right Release Notes

1. **Kubernetes upstream changelog:** https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/
2. **Amazon EKS release notes:** https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html
3. **EKS platform version notes:** https://docs.aws.amazon.com/eks/latest/userguide/platform-versions.html

### Version Lifecycle — Key Facts

| Policy | Detail |
|---|---|
| Standard support | 14 months from release date on EKS |
| Extended support | Up to 26 months (additional cost) |
| Deprecated API check window | Rolling 30 days |
| Insight refresh cadence | Every 24 hours |

### What to Review in Release Notes Before Upgrading

- [ ] **Removed / deprecated APIs** — check the [Kubernetes Deprecated API Migration Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
- [ ] **Breaking changes** in admission webhooks, RBAC, or storage classes
- [ ] **Add-on minimum versions** — VPC CNI, CoreDNS, kube-proxy compatibility matrix
- [ ] **Node AMI changes** — new EKS-optimized AMI for the target version
- [ ] **Feature gates** — any gates moving from alpha → beta → GA that could affect workloads

### Example: Checking for Deprecated API Usage

```bash
# List upgrade insights for your cluster
aws eks list-insights \
  --cluster-name <cluster-name> \
  --region <region> \
  --filter category=UPGRADE_READINESS

# Get detail on a specific insight
aws eks describe-insight \
  --cluster-name <cluster-name> \
  --region <region> \
  --id <insight-id>
```

---

## Phase 1 — Pre-Upgrade Backup

### 1.1 Enable AWS Backup for EKS

AWS Backup can capture the full EKS cluster state including persistent volumes.

```bash
# Verify the EKS cluster resource type is registered with AWS Backup
aws backup list-backup-plans --region <region>
```

In the **AWS Console**:
1. Go to **AWS Backup → Backup plans → Create backup plan**
2. Choose **Build a new plan**
3. Set schedule (run immediately for a one-off pre-upgrade backup)
4. Add resource: **EKS cluster** → select your cluster
5. Set retention (minimum 30 days recommended)
6. Choose or create a **Backup vault**
7. Click **Create plan**

Then trigger an **on-demand backup**:
```bash
aws backup start-backup-job \
  --backup-vault-name <vault-name> \
  --resource-arn arn:aws:eks:<region>:<account-id>:cluster/<cluster-name> \
  --iam-role-arn arn:aws:iam::<account-id>:role/AWSBackupDefaultServiceRole \
  --region <region>
```

Wait for it to complete:
```bash
aws backup list-backup-jobs \
  --by-resource-arn arn:aws:eks:<region>:<account-id>:cluster/<cluster-name> \
  --region <region>
```
Status must be `COMPLETED` before proceeding.

### 1.2 Export Kubernetes Resource Manifests

Back up all namespaced and cluster-scoped resources as a safety net:

```bash
# Set your output directory
BACKUP_DIR=~/eks-backup-$(date +%Y%m%d)
mkdir -p $BACKUP_DIR

# Cluster-scoped resources
for resource in namespaces clusterroles clusterrolebindings storageclasses \
  persistentvolumes customresourcedefinitions; do
  kubectl get $resource -o yaml > $BACKUP_DIR/$resource.yaml
done

# Namespaced resources (repeat per namespace or use --all-namespaces)
kubectl get all --all-namespaces -o yaml > $BACKUP_DIR/all-workloads.yaml
kubectl get configmaps --all-namespaces -o yaml > $BACKUP_DIR/configmaps.yaml
kubectl get secrets --all-namespaces -o yaml > $BACKUP_DIR/secrets.yaml
kubectl get pvc --all-namespaces -o yaml > $BACKUP_DIR/pvcs.yaml
kubectl get ingress --all-namespaces -o yaml > $BACKUP_DIR/ingresses.yaml

echo "Backup saved to $BACKUP_DIR"
```

### 1.3 Record Current Cluster State

```bash
# Record current versions
kubectl version > $BACKUP_DIR/version-before.txt
kubectl get nodes -o wide >> $BACKUP_DIR/version-before.txt
kubectl get pods --all-namespaces >> $BACKUP_DIR/version-before.txt

# Record node group details
aws eks list-nodegroups --cluster-name <cluster-name> --region <region>
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --region <region> > $BACKUP_DIR/nodegroup-before.json
```

---

## Phase 2 — Pre-Upgrade Health Check

### 2.1 Verify Node & Control Plane Versions Match

```bash
# Control plane version
kubectl version --short

# All 3 nodes — must match control plane minor version
kubectl get nodes -o wide
```

Expected output example (all nodes on same version as control plane):
```
NAME        STATUS   VERSION   AGE
node-01     Ready    v1.29.x   30d
node-02     Ready    v1.29.x   30d
node-03     Ready    v1.29.x   30d
```

### 2.2 Check Cluster Health

```bash
# All nodes must be Ready
kubectl get nodes

# No pods in CrashLoopBackOff or Error state
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Check system pods
kubectl get pods -n kube-system
```

### 2.3 Review EKS Upgrade Insights (Console)

1. Open **EKS Console → Clusters → \<your cluster\> → Upgrade insights**
2. Resolve any **ERROR** or **WARNING** items before proceeding
3. Pay attention to deprecated API usage (insight status may lag up to 30 days)

### 2.4 Verify Subnet IP Availability

EKS requires **at least 5 free IPs** per subnet:

```bash
# List subnets used by the cluster
aws eks describe-cluster \
  --name <cluster-name> \
  --region <region> \
  --query "cluster.resourcesVpcConfig.subnetIds"

# Check available IPs in each subnet
aws ec2 describe-subnets \
  --subnet-ids <subnet-id-1> <subnet-id-2> \
  --query "Subnets[*].{ID:SubnetId, AvailableIPs:AvailableIpAddressCount}" \
  --region <region>
```

### 2.5 Check VPC CNI Version

```bash
kubectl describe daemonset aws-node -n kube-system | grep Image
```

If VPC CNI is below `1.8.0`, update it before proceeding:
```bash
# Update via EKS add-on
aws eks update-addon \
  --cluster-name <cluster-name> \
  --addon-name vpc-cni \
  --resolve-conflicts OVERWRITE \
  --region <region>
```

---

## Phase 3 — Upgrade Control Plane

Choose **one** of the three methods below.

### Method A — eksctl (Recommended)

```bash
# Dry-run first (no --approve flag)
eksctl upgrade cluster --name <cluster-name> --version <target-version>

# Execute upgrade
eksctl upgrade cluster \
  --name <cluster-name> \
  --version <target-version> \
  --approve
```

### Method B — AWS Console

1. Open **EKS Console → Clusters**
2. Click **Upgrade now** next to your cluster
3. Select the target version
4. Click **Upgrade**

### Method C — AWS CLI

```bash
# Submit the upgrade
aws eks update-cluster-version \
  --name <cluster-name> \
  --kubernetes-version <target-version> \
  --region <region>

# Note the update-id from the response, then monitor:
aws eks describe-update \
  --name <cluster-name> \
  --update-id <update-id> \
  --region <region>
```

### Monitoring Control Plane Upgrade

```bash
# Poll until status is Successful (typically 5–15 minutes)
watch -n 30 "aws eks describe-update \
  --name <cluster-name> \
  --update-id <update-id> \
  --region <region> \
  --query 'update.status' \
  --output text"
```

Expected final status: `Successful`

### Verify Control Plane Version

```bash
kubectl version --short
# Server version should now show the new minor version
```

---

## Phase 4 — Upgrade the 3 Worker Nodes

> Nodes must be upgraded **after** the control plane. Upgrade nodes one at a time or use a rolling managed node group update.

### 4.1 Managed Node Group Rolling Update (Recommended)

```bash
# Trigger rolling update for the node group
aws eks update-nodegroup-version \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --region <region>

# Monitor the node group update
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --region <region> \
  --query "nodegroup.status"
```

EKS will:
1. Launch a new node with the updated AMI
2. Cordon and drain the old node (respecting PodDisruptionBudgets)
3. Terminate the old node
4. Repeat for all 3 nodes

### 4.2 Monitor Node Replacement

```bash
# Watch nodes cycling through (run in a separate terminal)
watch kubectl get nodes -o wide
```

Typical sequence for each of the 3 nodes:
```
node-01   NotReady,SchedulingDisabled   v1.29.x   (draining)
node-01   (terminated)
node-01-new   Ready                     v1.30.x   (new node joined)
```

### 4.3 eksctl Alternative

```bash
eksctl upgrade nodegroup \
  --name <nodegroup-name> \
  --cluster <cluster-name> \
  --region <region>
```

### 4.4 Self-Managed Nodes (if applicable)

For self-managed nodes, drain and replace each node manually:

```bash
# For each of the 3 nodes:
NODE=<node-name>

# Cordon (prevent new scheduling)
kubectl cordon $NODE

# Drain (evict existing pods gracefully)
kubectl drain $NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# Terminate and replace EC2 instance with new EKS-optimized AMI
# (done via Auto Scaling Group or manually in EC2 console)

# After new node joins, verify
kubectl get nodes
```

### 4.5 Verify All 3 Nodes Are Upgraded

```bash
kubectl get nodes -o wide
```

All 3 nodes must show `Ready` status and the new Kubernetes version:
```
NAME        STATUS   VERSION   AGE
node-01     Ready    v1.30.x   5m
node-02     Ready    v1.30.x   8m
node-03     Ready    v1.30.x   11m
```

---

## Phase 5 — Upgrade Core Add-ons

### 5.1 Check Current Add-on Versions

```bash
aws eks list-addons \
  --cluster-name <cluster-name> \
  --region <region>

# Get version detail for each add-on
for addon in vpc-cni coredns kube-proxy; do
  echo "--- $addon ---"
  aws eks describe-addon \
    --cluster-name <cluster-name> \
    --addon-name $addon \
    --region <region> \
    --query "addon.{version:addonVersion,status:status}"
done
```

### 5.2 Get Available Versions for Target Kubernetes Version

```bash
aws eks describe-addon-versions \
  --kubernetes-version <target-version> \
  --region <region> \
  --query "addons[*].{name:addonName, latest:addonVersions[0].addonVersion}"
```

### 5.3 Update Each Add-on

#### VPC CNI

```bash
aws eks update-addon \
  --cluster-name <cluster-name> \
  --addon-name vpc-cni \
  --addon-version <latest-compatible-version> \
  --resolve-conflicts OVERWRITE \
  --region <region>
```

#### CoreDNS

```bash
aws eks update-addon \
  --cluster-name <cluster-name> \
  --addon-name coredns \
  --addon-version <latest-compatible-version> \
  --resolve-conflicts OVERWRITE \
  --region <region>
```

#### kube-proxy

```bash
aws eks update-addon \
  --cluster-name <cluster-name> \
  --addon-name kube-proxy \
  --addon-version <latest-compatible-version> \
  --resolve-conflicts OVERWRITE \
  --region <region>
```

### 5.4 Monitor Add-on Updates

```bash
# Check status of all add-ons
aws eks list-addons \
  --cluster-name <cluster-name> \
  --region <region> \
  --output table

# All should reach ACTIVE status
watch -n 10 "aws eks describe-addon \
  --cluster-name <cluster-name> \
  --addon-name vpc-cni \
  --region <region> \
  --query addon.status \
  --output text"
```

### 5.5 Update kubectl (Local Client)

`kubectl` must be within one minor version of the cluster control plane.

```bash
# Check current version
kubectl version --client

# Update via package manager (Linux example)
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# macOS via brew
brew upgrade kubectl

# Windows via choco
choco upgrade kubernetes-cli
```

### 5.6 Update Cluster Autoscaler (if deployed)

```bash
# Find matching version at: https://github.com/kubernetes/autoscaler/releases
# Set image tag to match your new Kubernetes minor version (e.g., 1.30.x)
kubectl -n kube-system set image \
  deployment.apps/cluster-autoscaler \
  cluster-autoscaler=registry.k8s.io/autoscaling/cluster-autoscaler:v<X.XX.X>
```

---

## Phase 6 — Post-Upgrade Validation

### 6.1 Full Cluster Health Check

```bash
# All nodes Ready on new version
kubectl get nodes -o wide

# All system pods running
kubectl get pods -n kube-system

# All application pods running
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed
```

### 6.2 Verify Add-on Versions

```bash
# VPC CNI
kubectl describe daemonset aws-node -n kube-system | grep Image

# CoreDNS
kubectl describe deployment coredns -n kube-system | grep Image

# kube-proxy
kubectl describe daemonset kube-proxy -n kube-system | grep Image
```

### 6.3 Check Cluster Control Plane Version

```bash
kubectl version --short
aws eks describe-cluster \
  --name <cluster-name> \
  --region <region> \
  --query "cluster.{version:version, platformVersion:platformVersion, status:status}"
```

### 6.4 Run Smoke Tests

```bash
# Test DNS resolution
kubectl run dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default
kubectl logs dns-test
kubectl delete pod dns-test

# Test pod scheduling across all 3 nodes
kubectl run node-test --image=nginx --restart=Never
kubectl get pod node-test -o wide
kubectl delete pod node-test

# Test persistent volume provisioning (if using EBS/EFS)
kubectl apply -f $BACKUP_DIR/pvcs.yaml --dry-run=client
```

### 6.5 Save Post-Upgrade State

```bash
kubectl version > ~/eks-backup-$(date +%Y%m%d)/version-after.txt
kubectl get nodes -o wide >> ~/eks-backup-$(date +%Y%m%d)/version-after.txt
kubectl get pods --all-namespaces >> ~/eks-backup-$(date +%Y%m%d)/version-after.txt
```

---

## Rollback / Recovery

> Since control plane downgrade is not supported, prevention (backup + pre-checks) is the only true rollback path for the control plane.

### Scenario 1: Control Plane Upgrade Failed

EKS automatically rolls back the control plane if its own health checks fail. No action needed — your cluster will remain on the previous version.

Verify:
```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --region <region> \
  --query "cluster.{version:version, status:status}"
```

### Scenario 2: Node Upgrade Failure

If a node fails to join after replacement:

```bash
# Check node group status
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --region <region>

# Force-rollback the node group update (if still in progress)
aws eks update-nodegroup-version \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --force \
  --region <region>
```

### Scenario 3: Application Issues After Upgrade

```bash
# Restore workload manifests from backup
kubectl apply -f $BACKUP_DIR/all-workloads.yaml

# Restore specific resource
kubectl apply -f $BACKUP_DIR/<resource>.yaml
```

### Scenario 4: Full Cluster Restore from AWS Backup

1. Go to **AWS Backup Console → Backup vaults → \<vault-name\>**
2. Find the recovery point taken before the upgrade
3. Click **Restore**
4. This creates a **new EKS cluster** from the backup state
5. Update your kubeconfig to point to the restored cluster:
   ```bash
   aws eks update-kubeconfig \
     --name <restored-cluster-name> \
     --region <region>
   ```

---

## Quick Reference Cheatsheet

```bash
# ─── BACKUP ───────────────────────────────────────────────────
aws backup start-backup-job --backup-vault-name <vault> \
  --resource-arn arn:aws:eks:<region>:<acct>:cluster/<name> \
  --iam-role-arn arn:aws:iam::<acct>:role/AWSBackupDefaultServiceRole

# ─── PRE-CHECKS ───────────────────────────────────────────────
kubectl get nodes -o wide
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed
aws eks list-insights --cluster-name <name> --region <region>

# ─── UPGRADE CONTROL PLANE ────────────────────────────────────
eksctl upgrade cluster --name <name> --version <ver> --approve
# OR
aws eks update-cluster-version --name <name> \
  --kubernetes-version <ver> --region <region>

# ─── UPGRADE NODES ────────────────────────────────────────────
aws eks update-nodegroup-version --cluster-name <name> \
  --nodegroup-name <ng-name> --region <region>

# ─── UPGRADE ADD-ONS ──────────────────────────────────────────
for addon in vpc-cni coredns kube-proxy; do
  aws eks update-addon --cluster-name <name> --addon-name $addon \
    --resolve-conflicts OVERWRITE --region <region>
done

# ─── VALIDATE ─────────────────────────────────────────────────
kubectl version --short
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

---

*Generated: 2026-06-29 | Based on AWS EKS official documentation*
*Always verify against the latest EKS release notes before performing an upgrade.*
