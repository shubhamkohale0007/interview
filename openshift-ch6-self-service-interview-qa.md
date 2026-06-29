# OpenShift — Enable Developer Self-Service
### Chapter 6 | Interview Questions, Answers & Key Points | Easy English | 8+ Years Experience

---

## Table of Contents

1. [Important Concepts to Remember](#1-important-concepts-to-remember)
2. [Resource Quotas — Questions](#2-resource-quotas--questions)
3. [Cluster Resource Quotas — Questions](#3-cluster-resource-quotas--questions)
4. [Limit Ranges — Questions](#4-limit-ranges--questions)
5. [Project Templates — Questions](#5-project-templates--questions)
6. [Self-Provisioner Role — Questions](#6-self-provisioner-role--questions)
7. [Hands-On / Practical Questions](#7-hands-on--practical-questions)
8. [Scenario-Based Questions](#8-scenario-based-questions)
9. [Quick Reference Cheatsheet](#9-quick-reference-cheatsheet)

---

## 1. Important Concepts to Remember

> Read these key points first — they are the most likely interview topics.

---

### Key Point 1 — The Big Picture: Why Self-Service Needs Guardrails

```
Without limits:
  One developer team → requests 8 CPUs → uses all cluster CPU
  → Other teams cannot schedule pods → cluster broken

With limits:
  ResourceQuota on each namespace → each team gets fair share
  LimitRange → every pod gets default limits automatically
  ProjectTemplate → all new projects start with rules built-in
  Self-Provisioner → control WHO can create projects
```

> **The goal:** Let developers create their own projects safely without breaking the cluster for others.

---

### Key Point 2 — Resource Requests vs Resource Limits

| | Resource Request | Resource Limit |
|---|---|---|
| **What it does** | Minimum resources the pod needs | Maximum resources the pod can use |
| **Kubernetes uses it for** | Scheduling decisions (which node to place pod) | Enforcing cap (kill/throttle if exceeded) |
| **If exceeded** | Pod cannot be scheduled | CPU throttled, memory → OOMKilled |
| **Quota type** | `requests.cpu`, `requests.memory` | `limits.cpu`, `limits.memory` |

> **Remember:** Requests = what you reserve. Limits = the maximum cap.

---

### Key Point 3 — ResourceQuota Key Facts

- Lives inside **one namespace** — only affects that namespace
- **Two categories:** Compute quotas (CPU/memory) and Object count quotas
- Once any compute quota is set → **ALL pods in that namespace MUST define the corresponding request/limit**
- Wrong quota name (e.g., `count/deployment` instead of `count/deployments.apps`) → quota is created but has **NO effect** — always verify!
- Status shows: `used` (current) vs `hard` (limit)

---

### Key Point 4 — ClusterResourceQuota Key Facts

- OpenShift-specific (not standard Kubernetes)
- Spans **multiple namespaces** using label or annotation selectors
- Uses `quota.openshift.io/v1` API group
- The quota key is **nested differently** — `spec.quota.hard` (not `spec.hard`)
- Project admins see it via `AppliedClusterResourceQuota` (read-only view)

---

### Key Point 5 — LimitRange Key Facts

- Applies to: Containers, Pods, Images, ImageStreams, PVCs
- **Does NOT affect existing pods** — only new pods created after the LimitRange is set
- LimitRange modifies **pod/container spec**, NOT the deployment spec
- The deployment `resources: {}` stays empty even when LimitRange applies defaults
- Value rules (must follow this order):
  ```
  max ≥ default ≥ defaultRequest ≥ min
  ```

---

### Key Point 6 — Project Template Key Facts

- Default template: creates a **Project** + **RoleBinding** (admin role to requesting user)
- Stored in the **`openshift-config`** namespace
- Applied via **`projects.config.openshift.io/cluster`** resource
- After updating the cluster config → **openshift-apiserver pods restart** (takes a few minutes)
- Template uses `${PROJECT_NAME}`, `${PROJECT_ADMIN_USER}` etc. as variables

---

### Key Point 7 — Self-Provisioner Role Key Facts

- Default: **ALL authenticated OAuth users** can create projects (`system:authenticated:oauth`)
- Role: `self-provisioner` ClusterRole
- Binding: `self-provisioners` ClusterRoleBinding
- This binding has `rbac.authorization.kubernetes.io/autoupdate: true` → **auto-restores on API server restart** unless you set it to `false`
- To disable self-provisioning → patch subjects to `null`
- To restrict → change subjects to a specific group

---

## 2. Resource Quotas — Questions

---

**Q1. What is a ResourceQuota in OpenShift/Kubernetes? Why do we need it?**

**A:**
A ResourceQuota is a Kubernetes object that **limits the total resources** that all workloads in a namespace can use together.

**Why we need it:**
- Without quotas, one team's namespace can request all cluster CPUs, preventing other namespaces from scheduling pods
- Protects clusters from accidental runaway deployments
- Mirrors organizational resource allocation (each team gets a fair share)
- Prevents the backing store (etcd) from being overloaded with too many objects

Example: A developer accidentally scales a deployment to 100 replicas. Without a quota, all cluster CPU is consumed. With a quota (`requests.cpu=4`), the namespace can never request more than 4 CPUs total.

---

**Q2. What are the two types of quotas you can set in a ResourceQuota?**

**A:**

**Type 1 — Compute Resource Quotas:**
```yaml
spec:
  hard:
    limits.cpu: "8"        # total CPU limits across all pods
    limits.memory: 8Gi     # total memory limits
    requests.cpu: "4"      # total CPU requests
    requests.memory: 4Gi   # total memory requests
```

**Type 2 — Object Count Quotas:**
```yaml
spec:
  hard:
    count/pods: "10"                  # max 10 pods
    count/deployments.apps: "5"       # max 5 deployments
    count/services: "10"              # max 10 services
    count/persistentvolumeclaims: "4" # max 4 PVCs
```

> **Syntax rule:**
> - Core group resources: `count/pods`, `count/services`
> - Other groups: `count/deployments.apps`, `count/configmaps`

---

**Q3. What happens when you set a compute quota in a namespace and then create a pod WITHOUT setting resource limits?**

**A:**
The pod **fails to create**. Kubernetes blocks pod creation and shows an error:

```
Error creating: pods "test-abc123" is forbidden: failed quota: example:
must specify limits.cpu for: hello-world-nginx;
limits.memory for: hello-world-nginx;
requests.cpu for: hello-world-nginx;
requests.memory for: hello-world-nginx
```

> **Important rule:** Once ANY compute quota is set → ALL pods in that namespace MUST declare the corresponding limits and requests. This is why LimitRanges (default values) are usually set alongside quotas.

---

**Q4. You created a quota with `count/deployment=1` but it has no effect. Why?**

**A:**
The quota name is wrong. The correct syntax for Deployments (which are in the `apps` API group) is:
```
count/deployments.apps   ✅ correct
count/deployment         ❌ wrong — no effect
```

**How to verify:**
```bash
oc get resourcequota
# Wrong quota shows:  count/deployment: 0/1   ← always 0, never counts
# Correct quota shows: count/deployments.apps: 1/1  ← counts correctly
```

> **Best practice:** Always test a quota with an artificially low value in a test namespace and verify it actually blocks creation before using in production.

---

**Q5. How do you create a ResourceQuota using the command line?**

**A:**
```bash
# Create quota to limit pods
oc create resourcequota example --hard=count/pods=10

# Create quota to limit CPU and memory
oc create resourcequota compute-limits \
  --hard=requests.cpu=4,limits.cpu=8,requests.memory=4Gi,limits.memory=8Gi

# Create quota to limit deployments
oc create resourcequota object-counts \
  --hard=count/deployments.apps=5,count/pods=20

# Check quota status
oc get quota
oc get quota example -o yaml
```

---

**Q6. How do you check if a quota is blocking pod creation?**

**A:**
Two ways:

**Method 1 — Check events:**
```bash
oc get event --sort-by .metadata.creationTimestamp
# Look for: "exceeded quota:" or "failed quota:" messages
```

**Method 2 — Check quota status:**
```bash
oc get quota
# If used == hard, the namespace is at its limit

oc describe quota example
# Shows each resource: used vs hard
```

> **Tricky case:** Creating a deployment appears to succeed (`deployment created`) but pods stay in `Pending`. This is because the deployment creation itself is not blocked — only the pod creation is blocked by the quota. Always check events, not just pod status.

---

## 3. Cluster Resource Quotas — Questions

---

**Q7. What is a ClusterResourceQuota and how is it different from a ResourceQuota?**

**A:**

| | ResourceQuota | ClusterResourceQuota |
|---|---|---|
| **Scope** | Single namespace | Multiple namespaces |
| **API group** | `v1` (core) | `quota.openshift.io/v1` |
| **Who can create** | Any namespace admin | Cluster admin only |
| **Selector** | No selector needed | Must define label/annotation selector |
| **Use case** | Per-team namespace limit | Per-organization or per-group total limit |
| **OpenShift only?** | No — standard Kubernetes | Yes — OpenShift only |

---

**Q8. Write an example ClusterResourceQuota that limits all namespaces with the label `group=dev` to 10 CPUs total.**

**A:**
```yaml
apiVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  name: dev-team-quota
spec:
  quota:              # Note: nested under "quota" key, not directly under "spec"
    hard:
      requests.cpu: "10"
  selector:
    labels:
      matchLabels:
        group: dev    # applies to all namespaces with this label
```

Using CLI:
```bash
oc create clusterresourcequota dev-team-quota \
  --project-label-selector=group=dev \
  --hard=requests.cpu=10
```

---

**Q9. A project admin wants to see which ClusterResourceQuota applies to their namespace. How do they check it?**

**A:**
Project admins may NOT have permission to read `ClusterResourceQuota` directly. OpenShift automatically creates an `AppliedClusterResourceQuota` object in each affected namespace.

```bash
# Check as project admin
oc describe AppliedClusterResourceQuota -n my-namespace

# Output shows:
# Namespace Selector: ["namespace1", "namespace2"]
# Label Selector: group=dev
# Resource    Used    Hard
# --------    ----    ----
# requests.cpu  2250m  10
```

> **Note:** `--all-namespaces` does NOT work with `AppliedClusterResourceQuota`. You must specify a namespace.

---

## 4. Limit Ranges — Questions

---

**Q10. What is a LimitRange? How is it different from a ResourceQuota?**

**A:**

| | LimitRange | ResourceQuota |
|---|---|---|
| **What it limits** | Individual pod/container resources | Total namespace resources |
| **Purpose** | Set defaults and min/max per container | Limit total usage across all pods |
| **Solves** | Forgetting to set limits, accidental huge limits | Runaway namespaces consuming cluster resources |
| **Works on** | Containers, Pods, Images, PVCs | Namespace totals |

**Simple analogy:**
- LimitRange = rules per person ("each employee can spend max $100 per day")
- ResourceQuota = rules per team ("entire team budget is $500 per day")

---

**Q11. What are the five keys you can set in a LimitRange? Explain each.**

**A:**
```yaml
spec:
  limits:
  - type: Container
    default:            # 1. Default LIMIT — applied if container sets no limit
      cpu: 500m
      memory: 512Mi
    defaultRequest:     # 2. Default REQUEST — applied if container sets no request
      cpu: 250m
      memory: 256Mi
    max:                # 3. Maximum — container cannot exceed this
      cpu: "1"
      memory: 1Gi
    min:                # 4. Minimum — container must request at least this
      cpu: 125m
      memory: 128Mi
    maxLimitRequestRatio: # 5. Ratio — limit cannot be more than X times the request
      cpu: "4"          # limit.cpu cannot be > 4 × request.cpu
```

**Value rule (must follow this order):**
```
max ≥ default ≥ defaultRequest ≥ min
```

---

**Q12. Does a LimitRange apply to existing pods? What about new pods?**

**A:**
**NO** — LimitRange does NOT apply to existing/running pods.

- Pods created **before** the LimitRange → NOT affected (keep running without limits)
- Pods created **after** the LimitRange → defaults are applied automatically

> **From the exercise:** When they created a deployment first, then added a LimitRange — the existing pods had no limits. Only after deleting and recreating the deployment did the new pods get the LimitRange defaults.

**Important:** The LimitRange modifies the **pod spec**, NOT the deployment spec. The deployment's `resources: {}` stays empty, but pods created by the deployment have the limits applied.

---

**Q13. Why should you always set `default` and `defaultRequest` when you also set `min` or `max`?**

**A:**
If you only set `min` and `max` but not `default`/`defaultRequest`, Kubernetes **automatically fills them in** by copying from `min` or `max`. This can produce unexpected behavior.

Example: If you set `max.memory: 1Gi` but no default, Kubernetes might set `default.memory: 1Gi`. This means every container gets a 1Gi memory limit by default — even tiny containers that need only 50Mi.

**Best practice:**
```yaml
spec:
  limits:
  - max:
      memory: 1Gi
    default:             # always set this explicitly
      memory: 512Mi
    defaultRequest:      # always set this explicitly
      memory: 256Mi
    min:
      memory: 128Mi
    type: Container
```

---

**Q14. A pod fails to create with: "maximum memory usage per Container is 1Gi, but limit is 2Gi". What caused this and how do you fix it?**

**A:**
The pod's container declares a memory limit of `2Gi`, but the LimitRange has `max.memory: 1Gi`. The pod violates the maximum.

```bash
# Check the LimitRange
oc describe limitrange -n my-namespace

# Check the pod/deployment spec
oc describe deployment my-deployment | grep -A5 resources
```

**Fix options:**
1. **Reduce the container's memory limit** to 1Gi or less in the deployment spec
2. **Increase the LimitRange max** (if the workload genuinely needs more)
   ```bash
   oc edit limitrange max-memory -n my-namespace
   ```

---

## 5. Project Templates — Questions

---

**Q15. What is a Project Template in OpenShift? Why is it useful?**

**A:**
A Project Template is an **OpenShift Template resource** that defines what gets created automatically whenever a new project (namespace) is requested.

**By default, a new project creates:**
1. The Project/Namespace itself
2. A RoleBinding (granting the requesting user admin role)

**With a custom template, you can also auto-create:**
- ResourceQuotas (every new project has a resource limit)
- LimitRanges (every new project has default container limits)
- NetworkPolicies (every new project has network isolation rules)
- Custom RoleBindings (e.g., grant admin to a group, not just the requesting user)

> **Why useful:** Enforces organizational standards on ALL new projects automatically. No project can escape quotas or limits.

---

**Q16. Where is the project template stored? How do you enable it?**

**A:**
**Step 1: Create the template in the `openshift-config` namespace:**
```bash
oc create -f template.yaml -n openshift-config
```

**Step 2: Tell OpenShift to use the template:**
```bash
oc edit projects.config.openshift.io cluster
```

Add to the spec:
```yaml
spec:
  projectRequestTemplate:
    name: project-request   # must match the template name
```

**Step 3: Wait for API server to restart:**
```bash
watch oc get pod -n openshift-apiserver
# Wait until new pods are Running
```

---

**Q17. What variables are available in a project template?**

**A:**

| Variable | Value |
|---|---|
| `${PROJECT_NAME}` | The name the user chose for the project |
| `${PROJECT_DISPLAYNAME}` | The display name (optional) |
| `${PROJECT_DESCRIPTION}` | The description (optional) |
| `${PROJECT_ADMIN_USER}` | The user who requested the project |
| `${PROJECT_REQUESTING_USER}` | Same as ADMIN_USER (requesting user) |

**Example use in template:**
```yaml
metadata:
  name: ${PROJECT_NAME}       # project name becomes namespace name
  namespace: ${PROJECT_NAME}  # use this in all nested resources
```

---

**Q18. What is the command to generate a starting project template?**

**A:**
```bash
# Generate the default bootstrap template
oc adm create-bootstrap-project-template -o yaml > template.yaml

# This creates a template with:
# 1. A Project resource
# 2. A RoleBinding (admin role for requesting user)
# You then edit this file to add quotas, limit ranges, etc.
```

> **Best practice workflow:**
> 1. Create a test namespace
> 2. Create and test your quotas/limit ranges there
> 3. Export them: `oc get limitrange,resourcequota -n test-ns -o yaml`
> 4. Add them to the template, replacing `namespace: test-ns` with `namespace: ${PROJECT_NAME}`
> 5. Remove `creationTimestamp`, `resourceVersion`, `uid` fields
> 6. Create template in `openshift-config` and activate it

---

**Q19. In the project template's RoleBinding, the exercise changed the subject from a User to a Group. What is the difference and why?**

**A:**

**Default (User):**
```yaml
subjects:
- kind: User
  name: ${PROJECT_ADMIN_USER}   # only the creator gets admin
```
→ Only the person who created the project gets admin access.

**Changed to Group:**
```yaml
subjects:
- kind: Group
  name: provisioners            # entire group gets admin
```
→ ALL members of the `provisioners` group get admin access to every new project.

**Why change it:** In the exercise, they wanted `provisioner1` and `provisioner2` to both have access to projects created by either of them. By granting the group admin rights, both users automatically have access to all projects created by anyone in the group.

---

## 6. Self-Provisioner Role — Questions

---

**Q20. What is the self-provisioner role and what does it control?**

**A:**
The `self-provisioner` ClusterRole allows users to **create their own projects (namespaces)** via the OpenShift project request API.

By default:
```
system:authenticated:oauth group → has self-provisioner role
= ANY logged-in OpenShift user can create projects
```

When you remove this binding, users see:
```
You may not request a new project via this API.
```

> **Important distinction:** Users with direct namespace permissions can still create namespaces using `kubectl create namespace` — this bypasses the project template. Self-provisioner only controls access via the OpenShift project request API.

---

**Q21. How do you disable project self-provisioning for all users?**

**A:**
```bash
# Step 1: Disable auto-restore (important! otherwise reverts on restart)
oc annotate clusterrolebinding/self-provisioners \
  --overwrite rbac.authorization.kubernetes.io/autoupdate=false

# Step 2: Remove all subjects (disables for everyone)
oc patch clusterrolebinding.rbac self-provisioners \
  -p '{"subjects": null}'

# Verify
oc describe clusterrolebinding self-provisioners
# Subjects section should be empty
```

---

**Q22. How do you restrict project creation to only members of a specific group?**

**A:**
```bash
# Method 1: Use oc edit
oc edit clusterrolebinding self-provisioners
```
Change subjects from:
```yaml
subjects:
- kind: Group
  name: system:authenticated:oauth   # all users
```
To:
```yaml
subjects:
- kind: Group
  name: provisioners   # only provisioners group
```

**Method 2: Use oc patch**
```bash
oc patch clusterrolebinding self-provisioners \
  -p '{"subjects":[{"apiGroup":"rbac.authorization.k8s.io","kind":"Group","name":"provisioners"}]}'
```

---

**Q23. Why is the `rbac.authorization.kubernetes.io/autoupdate` annotation important?**

**A:**
This annotation is on the `self-provisioners` ClusterRoleBinding.

When set to `true` (default): Every time the **API server restarts**, Kubernetes **automatically restores** the original binding (all authenticated users can create projects).

This means:
- If you remove self-provisioning without setting this to `false`
- After any API server restart (upgrades, node restarts)
- The change is **reverted** automatically

**To make changes permanent:**
```bash
# First: set autoupdate to false to prevent auto-restore
oc annotate clusterrolebinding/self-provisioners \
  --overwrite rbac.authorization.kubernetes.io/autoupdate=false

# Then make your subject changes
oc edit clusterrolebinding self-provisioners
```

> **This is a common exam trap question** — many people change the binding but forget this annotation, and the change disappears after a restart.

---

## 7. Hands-On / Practical Questions

---

**Q24. Write complete commands to set up a namespace with a quota of 4 CPUs and 4Gi memory, and a LimitRange with default 250m CPU and 256Mi memory.**

**A:**
```bash
# Step 1: Create the namespace
oc new-project my-team

# Step 2: Create ResourceQuota
oc create resourcequota team-quota \
  --hard=requests.cpu=4,limits.cpu=8,requests.memory=4Gi,limits.memory=8Gi \
  -n my-team

# Step 3: Create LimitRange (YAML approach)
cat <<EOF | oc apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: team-limits
  namespace: my-team
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    max:
      cpu: "2"
      memory: 2Gi
    min:
      cpu: 100m
      memory: 128Mi
EOF

# Step 4: Verify
oc get quota,limitrange -n my-team
```

---

**Q25. Show the complete YAML structure of a project template that includes a quota and limit range.**

**A:**
```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: project-request
objects:
# 1. The Project itself
- apiVersion: project.openshift.io/v1
  kind: Project
  metadata:
    annotations:
      openshift.io/description: ${PROJECT_DESCRIPTION}
      openshift.io/display-name: ${PROJECT_DISPLAYNAME}
      openshift.io/requester: ${PROJECT_REQUESTING_USER}
    name: ${PROJECT_NAME}
  spec: {}
  status: {}

# 2. RoleBinding — give the provisioners group admin access
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: admin
    namespace: ${PROJECT_NAME}
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: admin
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: provisioners

# 3. LimitRange — max 1Gi memory per container
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: max-memory
    namespace: ${PROJECT_NAME}
  spec:
    limits:
    - type: Container
      default:
        memory: 1Gi
      defaultRequest:
        memory: 1Gi
      max:
        memory: 1Gi

# 4. ResourceQuota — limit total namespace resources
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: team-quota
    namespace: ${PROJECT_NAME}
  spec:
    hard:
      requests.cpu: "4"
      requests.memory: 4Gi
      limits.cpu: "8"
      limits.memory: 8Gi

# Template variables
parameters:
- name: PROJECT_NAME
- name: PROJECT_DISPLAYNAME
- name: PROJECT_DESCRIPTION
- name: PROJECT_ADMIN_USER
- name: PROJECT_REQUESTING_USER
```

Apply it:
```bash
oc create -f template.yaml -n openshift-config
oc edit projects.config.openshift.io cluster
# Add: spec.projectRequestTemplate.name: project-request
```

---

**Q26. How do you check the resource usage of a node, and how do you find what resources are already requested?**

**A:**
```bash
# Check actual CPU/memory usage
oc adm top node

# Check requested vs allocatable resources
oc describe node/master01

# Key sections in the output:
# Capacity:        → total hardware
# Allocatable:     → usable (capacity minus system reserved)
# Allocated resources:
#   Resource    Requests    Limits
#   cpu         4627m (84%)  0 (0%)
#   memory      12102Mi (81%) 0 (0%)
```

> **Key insight from exercise:** Node CPU usage was only 14% (actual) but 84% of CPU was REQUESTED. These are different. Kubernetes uses requests for scheduling — even if pods are not actually using the CPU, if they requested it, that capacity is reserved.

---

## 8. Scenario-Based Questions

---

**Q27. A developer complains: "I created a deployment but it shows 0/1 ready and the pod is in Pending status. There are no errors." What do you check?**

**A:**
Most likely cause: **ResourceQuota is blocking pod creation** (or insufficient cluster resources).

```bash
# Step 1: Check events — always check events first
oc get event --sort-by .metadata.creationTimestamp -n <namespace>
# Look for "exceeded quota:" or "Insufficient cpu/memory"

# Step 2: Check quota status
oc get quota -n <namespace>
# If used == hard → quota is exhausted

# Step 3: Check node resources
oc adm top node
oc describe node/<node-name>
# Check "Allocated resources" section

# Step 4: Check if pod has resource requests set
oc describe deployment <name> -n <namespace>
# Look for resources: section in container spec
```

Possible fixes:
- Delete unused pods/deployments in that namespace
- Increase the quota (if justified)
- Scale down other deployments to free up resources

---

**Q28. Your team sets up a ResourceQuota but it has no effect — developers can still create unlimited pods. What went wrong?**

**A:**
The most common causes:

**Cause 1 — Wrong quota syntax:**
```bash
oc get resourcequota
# If "used" always shows 0 regardless of how many pods exist → wrong name
# Wrong:  count/pod: 0/10
# Correct: count/pods: 5/10
```

**Cause 2 — Quota applied to wrong namespace:**
```bash
oc get quota --all-namespaces
# Confirm it is in the right namespace
```

**Cause 3 — Compute quota set but pods don't define limits:**
```bash
# This creates a different problem:
# pods fail to create (not "no effect")
# Check events for "must specify limits.cpu" errors
```

**How to test a quota is working:**
```bash
# Set quota artificially low (e.g., count/pods=1)
# Try to create 2 pods
# If second pod fails → quota is working
# If second pod succeeds → quota name is wrong
```

---

**Q29. You want ALL new projects in the cluster to automatically have a 4Gi memory quota and a 512Mi default container memory limit. How do you implement this?**

**A:**
Use a **project template** with both a ResourceQuota and LimitRange built in.

**Step 1:** Create and test resources in a test namespace:
```bash
oc create namespace template-test
# Create and test your quota and limitrange here
```

**Step 2:** Generate base template:
```bash
oc adm create-bootstrap-project-template -o yaml > template.yaml
```

**Step 3:** Add ResourceQuota and LimitRange to the template's objects section with `namespace: ${PROJECT_NAME}`

**Step 4:** Create template in openshift-config:
```bash
oc create -f template.yaml -n openshift-config
```

**Step 5:** Activate it:
```bash
oc edit projects.config.openshift.io cluster
# Add: spec.projectRequestTemplate.name: project-request
```

**Step 6:** Wait for API server rollout:
```bash
watch oc get pod -n openshift-apiserver
```

**Step 7:** Test by creating a new project and verifying quota/limitrange exist:
```bash
oc new-project test-verify
oc get quota,limitrange -n test-verify
```

---

**Q30. A senior developer changed the self-provisioners binding to restrict project creation. After the next cluster upgrade (which restarts the API server), all users can create projects again. Why did this happen and how do you fix it permanently?**

**A:**
**Why it happened:** The `self-provisioners` ClusterRoleBinding has the annotation:
```
rbac.authorization.kubernetes.io/autoupdate: true
```
This causes the binding to **auto-restore to its original state** whenever the API server restarts (which happens during upgrades, node reboots, etc.).

**Fix (make it permanent):**
```bash
# Step 1: Disable auto-restore FIRST
oc annotate clusterrolebinding/self-provisioners \
  --overwrite rbac.authorization.kubernetes.io/autoupdate=false

# Step 2: Then make your subject changes
oc edit clusterrolebinding self-provisioners
# Change subjects to your specific group

# Step 3: Verify the annotation is set
oc describe clusterrolebinding self-provisioners | grep autoupdate
# Should show: autoupdate=false
```

After setting `autoupdate=false`, the change survives API server restarts.

---

**Q31. Two teams share a cluster. Team A has 3 namespaces, Team B has 2 namespaces. You want Team A to use no more than 10 CPUs TOTAL across all their namespaces. How do you do this?**

**A:**
Use a **ClusterResourceQuota** with a label selector targeting Team A's namespaces.

**Step 1:** Label all Team A namespaces:
```bash
oc label namespace team-a-ns1 team=a
oc label namespace team-a-ns2 team=a
oc label namespace team-a-ns3 team=a
```

**Step 2:** Create ClusterResourceQuota:
```bash
oc create clusterresourcequota team-a-quota \
  --project-label-selector=team=a \
  --hard=requests.cpu=10
```

Or as YAML:
```yaml
apiVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  name: team-a-quota
spec:
  quota:
    hard:
      requests.cpu: "10"
  selector:
    labels:
      matchLabels:
        team: a
```

**Step 3:** Verify total usage:
```bash
oc describe clusterresourcequota team-a-quota
# Shows per-namespace usage + total
```

Team A's total CPU requests across all 3 namespaces cannot exceed 10 CPUs.

---

## 9. Quick Reference Cheatsheet

```bash
# ─── RESOURCE QUOTAS ──────────────────────────────────────────
# Create quota
oc create resourcequota <name> --hard=requests.cpu=4,limits.memory=8Gi

# View quotas
oc get quota
oc describe quota <name>

# Check quota blocking events
oc get event --sort-by .metadata.creationTimestamp | grep -i quota

# ─── CLUSTER RESOURCE QUOTAS ──────────────────────────────────
# Create cluster quota
oc create clusterresourcequota <name> \
  --project-label-selector=<key>=<value> \
  --hard=requests.cpu=10

# View as project admin
oc describe AppliedClusterResourceQuota -n <namespace>

# ─── LIMIT RANGES ─────────────────────────────────────────────
# View limit ranges
oc get limitrange
oc describe limitrange <name>

# Set resources on a deployment
oc set resources deployment <name> --limits=cpu=500m,memory=512Mi
oc set resources deployment <name> --requests=cpu=250m,memory=256Mi

# ─── PROJECT TEMPLATE ─────────────────────────────────────────
# Generate base template
oc adm create-bootstrap-project-template -o yaml > template.yaml

# Create template
oc create -f template.yaml -n openshift-config

# Activate template
oc edit projects.config.openshift.io cluster
# Add: spec.projectRequestTemplate.name: project-request

# Wait for API server rollout
watch oc get pod -n openshift-apiserver

# ─── SELF-PROVISIONER ─────────────────────────────────────────
# View current binding
oc describe clusterrolebinding self-provisioners

# Disable autoupdate (make changes permanent)
oc annotate clusterrolebinding/self-provisioners \
  --overwrite rbac.authorization.kubernetes.io/autoupdate=false

# Disable project creation for all
oc patch clusterrolebinding.rbac self-provisioners \
  -p '{"subjects": null}'

# Restrict to a group
oc edit clusterrolebinding self-provisioners
# Change subjects name to your group name

# ─── NODE RESOURCE CHECK ──────────────────────────────────────
oc adm top node
oc describe node/<node-name>   # check Allocated resources section
```

---

## Summary Table — Most Important Interview Points

| Topic | Key Point |
|---|---|
| ResourceQuota | Lives in ONE namespace. Limits total compute + object counts. |
| Compute quota rule | Once set → ALL pods MUST define matching limits/requests |
| Wrong quota name | `count/deployment` vs `count/deployments.apps` — wrong name = silent failure |
| ClusterResourceQuota | OpenShift-only. Spans multiple namespaces via label selector. |
| `spec.quota.hard` | ClusterResourceQuota nests `hard` under `quota` key (not directly under `spec`) |
| AppliedClusterResourceQuota | What project admins see — read-only view of cluster quotas on their namespace |
| LimitRange | Sets default/min/max per container. Does NOT affect existing pods. |
| LimitRange rule | max ≥ default ≥ defaultRequest ≥ min |
| LimitRange + Deployment | LimitRange modifies pod spec, NOT deployment spec (`resources: {}` stays empty) |
| Project Template | Stored in `openshift-config`. Applied via `projects.config.openshift.io/cluster`. |
| Template activation | Causes openshift-apiserver pods to restart — wait for rollout to complete |
| Template variables | `${PROJECT_NAME}`, `${PROJECT_ADMIN_USER}`, etc. |
| self-provisioner | Default: all OAuth users. ClusterRoleBinding named `self-provisioners`. |
| autoupdate annotation | `true` = changes revert on API server restart. Set to `false` to make permanent. |

---

*31 Questions | OpenShift Chapter 6 — Enable Developer Self-Service*
*Easy English | 8+ Years Experience Level | Generated: 2026-06-29*
