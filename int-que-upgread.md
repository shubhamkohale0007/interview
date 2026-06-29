# EKS Upgrade — Interview Questions & Answers
### For 8+ Years Experience | Easy English

---

## Table of Contents

1. [Basic Concepts](#1-basic-concepts)
2. [Pre-Upgrade Planning](#2-pre-upgrade-planning)
3. [Backup & Safety](#3-backup--safety)
4. [Control Plane Upgrade](#4-control-plane-upgrade)
5. [Node Upgrade](#5-node-upgrade)
6. [Add-ons & Components](#6-add-ons--components)
7. [Troubleshooting](#7-troubleshooting)
8. [Best Practices](#8-best-practices)
9. [Scenario-Based Questions](#9-scenario-based-questions)

---

## 1. Basic Concepts

---

**Q1. What is EKS and why do we upgrade it?**

**A:**
EKS is Amazon's managed Kubernetes service. AWS runs the control plane for you.
We upgrade EKS because:
- Old versions stop getting security fixes
- New versions bring better features and stability
- AWS stops supporting old versions after about 14 months

---

**Q2. Can we skip versions during EKS upgrade? For example, go from 1.27 directly to 1.30?**

**A:**
No. EKS allows only **one minor version at a time**.
To go from 1.27 to 1.30 you must do:
```
1.27 → 1.28 → 1.29 → 1.30
```
This is because Kubernetes has strict version skew rules.

---

**Q3. Can we downgrade an EKS cluster after upgrading?**

**A:**
No. Once you upgrade the control plane, you **cannot go back**.
The only option is:
- Restore from a backup (creates a new cluster)
- Build a fresh cluster on the old version and migrate workloads

---

**Q4. What are the main parts that need to be upgraded in EKS?**

**A:**
There are 4 main parts:

| Part | What it is |
|---|---|
| Control Plane | AWS-managed API server, etcd, scheduler |
| Worker Nodes | EC2 machines running your pods |
| Core Add-ons | VPC CNI, CoreDNS, kube-proxy |
| kubectl (client) | Your local command-line tool |

---

**Q5. What is the correct order to upgrade EKS?**

**A:**
Always follow this order:
```
1. Take Backup
2. Check cluster health
3. Upgrade Control Plane
4. Upgrade Worker Nodes
5. Upgrade Add-ons (VPC CNI, CoreDNS, kube-proxy)
6. Update kubectl
7. Validate everything works
```
Never upgrade nodes before the control plane.

---

## 2. Pre-Upgrade Planning

---

**Q6. What checks do you do before starting an EKS upgrade?**

**A:**
I check these things before upgrading:

1. **Node versions match control plane** — all nodes must be on the same version
2. **All nodes are Ready** — `kubectl get nodes`
3. **No failing pods** — `kubectl get pods --all-namespaces`
4. **Deprecated API usage** — check EKS upgrade insights in the console
5. **Subnet has 5+ free IPs** — EKS needs them for new network interfaces
6. **VPC CNI version is 1.8.0+** — older versions can cause issues
7. **Backup is taken** — always before starting

---

**Q7. What are EKS Upgrade Insights?**

**A:**
EKS Upgrade Insights is a built-in tool that **automatically scans your cluster** for problems before upgrade.
It checks things like:
- Are you using any APIs that will be removed in the new version?
- Are there any configuration issues?

You can see it in the EKS Console under your cluster → **Upgrade Insights** tab.

```bash
# Also check via CLI
aws eks list-insights \
  --cluster-name my-cluster \
  --region us-east-1 \
  --filter category=UPGRADE_READINESS
```

---

**Q8. What is version skew in Kubernetes and why does it matter for upgrade?**

**A:**
Version skew means the **difference in versions** between components.

Rules:
- Node `kubelet` can be **at most 3 minor versions older** than the API server (from Kubernetes 1.28+)
- `kubectl` must be within **±1 minor version** of the API server

Example: If control plane is 1.30, nodes can be 1.27, 1.28, 1.29, or 1.30 — but not 1.26.

This matters because if skew is too big, things break silently.

---

**Q9. How do you check the current versions of your cluster and nodes?**

**A:**
```bash
# Control plane version
kubectl version --short

# All node versions
kubectl get nodes -o wide

# EKS cluster details
aws eks describe-cluster \
  --name my-cluster \
  --region us-east-1 \
  --query "cluster.version"
```

---

**Q10. How do you check if a subnet has enough free IP addresses for upgrade?**

**A:**
EKS needs at least 5 free IPs in cluster subnets to create new network interfaces.

```bash
# Get subnets used by cluster
aws eks describe-cluster \
  --name my-cluster \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig.subnetIds"

# Check free IPs in each subnet
aws ec2 describe-subnets \
  --subnet-ids subnet-abc123 \
  --query "Subnets[*].{ID:SubnetId, FreeIPs:AvailableIpAddressCount}"
```

---

## 3. Backup & Safety

---

**Q11. How do you back up an EKS cluster before upgrading?**

**A:**
I use two methods together:

**Method 1 — AWS Backup (for full cluster + persistent volumes)**
```bash
aws backup start-backup-job \
  --backup-vault-name my-vault \
  --resource-arn arn:aws:eks:us-east-1:123456789:cluster/my-cluster \
  --iam-role-arn arn:aws:iam::123456789:role/AWSBackupDefaultServiceRole
```

**Method 2 — Export Kubernetes manifests**
```bash
kubectl get all --all-namespaces -o yaml > all-workloads.yaml
kubectl get configmaps --all-namespaces -o yaml > configmaps.yaml
kubectl get secrets --all-namespaces -o yaml > secrets.yaml
kubectl get pvc --all-namespaces -o yaml > pvcs.yaml
```

Both together give full safety coverage.

---

**Q12. What is AWS Backup and how does it help with EKS?**

**A:**
AWS Backup is a centralized backup service from AWS.
For EKS it can:+
- Snapshot the entire cluster state
- Back up EBS persistent volumes used by pods
- Store backups in a **Backup Vault** with encryption
- Restore the cluster if something goes wrong (creates a new cluster from backup)

It is the safest way to back up before a major upgrade.

---

**Q13. If upgrade fails, how do you restore from backup?**

**A:**
1. Go to **AWS Backup Console → Backup Vaults → your vault**
2. Find the recovery point taken before upgrade
3. Click **Restore** — this creates a **new EKS cluster** from the backup
4. Update your kubeconfig:
```bash
aws eks update-kubeconfig \
  --name restored-cluster-name \
  --region us-east-1
```
5. Verify everything is working
6. Redirect traffic to the new cluster

Note: You cannot restore into the same cluster — it always creates a new one.

---

## 4. Control Plane Upgrade

---

**Q14. How do you upgrade the EKS control plane?**

**A:**
Three ways — I prefer eksctl as it is simple:

**Using eksctl:**
```bash
eksctl upgrade cluster \
  --name my-cluster \
  --version 1.30 \
  --approve
```

**Using AWS CLI:**
```bash
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.30 \
  --region us-east-1
```

**Using Console:**
EKS Console → Clusters → Click **Upgrade now** → Select version → Confirm

---

**Q15. How long does control plane upgrade take and how do you monitor it?**

**A:**
Usually **5 to 15 minutes**.

Monitor using CLI:
```bash
aws eks describe-update \
  --name my-cluster \
  --update-id <update-id> \
  --region us-east-1 \
  --query "update.status"
```

Status values:
- `InProgress` — upgrade is running
- `Successful` — done, move to next step
- `Failed` — AWS rolled back, check errors

---

**Q16. Can you pause or cancel a control plane upgrade once started?**

**A:**
No. Once started, the control plane upgrade **cannot be paused or stopped**.

However:
- If AWS health checks fail, EKS **automatically rolls back** the control plane
- Your running applications are NOT affected during control plane upgrade
- The cluster is never left in a broken state by AWS

---

**Q17. What happens to running applications during control plane upgrade?**

**A:**
Running applications **continue to work normally** during control plane upgrade.

AWS does a rolling replacement of API server nodes behind the scenes. Your pods keep running. The only thing you may notice is a brief moment where `kubectl` commands are slow or timeout — this is normal.

---

## 5. Node Upgrade

---

**Q18. How do you upgrade worker nodes in EKS?**

**A:**
For **managed node groups** (most common):
```bash
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --region us-east-1
```

EKS automatically:
1. Launches a new node with updated AMI
2. Cordons the old node (stops new pods going there)
3. Drains the old node (moves pods safely)
4. Terminates the old node
5. Repeats for each node

For a 3-node cluster this happens one node at a time.

---

**Q19. What is cordon and drain? Why is it important during node upgrade?**

**A:**
**Cordon** — marks a node as unschedulable so no new pods land on it:
```bash
kubectl cordon node-01
```

**Drain** — safely moves all existing pods off the node:
```bash
kubectl drain node-01 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60
```

This is important because it gives pods time to shut down cleanly and restart on other nodes. Without drain, pods get killed suddenly which can cause downtime.

---

**Q20. What is a PodDisruptionBudget (PDB) and how does it affect node upgrade?**

**A:**
A PDB is a rule that says **"keep at least X pods of this app running at all times"**.

Example:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

During node drain, EKS respects PDBs. If draining a node would violate a PDB, the drain waits. This protects app availability but can slow down or block node upgrades if PDB rules are too strict.

---

**Q21. How do you upgrade self-managed nodes (not managed node group)?**

**A:**
For self-managed nodes you do it manually, one node at a time:

```bash
# Step 1: Cordon the node
kubectl cordon node-01

# Step 2: Drain the node
kubectl drain node-01 \
  --ignore-daemonsets \
  --delete-emptydir-data

# Step 3: Terminate old EC2 instance and launch new one
# (use new EKS-optimized AMI for the target version)
# Do this in EC2 console or via Auto Scaling Group

# Step 4: Verify new node joined
kubectl get nodes

# Repeat for node-02 and node-03
```

---

**Q22. How do you find the latest EKS-optimized AMI for a specific version?**

**A:**
```bash
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.30/amazon-linux-2/recommended/image_id \
  --region us-east-1 \
  --query "Parameter.Value" \
  --output text
```

Replace `1.30` with your target Kubernetes version.

---

## 6. Add-ons & Components

---

**Q23. Which core add-ons must be updated after EKS upgrade?**

**A:**
Three must-update add-ons:

| Add-on | Purpose |
|---|---|
| **VPC CNI** | Gives pods their IP addresses |
| **CoreDNS** | DNS for the cluster |
| **kube-proxy** | Manages network rules on nodes |

These must match or be compatible with the new Kubernetes version.

---

**Q24. How do you update EKS add-ons after upgrade?**

**A:**
```bash
# Update VPC CNI
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --resolve-conflicts OVERWRITE \
  --region us-east-1

# Update CoreDNS
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name coredns \
  --resolve-conflicts OVERWRITE \
  --region us-east-1

# Update kube-proxy
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name kube-proxy \
  --resolve-conflicts OVERWRITE \
  --region us-east-1
```

Check status:
```bash
aws eks describe-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --region us-east-1 \
  --query "addon.status"
```
Wait for `ACTIVE` status.

---

**Q25. What does --resolve-conflicts OVERWRITE do when updating add-ons?**

**A:**
If you made custom changes to an add-on's config (like editing CoreDNS configmap), `OVERWRITE` replaces those changes with the default new version config.

Options:
- `OVERWRITE` — use new defaults, lose your custom changes
- `PRESERVE` — keep your custom changes (may cause conflicts)
- `NONE` — fail if there is any conflict

For most upgrades, `OVERWRITE` is safe unless you have custom add-on configurations.

---

**Q26. How do you update the Cluster Autoscaler after EKS upgrade?**

**A:**
Cluster Autoscaler version must match Kubernetes minor version.

```bash
# Set image to matching version (e.g., for K8s 1.30 use CA 1.30.x)
kubectl -n kube-system set image \
  deployment.apps/cluster-autoscaler \
  cluster-autoscaler=registry.k8s.io/autoscaling/cluster-autoscaler:v1.30.0
```

Find the right version at:
`https://github.com/kubernetes/autoscaler/releases`

---

## 7. Troubleshooting

---

**Q27. Control plane upgrade is stuck in InProgress for a long time. What do you do?**

**A:**
1. Check update status:
```bash
aws eks describe-update \
  --name my-cluster \
  --update-id <id> \
  --region us-east-1
```

2. Look at the `errors` field in the output for specific reasons

3. Common causes:
   - Not enough free IPs in subnets
   - Security group blocking required ports
   - Service limits hit in the AWS account

4. If stuck beyond 30 minutes, raise an **AWS Support ticket** — you cannot cancel a control plane update manually

---

**Q28. A node is stuck in NotReady after upgrade. How do you troubleshoot?**

**A:**
```bash
# Check node details
kubectl describe node <node-name>

# Check node events
kubectl get events --field-selector involvedObject.name=<node-name>

# SSH into node and check kubelet
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50

# Check if node has correct IAM role attached
aws ec2 describe-instances --instance-ids <instance-id>
```

Common causes:
- Wrong AMI used (not EKS-optimized for target version)
- IAM role missing permissions
- Security group blocking node-to-control-plane communication
- Node bootstrap script failed

---

**Q29. Pods are crashlooping after EKS upgrade. What could be the reason?**

**A:**
Common reasons after upgrade:

| Cause | Fix |
|---|---|
| Deprecated API in pod spec | Update manifest to use new API version |
| Image incompatible with new K8s version | Update the container image |
| Add-on version mismatch (VPC CNI) | Update add-ons |
| ConfigMap or Secret format changed | Review and update configs |
| RBAC permission changed | Check and update roles/rolebindings |

```bash
# Check pod logs
kubectl logs <pod-name> -n <namespace>

# Check events
kubectl describe pod <pod-name> -n <namespace>
```

---

**Q30. How do you check if any deprecated APIs are being used in your cluster?**

**A:**
Three ways:

**1. EKS Upgrade Insights (easiest)**
```bash
aws eks list-insights \
  --cluster-name my-cluster \
  --region us-east-1
```

**2. Kubernetes API audit logs** — check CloudWatch logs for deprecated API calls

**3. Tool — pluto** (open source):
```bash
# Install pluto
brew install FairwindsOps/tap/pluto

# Scan cluster for deprecated APIs
pluto detect-all-in-cluster

# Scan helm releases
pluto detect-helm
```

---

## 8. Best Practices

---

**Q31. What are the best practices you follow for EKS upgrade?**

**A:**
These are the practices I follow:

1. **Test in non-production first** — always upgrade dev/staging before production
2. **One version at a time** — never skip minor versions
3. **Take backup before upgrade** — AWS Backup + manifest export
4. **Check upgrade insights** — fix all issues before starting
5. **Upgrade during low-traffic window** — weekends or off-peak hours
6. **Keep add-ons up to date** — don't let them fall too far behind
7. **Use managed node groups** — they handle drain/cordon automatically
8. **Set up PodDisruptionBudgets** — protect critical applications
9. **Monitor after upgrade** — watch CloudWatch metrics for 24 hours
10. **Document everything** — record versions before and after

---

**Q32. How do you test your applications work on the new Kubernetes version before upgrading production?**

**A:**
I use this approach:

1. **Create a test cluster** on the target version
2. **Deploy the same workloads** using the backup manifests
3. **Run integration tests** and smoke tests
4. **Check for deprecated API errors** in logs
5. **Test critical user journeys** manually

If all tests pass, proceed with production upgrade. This is called a **blue-green cluster upgrade strategy**.

---

**Q33. How often should you upgrade EKS?**

**A:**
- AWS releases a new Kubernetes version on EKS roughly **every 3-4 months**
- Standard support ends after **14 months** from release
- Best practice: stay **no more than 2 versions behind** the latest

Recommended: upgrade **every 6 months** to stay current and avoid being forced to do emergency upgrades when a version reaches end-of-life.

---

**Q34. What is the difference between in-place node upgrade and blue-green node upgrade?**

**A:**

| Method | How it works | Pros | Cons |
|---|---|---|---|
| **In-place** | Drain old node, replace with new node in same node group | Simple, less cost | Brief disruption per node |
| **Blue-Green** | Create brand new node group, migrate pods, delete old group | Zero downtime, clean cutover | Higher cost during migration |

For production clusters with strict SLA, blue-green is safer.

```bash
# Blue-green: create new node group at target version
eksctl create nodegroup \
  --cluster my-cluster \
  --name new-nodegroup \
  --node-ami-family AmazonLinux2 \
  --kubernetes-version 1.30

# Then drain and delete old node group
eksctl delete nodegroup \
  --cluster my-cluster \
  --name old-nodegroup \
  --drain
```

---

## 9. Scenario-Based Questions

---

**Q35. Your team needs to upgrade a production EKS cluster from 1.27 to 1.30. How do you plan it?**

**A:**
Since we can only go one version at a time, I plan 3 separate upgrade windows:

**Window 1:** 1.27 → 1.28
**Window 2:** 1.28 → 1.29
**Window 3:** 1.29 → 1.30

For each window:
1. Read release notes for that version
2. Fix any deprecated API issues
3. Take backup
4. Upgrade control plane
5. Upgrade nodes
6. Upgrade add-ons
7. Run tests and validate
8. Monitor for 24 hours before next window

Total time estimate: 3-4 weeks with proper testing.

---

**Q36. During node upgrade, one of your 3 nodes is stuck draining because of a PodDisruptionBudget. How do you handle it?**

**A:**
First I investigate:
```bash
# See why drain is stuck
kubectl get pdb --all-namespaces

# Check which pods are blocking
kubectl describe pdb <pdb-name> -n <namespace>
```

Options (in order of preference):
1. **Wait** — if it will resolve shortly (e.g., a deployment is rolling)
2. **Scale up the deployment** — so PDB can be satisfied during drain
3. **Temporarily relax the PDB** — reduce `minAvailable` just for the upgrade window
4. **Force drain (last resort)** — only if business accepts the risk:
   ```bash
   kubectl drain node-01 --ignore-daemonsets --force --delete-emptydir-data
   ```

I always prefer option 1 or 2 to avoid any downtime.

---

**Q37. After upgrading EKS, your VPC CNI pods are in CrashLoopBackOff. What do you do?**

**A:**
```bash
# Check VPC CNI pod logs
kubectl logs -n kube-system -l k8s-app=aws-node

# Check events
kubectl describe pod <aws-node-pod> -n kube-system

# Check add-on status
aws eks describe-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --region us-east-1
```

Common fixes:
1. **Update VPC CNI to compatible version** for the new K8s version
2. **Check IAM permissions** — VPC CNI needs `AmazonEKS_CNI_Policy` on node role
3. **Resolve config conflict** — re-run update-addon with `OVERWRITE`

```bash
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --resolve-conflicts OVERWRITE \
  --region us-east-1
```

---

**Q38. Your company wants zero-downtime EKS upgrade for a stateful application with a 3-node cluster. What strategy do you use?**

**A:**
For zero-downtime with stateful apps:

1. **Set up PodDisruptionBudgets** — ensure minimum replicas stay up
```yaml
spec:
  minAvailable: 2   # at least 2 pods always running
```

2. **Use blue-green node upgrade** — create new node group first, migrate, then delete old

3. **For StatefulSets** — use `maxUnavailable: 1` in the update strategy
```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```

4. **For persistent volumes** — ensure EBS volumes are in the same AZ as new nodes

5. **Test failover** — verify the app recovers correctly before starting upgrade

6. **Monitor during upgrade** — watch application metrics in CloudWatch or Datadog

---

**Q39. How do you handle EKS upgrade when you have custom admission webhooks?**

**A:**
Admission webhooks can block upgrades if they use old APIs or have tight TLS configs.

Before upgrade:
```bash
# List all webhooks
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Check their API version and caBundle
kubectl describe mutatingwebhookconfiguration <name>
```

Steps:
1. Make sure webhook server is compatible with new K8s version
2. Check the `failurePolicy` — if set to `Fail`, a broken webhook blocks all pod creation
3. Temporarily set `failurePolicy: Ignore` during upgrade if needed
4. After upgrade, fix and set back to `Fail`

---

**Q40. How do you set up a CI/CD pipeline to automatically test against new EKS versions?**

**A:**
My approach:

1. **GitHub Actions / Jenkins** triggers when AWS announces new EKS version
2. **Spin up a test EKS cluster** on the new version using Terraform or eksctl
3. **Deploy application manifests** from the main branch
4. **Run test suite** — unit, integration, smoke tests
5. **Check for deprecated API warnings** in logs using pluto or audit logs
6. **Notify the team** — Slack alert with results
7. **Destroy test cluster** after tests

This gives early warning of compatibility issues weeks before you need to upgrade production.

```yaml
# Example GitHub Actions step
- name: Create test EKS cluster
  run: |
    eksctl create cluster \
      --name test-upgrade-cluster \
      --version 1.30 \
      --nodes 2 \
      --region us-east-1

- name: Run tests
  run: kubectl apply -f k8s/ && ./run-tests.sh

- name: Cleanup
  if: always()
  run: eksctl delete cluster --name test-upgrade-cluster
```

---

## Quick Reference — Key Commands

```bash
# Check versions
kubectl version --short
kubectl get nodes -o wide

# Check upgrade insights
aws eks list-insights --cluster-name <name> --region <region>

# Backup
aws backup start-backup-job --backup-vault-name <vault> \
  --resource-arn arn:aws:eks:<region>:<acct>:cluster/<name> \
  --iam-role-arn arn:aws:iam::<acct>:role/AWSBackupDefaultServiceRole

# Upgrade control plane
eksctl upgrade cluster --name <name> --version <ver> --approve

# Upgrade nodes
aws eks update-nodegroup-version \
  --cluster-name <name> --nodegroup-name <ng> --region <region>

# Upgrade add-ons
aws eks update-addon --cluster-name <name> \
  --addon-name vpc-cni --resolve-conflicts OVERWRITE --region <region>

# Validate
kubectl get nodes -o wide
kubectl get pods --all-namespaces
kubectl get pods -n kube-system
```

---

*40 Questions | Easy English | 8+ Years Experience Level*
*Generated: 2026-06-29*
