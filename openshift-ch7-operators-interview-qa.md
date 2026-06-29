# OpenShift — Manage Kubernetes Operators
### Chapter 7 | Interview Questions, Answers & Key Points | Easy English | 8+ Years Experience

---

## Table of Contents

1. [Important Concepts to Remember](#1-important-concepts-to-remember)
2. [Operator Pattern — Questions](#2-operator-pattern--questions)
3. [Operator Lifecycle Manager (OLM) — Questions](#3-operator-lifecycle-manager-olm--questions)
4. [OLM Resources — Questions](#4-olm-resources--questions)
5. [Installing Operators via Web Console — Questions](#5-installing-operators-via-web-console--questions)
6. [Installing Operators via CLI — Questions](#6-installing-operators-via-cli--questions)
7. [Operator Updates and Install Plans — Questions](#7-operator-updates-and-install-plans--questions)
8. [Troubleshooting Operators — Questions](#8-troubleshooting-operators--questions)
9. [Hands-On / Practical Questions](#9-hands-on--practical-questions)
10. [Scenario-Based Questions](#10-scenario-based-questions)
11. [Quick Reference Cheatsheet](#11-quick-reference-cheatsheet)

---

## 1. Important Concepts to Remember

> Read these key points first — highest chance of appearing in interviews.

---

### Key Point 1 — The Big Picture: Two Separate Operator Systems

```
OpenShift has TWO separate operator management systems:

1. Cluster Version Operator (CVO)
   → Manages CLUSTER operators (web console, OAuth, DNS, networking)
   → Installed/updated as part of OpenShift platform upgrades
   → You cannot install/remove these yourself

2. Operator Lifecycle Manager (OLM)
   → Manages ADD-ON operators (MetalLB, File Integrity, Compliance, etc.)
   → Installed/updated INDEPENDENTLY from cluster upgrades
   → Admins install, update, and remove these themselves
```

---

### Key Point 2 — The Operator Pattern (Most Important Concept)

```
Traditional approach:
  Admin manually creates → Deployment + Service + PVC + ConfigMap + CronJob

Operator approach:
  Admin creates ONE custom resource (e.g., Database CR)
  Operator watches the CR
  Operator automatically creates → Deployment + Service + PVC + CronJob

Benefits:
  → Reusable automation for complex workloads
  → Maintenance tasks automated (backups, updates, scaling)
  → Domain knowledge baked into the operator
```

---

### Key Point 3 — Key OLM Resources (Must Know All 7)

| Resource | Who Creates It | Purpose |
|---|---|---|
| **CatalogSource** | Admin (or pre-installed) | Points to operator repository |
| **PackageManifest** | OLM (auto) | One per available operator, has install info |
| **OperatorGroup** | Admin | Defines which namespaces operator monitors |
| **Subscription** | Admin | Triggers operator installation |
| **InstallPlan** | OLM (auto) | Represents steps to install/update |
| **ClusterServiceVersion (CSV)** | OLM (auto) | One per operator version, has all operator details |
| **Operator** | OLM (auto) | Stores info about installed operator |

> **Admin only creates 2 things:** OperatorGroup + Subscription (and Namespace if needed)

---

### Key Point 4 — Install Modes

| Mode | Meaning | Common use |
|---|---|---|
| **OwnNamespace** | Operator only monitors its own namespace | Restricted operators |
| **SingleNamespace** | Operator monitors one specific namespace | Targeted operators |
| **MultiNamespace** | Operator monitors multiple specific namespaces | Rare |
| **AllNamespaces** | Operator monitors ALL namespaces | Most common (default) |

> The `openshift-operators` namespace has a pre-existing **global-operators** OperatorGroup → perfect for AllNamespaces mode operators.

---

### Key Point 5 — Automatic vs Manual Updates

| | Automatic | Manual |
|---|---|---|
| **When update applied** | Immediately when available | Only after admin approves InstallPlan |
| **InstallPlan** | Auto-approved | Must be approved with `oc patch` |
| **Good for** | Dev/test environments | Production environments |
| **Risk** | Unexpected breaking changes | Admin controls timing |

---

### Key Point 6 — Two Workload Sets Per Operator

Every installed operator has TWO sets of workloads:

```
Set 1: The OPERATOR workload (created by OLM)
  → e.g., file-integrity-operator deployment
  → This watches for custom resources (FileIntegrity)
  → Runs in the operator namespace

Set 2: The MANAGED workloads (created by the operator)
  → e.g., aide-example-fileintegrity DaemonSet
  → Created when you create a custom resource
  → These do the actual work
```

---

### Key Point 7 — Default Operator Catalogs in OpenShift

| Catalog | Who Supports It |
|---|---|
| **Red Hat** | Red Hat fully packages and supports |
| **Certified** | Independent software vendors (ISVs) |
| **Community** | No official support |
| **Marketplace** | Commercial — paid via Red Hat Marketplace |

---

## 2. Operator Pattern — Questions

---

**Q1. What is the Operator Pattern? Explain it in simple words.**

**A:**
The Operator Pattern is a way to **automate the management of complex applications** on Kubernetes by using custom code.

**Without operator:**
- You manually create a Deployment, Service, PersistentVolumeClaim, ConfigMap, CronJob, etc.
- You manually handle backups, scaling, updates

**With operator:**
- You create ONE simple custom resource (e.g., `kind: Database`)
- The operator watches for this resource
- The operator automatically creates all needed Kubernetes resources
- The operator also handles maintenance tasks (backups, failover, updates)

**Real example:**
```yaml
# You create just this:
apiVersion: postgres.example.com/v1
kind: PostgreSQLCluster
metadata:
  name: my-db
spec:
  replicas: 3
  storageSize: 50Gi
  backupSchedule: "0 2 * * *"
```
The operator sees this and automatically creates a StatefulSet (3 pods), PVCs (3 × 50Gi), Services, and a CronJob for backups.

---

**Q2. What are Custom Resources (CRs) and Custom Resource Definitions (CRDs)?**

**A:**
- **CRD (Custom Resource Definition)** = extends the Kubernetes API with a new resource type
  - Like a blueprint or schema
  - Example: CRD defines what a `FileIntegrity` resource looks like

- **CR (Custom Resource)** = an instance of that new resource type
  - Like an object created from the blueprint
  - Example: You create a `FileIntegrity` CR to start a file integrity check

**Analogy:**
- CRD = class definition in programming
- CR = object/instance of that class

```bash
# See what CRDs an operator adds
oc get crd | grep fileintegrity

# Create a custom resource
oc apply -f fileintegrity.yaml
```

---

**Q3. What is the difference between a Cluster Operator and an Add-on Operator?**

**A:**

| | Cluster Operator | Add-on Operator |
|---|---|---|
| **Purpose** | Provides core OpenShift platform services | Adds optional functionality |
| **Examples** | Web console, OAuth server, DNS, Networking | MetalLB, File Integrity, Compliance |
| **Managed by** | Cluster Version Operator (CVO) | Operator Lifecycle Manager (OLM) |
| **Update cycle** | Updated WITH the cluster (lockstep) | Updated INDEPENDENTLY |
| **Admin control** | Cannot install/remove freely | Admin can install/update/remove |
| **View command** | `oc get clusteroperator` | `oc get csv --all-namespaces` |

---

**Q4. What is the Cluster Version Operator (CVO)?**

**A:**
The CVO is the operator that **installs and updates all core OpenShift platform components** as part of cluster installation and upgrades.

It manages resources of the `ClusterOperator` type. Each cluster operator (authentication, DNS, network, storage, etc.) has a `ClusterOperator` resource showing its health.

```bash
# Check all cluster operators
oc get clusteroperator

# Check status of a specific cluster operator
oc describe clusteroperator authentication

# Expected output columns:
# NAME         VERSION   AVAILABLE   PROGRESSING   DEGRADED
# A healthy operator: True / False / False
```

> **If any ClusterOperator shows DEGRADED=True → the platform service is broken** and should be investigated immediately.

---

## 3. Operator Lifecycle Manager (OLM) — Questions

---

**Q5. What is the Operator Lifecycle Manager (OLM)? What does it do?**

**A:**
OLM is an OpenShift component that helps administrators **install, update, and remove add-on operators**.

OLM provides:
1. **Catalog browsing** — see all available operators via OperatorHub
2. **Installation** — create subscription → OLM handles the rest
3. **Automatic updates** — monitors for new operator versions
4. **Manual update control** — approve or reject updates via install plans
5. **Dependency management** — installs operator dependencies automatically
6. **Lifecycle tracking** — tracks operator versions, CSVs, install plans

OLM is itself an operator (follows the operator pattern).

---

**Q6. What is an Operator Catalog? What catalogs are available by default in OpenShift?**

**A:**
An Operator Catalog is a **container image** that contains information about available operators — their descriptions, versions, and installation requirements.

OLM reads catalog images and creates `PackageManifest` resources for each available operator.

**Default catalogs in OpenShift:**

| Catalog | Trust Level | Example Operators |
|---|---|---|
| **Red Hat** | Fully supported by Red Hat | File Integrity, Compliance, MetalLB |
| **Certified** | Supported by the ISV partner | 3rd party databases, monitoring tools |
| **Community** | No official support | Open source community operators |
| **Marketplace** | Paid commercial operators | Commercial software |

```bash
# View available catalogs
oc get catalogsource -n openshift-marketplace

# View all available operators from all catalogs
oc get packagemanifests
```

---

**Q7. What is an OperatorGroup? Why is it needed?**

**A:**
An OperatorGroup tells the OLM **which namespaces the operator should monitor** for custom resources.

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: file-integrity-operator
  namespace: openshift-file-integrity    # operator lives here
spec:
  targetNamespaces:
  - openshift-file-integrity             # monitors custom resources here
```

**Why it's needed:**
- An operator installed in namespace A might need to watch custom resources in namespaces B, C, D
- OperatorGroup defines the scope of watching
- Every operator namespace must have exactly ONE OperatorGroup

**Pre-existing OperatorGroup:**
- The `openshift-operators` namespace has a `global-operators` OperatorGroup
- It targets ALL namespaces
- If you install an operator in `openshift-operators`, it can watch ALL namespaces automatically

---

## 4. OLM Resources — Questions

---

**Q8. What is a PackageManifest and how do you use it?**

**A:**
A PackageManifest is a resource that the OLM creates automatically for each available operator. It contains all the information you need to install the operator.

```bash
# List all available operators
oc get packagemanifests

# Get details of a specific operator
oc describe packagemanifest file-integrity-operator

# Key information to extract:
# - Catalog Source name and namespace
# - Available channels (e.g., stable, stable-4.14)
# - Default channel
# - Install modes (AllNamespaces, OwnNamespace, etc.)
# - Suggested namespace
# - Current CSV name (e.g., file-integrity-operator.v1.3.3)
```

---

**Q9. What is a Cluster Service Version (CSV)? Why is it important?**

**A:**
A CSV is the **main resource that describes a specific version of an operator**. Every version of an operator has its own CSV.

The CSV contains:
- The operator's deployment spec (how the operator workload runs)
- Custom resource definitions it owns
- Required permissions (RBAC)
- Example custom resources (`alm-examples` annotation)
- Installation instructions
- Version and upgrade path info

```bash
# List installed CSVs
oc get csv --all-namespaces

# Describe a specific CSV
oc describe csv file-integrity-operator.v1.3.3 -n openshift-file-integrity

# Get example custom resources from CSV
oc get csv <csv-name> \
  -o jsonpath='{.metadata.annotations.alm-examples}' | jq

# List CRDs owned by an operator via CSV
oc get csv <csv-name> \
  -o jsonpath="{.spec.customresourcedefinitions.owned[*].name}{'\n'}"
```

> **Phase check:** A successfully installed CSV shows phase `Succeeded`. Any other phase means installation problems.

---

**Q10. What is an InstallPlan? When do you need to manually approve it?**

**A:**
An InstallPlan is a resource that the OLM creates to **represent the steps needed to install or update an operator**.

It lists:
- Which CSVs will be installed
- What CRDs will be created
- Whether it is approved or not

**When manual approval is needed:**
- You set `installPlanApproval: Manual` in the Subscription
- OLM creates InstallPlan but sets `spec.approved: false`
- Admin must approve:

```bash
# Find the install plan
oc describe operator file-integrity-operator
# Look for: Kind: InstallPlan, Name: install-XXXXX

# View install plan details
oc get installplan install-4wsq6 \
  -o jsonpath='{.spec}{"\n"}' \
  -n openshift-file-integrity

# Approve the install plan
oc patch installplan install-4wsq6 \
  --type merge \
  -p '{"spec":{"approved":true}}' \
  -n openshift-file-integrity
```

---

**Q11. What is a Subscription resource? Write an example.**

**A:**
A Subscription is the resource that **tells OLM to install an operator and keep it updated**. It is the main resource that admins create to trigger operator installation.

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: file-integrity-operator
  namespace: openshift-file-integrity    # operator workload namespace
spec:
  channel: "stable"                      # which update channel to follow
  name: file-integrity-operator          # package name from PackageManifest
  source: do280-catalog-cs               # catalog source name
  sourceNamespace: openshift-marketplace # catalog source namespace
  installPlanApproval: Manual            # Automatic or Manual
```

Key fields:
- `channel` → from `oc describe packagemanifest` (e.g., `stable`, `stable-4.14`)
- `name` → PackageManifest name
- `source` → CatalogSource name
- `sourceNamespace` → always `openshift-marketplace`
- `installPlanApproval` → `Automatic` (auto-update) or `Manual` (admin approves)

---

## 5. Installing Operators via Web Console — Questions

---

**Q12. What are the steps to install an operator via the web console?**

**A:**
1. Navigate to **Operators → OperatorHub**
2. Search for the operator by name or category
3. Click the operator to read its description
4. Click **Install** to open the Install Operator wizard
5. Configure:
   - **Update channel** — choose appropriate channel
   - **Installation mode** — All namespaces (default) or specific namespace
   - **Installed namespace** — where operator workload runs
   - **Update approval** — Automatic or Manual
6. Click **Install**
7. Wait for installation to complete
8. Click **View Operator** to confirm

> **Important:** Always read the operator description and documentation BEFORE clicking Install. Some operators require specific namespaces or extra configuration steps.

---

**Q13. What is the difference between "Installation mode" and "Installed namespace" in the operator wizard?**

**A:**

| Option | What it controls |
|---|---|
| **Installation mode** | Which namespaces the operator MONITORS for custom resources |
| **Installed namespace** | Where the operator WORKLOAD (deployment/pod) runs |

These are two separate concerns:

**Example:**
- Installation mode: `All namespaces` → operator watches every namespace for `FileIntegrity` resources
- Installed namespace: `openshift-file-integrity` → the operator's own pod runs here

A user in namespace `my-app` can create a `FileIntegrity` CR, and the operator running in `openshift-file-integrity` will respond to it.

---

**Q14. How do you view the status and update settings of an installed operator?**

**A:**
Navigate to **Operators → Installed Operators → click your operator**

The tabs show:
- **Details tab** — CSV info, conditions (installation progress)
- **YAML tab** — raw CSV resource
- **Subscription tab** — update channel, approval settings, link to InstallPlan
- **Events tab** — events related to the operator
- **Custom resource tabs** — one tab per CRD the operator defines

To change update channel or approval:
- Go to **Subscription tab** → Edit

To approve pending update:
- Go to **Subscription tab** → click the InstallPlan link → Approve

---

## 6. Installing Operators via CLI — Questions

---

**Q15. What are all the steps to install an operator via CLI? Give the complete workflow.**

**A:**

**Step 1: Find the operator**
```bash
oc get packagemanifests
oc describe packagemanifest <operator-name>
# Note: catalog source, channel, suggested namespace, install modes
```

**Step 2: Create namespace (if needed)**
```bash
oc create namespace openshift-file-integrity
# Some operators need specific labels
oc create -f namespace.yaml
```

**Step 3: Create OperatorGroup**
```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: file-integrity-operator
  namespace: openshift-file-integrity
spec:
  targetNamespaces:
  - openshift-file-integrity
```
```bash
oc create -f operator-group.yaml
```

**Step 4: Create Subscription**
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: file-integrity-operator
  namespace: openshift-file-integrity
spec:
  channel: "stable"
  installPlanApproval: Manual
  name: file-integrity-operator
  source: do280-catalog-cs
  sourceNamespace: openshift-marketplace
```
```bash
oc create -f subscription.yaml
```

**Step 5: Approve InstallPlan (if Manual)**
```bash
oc describe operator file-integrity-operator
# Find InstallPlan name

oc patch installplan <install-plan-name> \
  --type merge \
  -p '{"spec":{"approved":true}}' \
  -n openshift-file-integrity
```

**Step 6: Verify installation**
```bash
oc get csv -n openshift-file-integrity
# Wait for Succeeded phase

oc get all -n openshift-file-integrity
# Verify operator pod is Running
```

---

**Q16. When do you need to create an OperatorGroup and when can you skip it?**

**A:**

**Skip OperatorGroup when:**
- Installing operator in the `openshift-operators` namespace
- This namespace already has a `global-operators` OperatorGroup (targets all namespaces)

```bash
# Check if OperatorGroup already exists
oc get operatorgroup -n openshift-operators
# NAME               AGE
# global-operators   5d
```

**Create OperatorGroup when:**
- Installing operator in a new/custom namespace
- Installing operator in any namespace OTHER than `openshift-operators`
- Need to restrict which namespaces the operator monitors

> **Rule:** Each namespace can have ONLY ONE OperatorGroup. If you create a second one, installation fails.

---

**Q17. How do you find the example custom resources for an operator?**

**A:**
Two methods:

**Method 1 — From the CSV (alm-examples annotation):**
```bash
# Get CSV name first
oc get csv -n <namespace>

# Extract examples
oc get csv compliance-operator.v1.4.0 \
  -o jsonpath='{.metadata.annotations.alm-examples}' | jq
```

**Method 2 — Using oc explain:**
```bash
# List CRDs owned by the operator
oc get csv <csv-name> \
  -o jsonpath="{.spec.customresourcedefinitions.owned[*].name}{'\n'}"

# Get field documentation for a specific CRD
oc explain fileintegrity.spec
oc explain fileintegrity.spec.config
```

**Method 3 — Web console:**
- Operators → Installed Operators → click operator
- Custom resource tab → Create button → YAML editor shows example template

---

## 7. Operator Updates and Install Plans — Questions

---

**Q18. What is an Update Channel? How do you choose the right one?**

**A:**
An update channel is like a **release track** for an operator. Operators use channels to group related versions.

Common channel names:
- `stable` — latest stable release
- `stable-4.14` — stable for a specific OpenShift version
- `alpha` — pre-release/experimental features

**How to see available channels:**
```bash
oc describe packagemanifest lvms-operator
# Look for:
# Channels:
#   Name: stable-4.14   ← channel name
#   Current CSV: lvms-operator.v4.14.1  ← latest version in channel
# Default Channel: stable-4.14
```

**Choosing a channel:**
- For production: use `stable` or version-pinned channel (e.g., `stable-4.14`)
- Avoid `alpha` in production — may have breaking changes
- Some operators have only one channel — no choice needed

---

**Q19. What is the difference between Automatic and Manual update approval?**

**A:**

| | Automatic | Manual |
|---|---|---|
| **Behavior** | OLM updates operator immediately when new version available | OLM creates InstallPlan but waits for admin approval |
| **InstallPlan** | Auto-approved | `spec.approved: false` — must patch to `true` |
| **Risk** | Unexpected updates during business hours | Admin controls when updates happen |
| **Use case** | Dev/test clusters | Production clusters |

**How to switch from Automatic to Manual:**
```bash
# Edit the subscription
oc edit subscription file-integrity-operator -n openshift-file-integrity
# Change: installPlanApproval: Automatic → installPlanApproval: Manual
```

**How to approve a pending update:**
```bash
# Find pending install plans
oc get installplan -n openshift-file-integrity
# NAME           CSV                         APPROVAL   APPROVED
# install-abc12  file-integrity-operator.v2  Manual     false

# Approve it
oc patch installplan install-abc12 \
  --type merge -p '{"spec":{"approved":true}}' \
  -n openshift-file-integrity
```

---

**Q20. How do you update an operator?**

**A:**
**For Automatic updates:** Nothing to do — OLM updates automatically.

**For Manual updates:**
1. OLM detects new version → creates InstallPlan with `approved: false`
2. Admin finds the pending InstallPlan:
   ```bash
   oc get installplan -n <namespace>
   # or
   oc describe operator <operator-name>
   # Look for InstallPlanPending condition
   ```
3. Admin reviews the update (read release notes, changelog)
4. Admin approves:
   ```bash
   oc patch installplan <name> \
     --type merge -p '{"spec":{"approved":true}}' \
     -n <namespace>
   ```
5. OLM performs the update, installs new CSV

**To change update channel:**
- Edit the Subscription → change the `channel` field
- New channel may trigger immediate update to latest version in that channel

---

## 8. Troubleshooting Operators — Questions

---

**Q21. An operator is installed but not working. How do you troubleshoot it?**

**A:**
Follow this systematic approach:

**Step 1: Check operator installation status**
```bash
# Check CSV phase (should be Succeeded)
oc get csv -n <operator-namespace>
oc describe csv <csv-name> -n <operator-namespace>
# Look at Conditions section
```

**Step 2: Check subscription and install plan**
```bash
oc describe subscription <name> -n <namespace>
oc get installplan -n <namespace>
# Check if install plan is approved and succeeded
```

**Step 3: Check operator workload**
```bash
# Find operator deployment
oc get deployment -n <operator-namespace>

# Check pod status
oc get pod -n <operator-namespace>

# Check pod logs
oc logs deployment/<operator-name> -n <operator-namespace>
```

**Step 4: Check events**
```bash
oc get events -n <operator-namespace> \
  --sort-by .metadata.creationTimestamp
```

**Step 5: Check custom resource status**
```bash
oc describe <custom-resource-kind> <name> -n <namespace>
# Check Status and Conditions sections
```

---

**Q22. How do you identify which workloads belong to the operator itself vs. workloads the operator created?**

**A:**
An operator has two types of workloads:

**Type 1 — The Operator's own workload (created by OLM):**
- Usually a `Deployment` with the operator name
- Found in the operator's installed namespace
- Listed in `spec.install.spec.deployments` in the CSV

```bash
# Find operator workload from CSV
oc get csv <csv-name> \
  -o jsonpath="{.spec.install.spec.deployments[*].name}{'\n'}"
# Example: file-integrity-operator
```

**Type 2 — Workloads created for custom resources (created by the operator):**
- Created when you create a CR (e.g., `FileIntegrity`)
- Different names — usually include the CR name
- Example: `aide-example-fileintegrity` DaemonSet

```bash
# In File Integrity exercise:
# Operator workload: file-integrity-operator (Deployment)
# CR-managed workload: aide-example-fileintegrity (DaemonSet)
oc get deployment,daemonset -n openshift-file-integrity
```

---

**Q23. How do you uninstall an operator cleanly?**

**A:**
Uninstalling has multiple steps — just deleting the subscription is NOT enough.

**Via Web Console (safest):**
1. Operators → Installed Operators → click operator
2. Actions → Uninstall Operator → confirm
3. Delete the namespace the operator created (if applicable)

**Via CLI:**
```bash
# Step 1: Delete the subscription
oc delete subscription file-integrity-operator \
  -n openshift-file-integrity

# Step 2: Delete the CSV
oc delete csv file-integrity-operator.v1.3.3 \
  -n openshift-file-integrity

# Step 3: Delete custom resources (if any created)
oc delete fileintegrity --all -n openshift-file-integrity

# Step 4: Delete the operator namespace (if created for this operator)
oc delete namespace openshift-file-integrity

# Step 5: Delete CRDs (optional, if no other operators use them)
oc delete crd fileintegrities.fileintegrity.openshift.io
```

> **IMPORTANT:** Always read the operator documentation before uninstalling. Some operators have specific cleanup steps (data migration, removing webhooks, etc.).

---

## 9. Hands-On / Practical Questions

---

**Q24. Walk through the complete CLI installation of the File Integrity operator.**

**A:**
```bash
# 1. Login
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443

# 2. Find the operator
oc get packagemanifests
oc describe packagemanifest file-integrity-operator
# Note: source=do280-catalog-cs, channel=stable, namespace=openshift-file-integrity

# 3. Create namespace with required labels
cat <<EOF | oc create -f -
apiVersion: v1
kind: Namespace
metadata:
  labels:
    openshift.io/cluster-monitoring: "true"
    pod-security.kubernetes.io/enforce: privileged
  name: openshift-file-integrity
EOF

# 4. Create OperatorGroup
cat <<EOF | oc create -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: file-integrity-operator
  namespace: openshift-file-integrity
spec:
  targetNamespaces:
  - openshift-file-integrity
EOF

# 5. Create Subscription (Manual approval for production)
cat <<EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: file-integrity-operator
  namespace: openshift-file-integrity
spec:
  channel: "stable"
  installPlanApproval: Manual
  name: file-integrity-operator
  source: do280-catalog-cs
  sourceNamespace: openshift-marketplace
EOF

# 6. Find and approve InstallPlan
oc describe operator file-integrity-operator
# Find: Kind: InstallPlan, Name: install-XXXXX

oc patch installplan install-4wsq6 \
  --type merge -p '{"spec":{"approved":true}}' \
  -n openshift-file-integrity

# 7. Wait for CSV Succeeded
oc get csv -n openshift-file-integrity -w

# 8. Verify operator pod running
oc get all -n openshift-file-integrity

# 9. Test with a custom resource
oc apply -f worker-fileintegrity.yaml
oc get fileintegritynodestatuses -n openshift-file-integrity
```

---

**Q25. How do you check the health of all cluster operators at once?**

**A:**
```bash
# Quick overview of all cluster operators
oc get clusteroperator

# Columns to check:
# AVAILABLE = True  → operator is working
# PROGRESSING = False → no update in progress
# DEGRADED = False → no problems

# Find any degraded operators
oc get clusteroperator | grep -E "True.*False.*True"
# This pattern: AVAILABLE=True PROGRESSING=False DEGRADED=True

# Get details on a specific cluster operator
oc describe clusteroperator authentication

# Check in web console
# Administration → Cluster Settings → ClusterOperators tab
```

---

**Q26. How do you find all CRDs that an installed operator created?**

**A:**
```bash
# Method 1: From the CSV
oc get csv -n openshift-file-integrity
# Get CSV name from output

oc get csv file-integrity-operator.v1.3.3 \
  -o jsonpath="{.spec.customresourcedefinitions.owned[*].name}{'\n'}"

# Method 2: From the CRD list (filter by operator group)
oc get crd | grep fileintegrity
oc get crd | grep metallb

# Method 3: From package manifest
oc describe packagemanifest file-integrity-operator
# Look for "Customresourcedefinitions" section
```

---

## 10. Scenario-Based Questions

---

**Q27. You create an operator subscription but the operator pod never starts. How do you investigate?**

**A:**
```bash
# Step 1: Check if subscription was created correctly
oc describe subscription file-integrity-operator -n openshift-file-integrity
# Look for: InstallPlanRef and conditions

# Step 2: Check if install plan exists and is approved
oc get installplan -n openshift-file-integrity
# If Manual approval: approved should be True
# If still False: patch to approve it

oc patch installplan <name> \
  --type merge -p '{"spec":{"approved":true}}' \
  -n openshift-file-integrity

# Step 3: Check CSV phase
oc get csv -n openshift-file-integrity
# Phase should be Succeeded, not Failed or Installing

# Step 4: Check if OperatorGroup exists
oc get operatorgroup -n openshift-file-integrity
# Must have exactly ONE OperatorGroup

# Step 5: Check catalog source is healthy
oc get catalogsource -n openshift-marketplace
# Status should show healthy

# Step 6: Check events
oc get events -n openshift-file-integrity \
  --sort-by .metadata.creationTimestamp
```

Common causes:
- InstallPlan not approved (Manual mode)
- Missing OperatorGroup
- Wrong catalog source name in subscription
- Catalog source pod not running
- Namespace doesn't exist

---

**Q28. A developer asks "Can I use the Compliance operator in my namespace?" The operator is installed in the `openshift-compliance` namespace with OwnNamespace mode. What do you tell them?**

**A:**
**No, they cannot.** Here's why:

The operator was installed with **OwnNamespace** mode and the OperatorGroup targets only `openshift-compliance`. This means:
- The operator ONLY watches for CRs in `openshift-compliance`
- CRs created in other namespaces are completely ignored

**Options to fix this:**

**Option A** — Reinstall with AllNamespaces mode:
- Delete current operator
- Create OperatorGroup with no `targetNamespaces` (empty = all namespaces)
- Re-install with AllNamespaces mode

**Option B** — Expand the OperatorGroup:
```bash
oc edit operatorgroup compliance-operator -n openshift-compliance
# Add their namespace to targetNamespaces:
spec:
  targetNamespaces:
  - openshift-compliance
  - developer-namespace
```

**Option C** — Create custom resources in `openshift-compliance`:
- Grant the developer access to `openshift-compliance` namespace
- They create CRs there

> **Key lesson:** Installation mode determines which namespaces the operator can serve. Plan this before installing.

---

**Q29. You need to upgrade an operator in production. It currently has Automatic update approval. How do you safely upgrade?**

**A:**
Automatic updates in production are risky. Here's the safe approach:

**Step 1: Switch to Manual update approval first**
```bash
oc edit subscription <operator-name> -n <namespace>
# Change: installPlanApproval: Automatic → Manual
```

**Step 2: Read the release notes**
- Check operator documentation for breaking changes
- Check if any CRD changes require data migration

**Step 3: Test in a non-production cluster**
- Install the new version in dev/staging
- Run your workloads and verify functionality

**Step 4: Approve the update in production during a maintenance window**
```bash
# Find pending install plan
oc get installplan -n <namespace>

# Review what will be installed
oc get installplan <name> -o yaml -n <namespace>

# Approve during maintenance window
oc patch installplan <name> \
  --type merge -p '{"spec":{"approved":true}}' \
  -n <namespace>
```

**Step 5: Monitor after update**
```bash
oc get csv -n <namespace>    # should reach Succeeded
oc get pod -n <namespace>    # operator pod should restart with new version
oc get events -n <namespace> # check for errors
```

---

**Q30. You installed an operator but the custom resource you create never triggers any action. What do you check?**

**A:**
The operator is not responding to your custom resource. Check in this order:

```bash
# 1. Check operator pod is running
oc get pod -n <operator-namespace>
# Must be Running and Ready

# 2. Check if CR was created correctly
oc describe <cr-kind> <cr-name> -n <namespace>
# Look at Status and Events

# 3. Check if you created the CR in the RIGHT namespace
# Operator groups define which namespaces the operator watches
oc get operatorgroup -n <operator-namespace> -o yaml
# Check spec.targetNamespaces — your CR namespace must be in this list

# 4. Check operator logs for errors
oc logs deployment/<operator-name> -n <operator-namespace>
# Look for errors mentioning your CR

# 5. Check if correct API version was used in the CR
oc api-resources | grep <cr-kind>
# Compare API version with what your YAML uses

# 6. Check if CRD exists
oc get crd | grep <cr-kind>
```

Most common cause: **CR was created in a namespace not targeted by the OperatorGroup**.

---

## 11. Quick Reference Cheatsheet

```bash
# ─── DISCOVER OPERATORS ───────────────────────────────────────
# List catalogs
oc get catalogsource -n openshift-marketplace

# List available operators
oc get packagemanifests

# Get operator installation details
oc describe packagemanifest <operator-name>

# ─── INSTALL OPERATOR (CLI) ───────────────────────────────────
# 1. Create namespace
oc create namespace <namespace>

# 2. Create OperatorGroup
cat <<EOF | oc create -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: <name>
  namespace: <namespace>
spec:
  targetNamespaces:
  - <namespace>
EOF

# 3. Create Subscription
cat <<EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: <operator-name>
  namespace: <namespace>
spec:
  channel: "stable"
  installPlanApproval: Manual
  name: <package-manifest-name>
  source: <catalog-source-name>
  sourceNamespace: openshift-marketplace
EOF

# ─── INSTALLPLAN ──────────────────────────────────────────────
# Find install plan
oc describe operator <operator-name>
# or
oc get installplan -n <namespace>

# Approve install plan
oc patch installplan <name> \
  --type merge -p '{"spec":{"approved":true}}' \
  -n <namespace>

# ─── VERIFY INSTALLATION ──────────────────────────────────────
# Check CSV phase (must be Succeeded)
oc get csv -n <namespace>
oc describe csv <csv-name> -n <namespace>

# Check operator pod
oc get all -n <namespace>
oc logs deployment/<operator-name> -n <namespace>

# ─── CLUSTER OPERATORS (CVO-managed) ──────────────────────────
oc get clusteroperator
oc describe clusteroperator <name>

# ─── OPERATOR DETAILS ─────────────────────────────────────────
# List CRDs owned by operator
oc get csv <csv-name> \
  -o jsonpath="{.spec.customresourcedefinitions.owned[*].name}{'\n'}"

# Get example CRs
oc get csv <csv-name> \
  -o jsonpath='{.metadata.annotations.alm-examples}' | jq

# Describe CR fields
oc explain <cr-kind>.spec

# ─── UNINSTALL ────────────────────────────────────────────────
oc delete subscription <name> -n <namespace>
oc delete csv <csv-name> -n <namespace>
oc delete namespace <namespace>   # if created for this operator
```

---

## Summary Table — Most Important Interview Points

| Topic | Key Point |
|---|---|
| Operator pattern | CR describes what you want → operator makes it happen automatically |
| CVO | Manages cluster operators (web console, DNS, auth) — updated with cluster |
| OLM | Manages add-on operators — installed/updated independently |
| ClusterOperator | `oc get clusteroperator` — DEGRADED=True means platform broken |
| Admin creates | Only OperatorGroup + Subscription (+ namespace if needed) |
| OLM auto-creates | PackageManifest, InstallPlan, CSV, Operator resource |
| CSV | One per operator version — phase must be `Succeeded` |
| InstallPlan | Manual mode = must `oc patch` with `approved:true` to proceed |
| OperatorGroup | Defines which namespaces operator monitors. One per namespace. |
| openshift-operators ns | Has global-operators group → good for AllNamespaces mode operators |
| Two workload sets | Operator's own workload + workloads created FOR custom resources |
| alm-examples | Annotation in CSV containing example custom resources |
| Update channel | Like a release track — `stable`, `stable-4.14`, `alpha` |
| Auto vs Manual approval | Production = Manual. Auto = can update unexpectedly. |
| Uninstall steps | Delete Subscription + CSV + namespace + CRDs (read docs first) |

---

*30 Questions | OpenShift Chapter 7 — Manage Kubernetes Operators*
*Easy English | 8+ Years Experience Level | Generated: 2026-06-29*
