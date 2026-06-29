# OpenShift Cluster Upgrade Runbook

**Version:** 1.0  
**Last Updated:** 2026-06-29  
**Applies To:** OpenShift Container Platform 4.x  

---

## Table of Contents

1. [Overview](#overview)
2. [Roles & Responsibilities](#roles--responsibilities)
3. [Pre-Upgrade Checklist](#pre-upgrade-checklist)
4. [Upgrade Procedure](#upgrade-procedure)
5. [Post-Upgrade Checklist](#post-upgrade-checklist)
6. [Rollback Procedure](#rollback-procedure)
7. [Troubleshooting](#troubleshooting)

---

## Overview

This runbook documents the end-to-end procedure for upgrading an OpenShift Container Platform (OCP) 4.x cluster using the Cluster Version Operator (CVO). Follow every step in order and do not skip sections.

| Field | Details |
|---|---|
| Upgrade Type | Minor (e.g., 4.14 → 4.15) or Patch (e.g., 4.14.5 → 4.14.8) |
| Maintenance Window | `[INSERT DATE/TIME + TIMEZONE]` |
| Target Version | `[INSERT TARGET VERSION]` |
| Current Version | `[INSERT CURRENT VERSION]` |
| Cluster Name | `[INSERT CLUSTER NAME]` |
| API Endpoint | `[INSERT API URL]` |

---

## Roles & Responsibilities

| Role | Responsibility |
|---|---|
| Cluster Admin | Executes all `oc` commands, monitors upgrade |
| Application Owner | Validates workloads post-upgrade |
| Storage Admin | Verifies PV/PVC health before and after |
| Network Admin | Verifies ingress/egress, SDN health |
| On-Call Engineer | Available during window for escalation |

---

## Pre-Upgrade Checklist

Complete **all** checks before starting the upgrade. Record the result (Pass / Fail / N/A) and the timestamp.

### 1. Access & Authentication

```bash
# Log in to the cluster
oc login --token=<token> --server=https://<api-url>:6443

# Confirm you have cluster-admin
oc whoami
oc auth can-i '*' '*' --all-namespaces
```

- [ ] Logged in successfully as cluster-admin
- [ ] `kubeconfig` backed up to a secure location

---

### 2. Cluster Version & Update Path

```bash
# Check current cluster version
oc get clusterversion

# View available updates on the current channel
oc adm upgrade

# Confirm the target version is in the available list
oc get clusterversion -o jsonpath='{.items[0].status.availableUpdates}' | python3 -m json.tool
```

- [ ] Current version confirmed
- [ ] Target version appears in `availableUpdates`
- [ ] Update channel is correct (e.g., `stable-4.15`, `eus-4.14`)

```bash
# Check/set the update channel if needed
oc patch clusterversion version --type='merge' -p '{"spec":{"channel":"stable-4.15"}}'
```

---

### 3. Cluster Operator Health

All Cluster Operators must be `Available=True`, `Progressing=False`, `Degraded=False` before proceeding.

```bash
oc get clusteroperators
oc get clusteroperators | grep -v "True.*False.*False"
```

- [ ] All operators are **Available**
- [ ] No operator is **Progressing**
- [ ] No operator is **Degraded**

If any operator is degraded, investigate before proceeding:

```bash
oc describe clusteroperator <operator-name>
```

---

### 4. Node Health

```bash
# All nodes must be Ready
oc get nodes
oc get nodes | grep -v " Ready"

# Check for any node conditions (MemoryPressure, DiskPressure, PIDPressure)
oc describe nodes | grep -A5 "Conditions:"

# Check node resource utilization
oc adm top nodes
```

- [ ] All nodes are in `Ready` state
- [ ] No nodes have `MemoryPressure`, `DiskPressure`, or `PIDPressure`
- [ ] Adequate CPU and memory headroom exists on all nodes
- [ ] No nodes are `SchedulingDisabled` (cordoned) unexpectedly

---

### 5. etcd Health

```bash
# Check etcd cluster health
oc -n openshift-etcd get pods -l app=etcd

# Run etcdctl health check from an etcd pod
ETCD_POD=$(oc -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}')
oc -n openshift-etcd exec -it $ETCD_POD -- etcdctl endpoint health \
  --cluster \
  --cacert=/etc/kubernetes/static-pod-resources/etcd-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-serving/etcd-serving-*.crt \
  --key=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-serving/etcd-serving-*.key

# Check etcd member list
oc -n openshift-etcd exec -it $ETCD_POD -- etcdctl member list \
  --cacert=/etc/kubernetes/static-pod-resources/etcd-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-serving/etcd-serving-*.crt \
  --key=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-serving/etcd-serving-*.key
```

- [ ] All etcd members are healthy
- [ ] etcd has quorum (all 3 members present)
- [ ] No etcd alarms active

---

### 6. etcd Backup (MANDATORY)

> **This step is mandatory. Do not proceed without a current etcd backup.**

```bash
# SSH into a control plane node and run the backup script
ssh core@<control-plane-node-ip>
sudo /usr/local/bin/cluster-backup.sh /home/core/assets/backup

# Verify backup files exist
ls -lh /home/core/assets/backup/
# Expected: snapshot_<date>.db  static_kuberesources_<date>.tar.gz
```

- [ ] etcd snapshot created (`snapshot_<datetime>.db`)
- [ ] Static resources backed up (`static_kuberesources_<datetime>.tar.gz`)
- [ ] Backup files copied to off-cluster storage

---

### 7. Persistent Volume & Storage Health

```bash
# Check all PVs are Bound (no Failed/Released with data)
oc get pv --all-namespaces | grep -v Bound

# Check all PVCs
oc get pvc --all-namespaces | grep -v Bound

# Check storage operators
oc get clusteroperator storage
oc get clusteroperator cluster-storage-operator
```

- [ ] All PVCs in `Bound` state
- [ ] No PVs in `Failed` or unexpected state
- [ ] Storage operator is healthy

---

### 8. Certificate Health

```bash
# Check certificates managed by the cluster
oc -n openshift-kube-apiserver-operator get secrets \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.auth\.openshift\.io/certificate-not-after}{"\n"}{end}' \
  | sort -k2

# Check for expiring certificates (within 30 days)
oc get certificatesigningrequests
```

- [ ] No certificates expiring within 30 days
- [ ] No pending CSRs from nodes

---

### 9. Pending CSRs

```bash
oc get csr
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}'
```

- [ ] No unexpected pending CSRs
- [ ] All legitimate pending CSRs approved if required

---

### 10. Monitoring & Alerting

```bash
# Check Prometheus/Alertmanager are healthy
oc -n openshift-monitoring get pods
oc -n openshift-monitoring get pods | grep -v Running | grep -v Completed

# List any firing critical alerts
oc -n openshift-monitoring exec -it alertmanager-main-0 -- \
  amtool alert query --alertmanager.url=http://localhost:9093 severity=critical
```

- [ ] All monitoring pods are `Running`
- [ ] No critical alerts firing (or all are acknowledged and understood)
- [ ] Monitoring dashboards accessible

---

### 11. Workload & Application Health

```bash
# Check for pods not Running/Completed across all namespaces
oc get pods --all-namespaces | grep -Ev "(Running|Completed|Succeeded)"

# Check for deployment rollout issues
oc get deployments --all-namespaces | grep -v "1/1\|2/2\|3/3"

# Check PodDisruptionBudgets
oc get pdb --all-namespaces
```

- [ ] No pods in `CrashLoopBackOff`, `Error`, `OOMKilled`, or `Pending` state
- [ ] All deployments at desired replica count
- [ ] PodDisruptionBudgets reviewed — no blocker for node drains

---

### 12. Image Registry & Pull Secrets

```bash
# Verify image registry is accessible
oc get configs.imageregistry.operator.openshift.io cluster
oc -n openshift-image-registry get pods

# Verify pull secret
oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | python3 -m json.tool
```

- [ ] Image registry operator is healthy
- [ ] Pull secret is valid and not expired
- [ ] `registry.redhat.io` and `quay.io` are reachable from nodes

---

### 13. Network Health

```bash
# Check network operator
oc get clusteroperator network

# Verify SDN/OVN pods
oc -n openshift-sdn get pods         # for SDN clusters
oc -n openshift-ovn-kubernetes get pods  # for OVN clusters

# Check ingress
oc get clusteroperator ingress
oc -n openshift-ingress get pods
```

- [ ] Network operator is healthy
- [ ] All SDN/OVN pods are `Running`
- [ ] Ingress controllers are healthy

---

### 14. Capacity Planning

```bash
# Ensure nodes have headroom for control plane upgrades
# Control plane nodes: recommended >= 20% free CPU and memory
oc adm top nodes --selector='node-role.kubernetes.io/master'
oc adm top nodes --selector='node-role.kubernetes.io/worker'
```

- [ ] Control plane nodes have sufficient free resources
- [ ] Worker nodes have sufficient free resources for pod rescheduling during drain

---

### 15. Stakeholder Notification

- [ ] Maintenance window communicated to application teams
- [ ] On-call schedule confirmed for upgrade window
- [ ] Rollback decision authority identified
- [ ] Upgrade ticket / change record created: `[INSERT CHANGE ID]`

---

### Pre-Upgrade Sign-Off

| Name | Role | Signature | Date/Time |
|---|---|---|---|
| | Cluster Admin | | |
| | Application Owner | | |
| | Change Manager | | |

---

## Upgrade Procedure

> **Important:** Monitor the upgrade continuously. Do not leave the terminal unattended.

### Step 1 — Confirm Maintenance Window is Open

```bash
date
# Confirm time matches the approved maintenance window
```

### Step 2 — Initiate the Upgrade

**Option A — Upgrade to latest available version on channel:**

```bash
oc adm upgrade
```

**Option B — Upgrade to a specific version:**

```bash
# Replace <target-version> with the exact version, e.g., 4.15.8
oc adm upgrade --to=<target-version>
```

**Option C — Force upgrade to a specific image (use only when directed by Red Hat Support):**

```bash
oc adm upgrade --force --to-image=<release-image-pullspec>
```

---

### Step 3 — Monitor the Upgrade Progress

Open a separate terminal and watch continuously:

```bash
# Watch cluster version object
watch -n 30 'oc get clusterversion'

# Watch all cluster operators
watch -n 30 'oc get clusteroperators'

# Follow CVO logs in real time
oc logs -n openshift-cluster-version \
  $(oc get pods -n openshift-cluster-version -o name) -f

# Watch all nodes
watch -n 30 'oc get nodes'
```

The upgrade progresses in phases:
1. CVO updates cluster-wide config and operators
2. Control plane nodes are updated one at a time (cordon → drain → update → uncordon)
3. Worker nodes are updated via MachineConfigPool (in batches per `maxUnavailable`)

---

### Step 4 — Control Plane Node Upgrade Monitoring

```bash
# Watch Machine Config Pool status for masters
oc get mcp master -w

# Watch control plane node status
oc get nodes -l node-role.kubernetes.io/master -w
```

Expected MCP progression: `Updated=False, Updating=True` → `Updated=True, Updating=False`

- [ ] master MCP reaches `Updated=True`
- [ ] All control plane nodes are `Ready` after update

---

### Step 5 — Worker Node Upgrade Monitoring

```bash
# Watch worker MCP
oc get mcp worker -w

# Check how many workers are updating at once (default maxUnavailable=1)
oc get mcp worker -o jsonpath='{.spec.maxUnavailable}'
```

> **Note:** If you need to speed up worker node upgrades, you can increase `maxUnavailable` (ensure workloads can tolerate this):

```bash
oc patch mcp worker --type='merge' -p '{"spec":{"maxUnavailable":2}}'
```

> Reset after upgrade completes:

```bash
oc patch mcp worker --type='merge' -p '{"spec":{"maxUnavailable":1}}'
```

- [ ] All worker nodes updated
- [ ] worker MCP reaches `Updated=True`
- [ ] All worker nodes are `Ready`

---

### Step 6 — Confirm Upgrade Completion

```bash
oc get clusterversion
# STATUS column should show: Cluster version is <target-version>

oc get clusterversion -o jsonpath='{.items[0].status.history[0]}' | python3 -m json.tool
# "state": "Completed"
```

- [ ] `clusterversion` reports target version
- [ ] History entry shows `"state": "Completed"`
- [ ] No `Progressing` condition on clusterversion

---

## Post-Upgrade Checklist

Run all checks after the upgrade reports `Completed`.

### 1. Cluster Version Verification

```bash
oc get clusterversion
oc version
```

- [ ] `oc version` reports expected server version
- [ ] `clusterversion` history shows `Completed` for target version

---

### 2. Cluster Operator Health

```bash
oc get clusteroperators
oc get clusteroperators | grep -v "True.*False.*False"
```

- [ ] All operators are `Available=True`
- [ ] No operator is `Progressing=True`
- [ ] No operator is `Degraded=True`

---

### 3. Node Health

```bash
oc get nodes
oc get nodes | grep -v " Ready"
oc adm top nodes
```

- [ ] All nodes in `Ready` state
- [ ] No nodes cordoned unexpectedly
- [ ] Node OS version updated (for RHCOS nodes):

```bash
oc get nodes -o wide
oc debug node/<node-name> -- chroot /host rpm-ostree status
```

---

### 4. etcd Health Post-Upgrade

```bash
ETCD_POD=$(oc -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}')
oc -n openshift-etcd exec -it $ETCD_POD -- etcdctl endpoint health \
  --cluster \
  --cacert=/etc/kubernetes/static-pod-resources/etcd-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-serving/etcd-serving-*.crt \
  --key=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-serving/etcd-serving-*.key
```

- [ ] All etcd members healthy
- [ ] etcd has quorum

---

### 5. Pod Health Check

```bash
oc get pods --all-namespaces | grep -Ev "(Running|Completed|Succeeded)"
oc get pods -n openshift-monitoring | grep -v Running
oc get pods -n openshift-ingress | grep -v Running
oc get pods -n openshift-apiserver | grep -v Running
oc get pods -n openshift-etcd | grep -v Running
```

- [ ] No pods in error state in core namespaces
- [ ] No unexpected `CrashLoopBackOff` pods in application namespaces

---

### 6. API Server Availability

```bash
# Confirm API server responds
oc get namespaces
curl -sk https://<api-url>:6443/healthz

# Check for any API deprecation warnings relevant to the new version
oc get apirequestcounts --sort-by='.status.removedInRelease'
```

- [ ] API server is responsive
- [ ] Review deprecated API usage if upgrading across minor versions

---

### 7. Ingress & Route Verification

```bash
# Check default ingress controller
oc -n openshift-ingress get pods
oc -n openshift-ingress-operator get pods

# Test a known application route
curl -sk https://<application-route>/health
```

- [ ] Ingress controller pods are `Running`
- [ ] Application routes are reachable

---

### 8. Monitoring & Alerting

```bash
oc -n openshift-monitoring get pods
oc -n openshift-user-workload-monitoring get pods 2>/dev/null
```

- [ ] Prometheus, Alertmanager, Thanos pods are `Running`
- [ ] Grafana dashboards accessible and showing data
- [ ] No new critical alerts firing post-upgrade

---

### 9. Logging (If Deployed)

```bash
oc get clusterlogforwarder --all-namespaces 2>/dev/null
oc -n openshift-logging get pods
```

- [ ] Logging collector pods are `Running`
- [ ] Logs flowing to configured backends

---

### 10. Storage Verification

```bash
oc get pv
oc get pvc --all-namespaces | grep -v Bound
```

- [ ] All PVCs remain `Bound`
- [ ] No storage-related alerts in Prometheus

---

### 11. Certificate Rotation Check

```bash
# Confirm certificates are valid
oc -n openshift-kube-apiserver get secret \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.auth\.openshift\.io/certificate-not-after}{"\n"}{end}' \
  | sort -k2 | head -20
```

- [ ] No certificates expired or expiring within 7 days
- [ ] kubeconfig certificates are still valid

---

### 12. MachineConfigPool Status

```bash
oc get mcp
```

- [ ] All MCPs show `Updated=True`, `Updating=False`, `Degraded=False`
- [ ] No degraded MachineConfigs

---

### 13. Application Smoke Tests

Work with application owners to execute smoke tests for critical workloads.

| Application | Test | Result | Owner |
|---|---|---|---|
| | | Pass / Fail | |
| | | Pass / Fail | |
| | | Pass / Fail | |

- [ ] All critical applications smoke tested
- [ ] No regressions observed

---

### 14. Update Documentation

- [ ] Cluster inventory updated with new version
- [ ] CMDB / asset management updated
- [ ] Change ticket closed with actual start/end times
- [ ] Runbook updated if deviations were encountered
- [ ] Post-upgrade notes added: `[INSERT NOTES]`

---

### Post-Upgrade Sign-Off

| Name | Role | Signature | Date/Time |
|---|---|---|---|
| | Cluster Admin | | |
| | Application Owner | | |
| | Change Manager | | |

---

## Rollback Procedure

> **OpenShift does not support automatic downgrades.** Rollback means restoring from the etcd backup taken in the pre-upgrade phase. This is a last-resort procedure that will cause downtime.

### When to Rollback

- Cluster operators remain `Degraded` for > 2 hours with no remediation path
- etcd data corruption detected
- Critical application failures that cannot be resolved at application layer
- Red Hat Support recommends rollback

### Rollback Decision Gate

Before proceeding:
1. Contact Red Hat Support: `https://access.redhat.com`
2. Confirm rollback decision with change authority
3. Notify all stakeholders

### Rollback Steps

```bash
# 1. Stop the Cluster Version Operator to prevent re-upgrades
oc scale deployment cluster-version-operator -n openshift-cluster-version --replicas=0

# 2. Follow the official etcd restore procedure:
# https://docs.openshift.com/container-platform/4.x/backup_and_restore/control_plane_backup_and_restore/disaster_recovery/scenario-2-restoring-cluster-state.html

# On each control plane node (one at a time):
# a. Copy the backup files to the node
# b. Run the restore script
sudo /usr/local/bin/cluster-restore.sh /home/core/assets/backup

# 3. Restart the static pods on control plane nodes
# (the restore script handles this)

# 4. Verify etcd is healthy after restore
# 5. Re-enable CVO
oc scale deployment cluster-version-operator -n openshift-cluster-version --replicas=1
```

> Refer to the official Red Hat documentation for the exact restore commands for your OCP version.

---

## Troubleshooting

### Upgrade Stuck — Operator Degraded

```bash
# Identify which operator is degraded
oc get clusteroperators | grep -v "True.*False.*False"

# Get detailed operator status
oc describe clusteroperator <operator-name>

# Check operator pod logs
oc logs -n openshift-<operator-ns> deployment/<operator-deployment> --tail=100
```

### Upgrade Stuck — Node Not Draining

```bash
# Find which node is being drained
oc get nodes | grep SchedulingDisabled

# Check what is blocking the drain (PDBs, non-graceful pods)
oc adm drain <node-name> --dry-run=client --ignore-daemonsets --delete-emptydir-data

# Check PDB violations
oc get pdb --all-namespaces
oc describe pdb <pdb-name> -n <namespace>
```

### MachineConfigPool Degraded

```bash
oc get mcp
oc describe mcp <mcp-name>

# Check MachineConfig daemon pods
oc -n openshift-machine-config-operator get pods
oc logs -n openshift-machine-config-operator \
  $(oc -n openshift-machine-config-operator get pods -l k8s-app=machine-config-daemon -o jsonpath='{.items[0].metadata.name}') \
  -c machine-config-daemon --tail=100
```

### CVO Logs for Upgrade Failures

```bash
oc logs -n openshift-cluster-version \
  $(oc get pods -n openshift-cluster-version -o name) \
  --tail=200 | grep -i "error\|fail\|unable"
```

### etcd Defragmentation (If Needed Post-Upgrade)

```bash
ETCD_POD=$(oc -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}')
oc -n openshift-etcd exec -it $ETCD_POD -- etcdctl defrag \
  --cluster \
  --cacert=/etc/kubernetes/static-pod-resources/etcd-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-serving/etcd-serving-*.crt \
  --key=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-serving/etcd-serving-*.key
```

---

## Reference Links

- [OCP Upgrade Overview](https://docs.openshift.com/container-platform/latest/updating/understanding_updates/understanding-openshift-update-duration.html)
- [Preparing to Update OCP](https://docs.openshift.com/container-platform/latest/updating/preparing_for_updates/updating-cluster-prepare.html)
- [etcd Backup & Restore](https://docs.openshift.com/container-platform/latest/backup_and_restore/control_plane_backup_and_restore/backing-up-etcd.html)
- [Red Hat Support Portal](https://access.redhat.com/support)
- [OCP Release Notes](https://docs.openshift.com/container-platform/latest/release_notes/ocp-4-15-release-notes.html)

---

*End of Runbook*
