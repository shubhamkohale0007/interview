# OpenShift 4.x Cluster Creation Guide

> **Version:** OpenShift Container Platform 4.x (4.12 – 4.16+)
> **Updated:** June 2026

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Installation Methods](#installation-methods)
4. [Step-by-Step: IPI Installation (AWS)](#step-by-step-ipi-installation-aws)
5. [Step-by-Step: UPI Installation (Bare Metal / vSphere)](#step-by-step-upi-installation)
6. [Post-Installation Steps](#post-installation-steps)
7. [Verification Checklist](#verification-checklist)
8. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

### High-Level Cluster Architecture

```
                        ┌─────────────────────────────────────────────────────┐
                        │              OpenShift 4.x Cluster                  │
                        │                                                     │
                        │   ┌──────────────────────────────────────────────┐  │
                        │   │              Control Plane (Masters)         │  │
                        │   │                                              │  │
                        │   │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │  │
                        │   │  │ Master-1 │ │ Master-2 │ │ Master-3 │    │  │
                        │   │  │          │ │          │ │          │    │  │
                        │   │  │ etcd     │ │ etcd     │ │ etcd     │    │  │
                        │   │  │ API Srvr │ │ API Srvr │ │ API Srvr │    │  │
                        │   │  │ Scheduler│ │ Scheduler│ │ Scheduler│    │  │
                        │   │  │ Ctrl Mgr │ │ Ctrl Mgr │ │ Ctrl Mgr │    │  │
                        │   │  └──────────┘ └──────────┘ └──────────┘    │  │
                        │   └──────────────────────────────────────────────┘  │
                        │                        │                            │
                        │              ┌──────────▼──────────┐                │
                        │              │   Internal LB / HAProxy              │
                        │              │  API: 6443 | MCS: 22623              │
                        │              └──────────┬──────────┘                │
                        │                        │                            │
                        │   ┌────────────────────▼─────────────────────────┐  │
                        │   │              Compute Plane (Workers)         │  │
                        │   │                                              │  │
                        │   │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │  │
                        │   │  │ Worker-1 │ │ Worker-2 │ │ Worker-N │    │  │
                        │   │  │          │ │          │ │          │    │  │
                        │   │  │ CRI-O    │ │ CRI-O    │ │ CRI-O    │    │  │
                        │   │  │ Kubelet  │ │ Kubelet  │ │ Kubelet  │    │  │
                        │   │  │ OVN-K8s  │ │ OVN-K8s  │ │ OVN-K8s  │    │  │
                        │   │  └──────────┘ └──────────┘ └──────────┘    │  │
                        │   └──────────────────────────────────────────────┘  │
                        │                        │                            │
                        │   ┌────────────────────▼─────────────────────────┐  │
                        │   │            Infrastructure Nodes (Optional)   │  │
                        │   │   (Router/Ingress, Registry, Monitoring)     │  │
                        │   └──────────────────────────────────────────────┘  │
                        └─────────────────────────────────────────────────────┘
                                               │
              ┌────────────────────────────────┼─────────────────────────────┐
              │                                │                             │
    ┌─────────▼────────┐           ┌───────────▼──────────┐      ┌──────────▼──────┐
    │   External LB    │           │    DNS (Public +     │      │  NTP / Time     │
    │  (Apps: *.apps.) │           │     Private Zones)   │      │   Servers       │
    └──────────────────┘           └──────────────────────┘      └─────────────────┘
```

### Network Architecture

```
  Internet / Corporate Network
           │
    ┌──────▼──────────────────────────────────────────────────────┐
    │                      DMZ / Edge Zone                        │
    │   ┌──────────────────┐       ┌──────────────────────────┐   │
    │   │  External LB     │       │      Bastion Host        │   │
    │   │  Port 443/80     │       │  (SSH Jump / Installer)  │   │
    │   │  *.apps.cluster  │       │                          │   │
    │   └────────┬─────────┘       └────────────┬─────────────┘   │
    └────────────┼────────────────────────────── ┼ ───────────────┘
                 │               VPC / VLAN       │
    ┌────────────▼───────────────────────────────▼───────────────┐
    │                    Cluster Network                          │
    │  ┌─────────────────────────────────────────────────────┐   │
    │  │   API VIP: api.cluster.domain.com  :6443            │   │
    │  │   Ingress VIP: *.apps.cluster.domain.com  :443/80   │   │
    │  └─────────────────────────────────────────────────────┘   │
    │                                                             │
    │   Machine Network: 192.168.0.0/16                          │
    │   Cluster Network: 10.128.0.0/14  (Pod CIDRs)             │
    │   Service Network: 172.30.0.0/16  (ClusterIP CIDRs)       │
    └─────────────────────────────────────────────────────────────┘
```

### Component Interaction Diagram

```
  ┌──────────────┐     ┌───────────────────────────────────────────┐
  │   oc / CLI   │────▶│        Kubernetes API Server :6443        │
  └──────────────┘     └───────────────┬───────────────────────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                           │
  ┌────────▼─────────┐    ┌────────────▼──────────┐   ┌───────────▼──────────┐
  │  etcd (Key-Value │    │ Controller Manager    │   │  Scheduler            │
  │  Cluster State)  │    │ (Reconcile loops)     │   │  (Pod placement)      │
  └──────────────────┘    └───────────────────────┘   └───────────────────────┘
           │
  ┌────────▼──────────────────────────────────────────────────────────────────┐
  │                     Cluster Operators (CVO Managed)                       │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
  │  │ Network  │ │  Image   │ │  DNS     │ │ Ingress  │ │  Monitoring  │   │
  │  │ Operator │ │ Registry │ │ Operator │ │ Operator │ │  Operator    │   │
  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
  └───────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### 1. Infrastructure Requirements

#### Control Plane Nodes (3 Masters — Minimum, Always Odd Number)

| Resource   | Minimum      | Recommended   |
|------------|--------------|---------------|
| vCPUs      | 4            | 8             |
| RAM        | 16 GB        | 32 GB         |
| Disk (OS)  | 120 GB       | 200 GB        |
| Disk type  | SSD          | NVMe SSD      |

#### Worker / Compute Nodes (Minimum 2)

| Resource   | Minimum      | Recommended   |
|------------|--------------|---------------|
| vCPUs      | 2            | 8             |
| RAM        | 8 GB         | 32 GB         |
| Disk (OS)  | 120 GB       | 200 GB        |

#### Bootstrap Node (Temporary — removed after install)

| Resource   | Value        |
|------------|--------------|
| vCPUs      | 4            |
| RAM        | 16 GB        |
| Disk       | 100 GB       |

> **Note:** The bootstrap node is only required during installation. It can be decommissioned after the cluster is healthy.

---

### 2. Network Requirements

#### Required DNS Records

```
# Forward DNS (A Records)
api.<cluster_name>.<base_domain>          → API Load Balancer IP
api-int.<cluster_name>.<base_domain>      → Internal API Load Balancer IP
*.apps.<cluster_name>.<base_domain>       → Ingress/Router Load Balancer IP

# Bootstrap (temporary)
bootstrap.<cluster_name>.<base_domain>    → Bootstrap Node IP

# Master Nodes
master-0.<cluster_name>.<base_domain>     → Master-0 IP
master-1.<cluster_name>.<base_domain>     → Master-1 IP
master-2.<cluster_name>.<base_domain>     → Master-2 IP

# Worker Nodes
worker-0.<cluster_name>.<base_domain>     → Worker-0 IP
worker-1.<cluster_name>.<base_domain>     → Worker-1 IP

# Reverse DNS (PTR Records — Required)
<IP>  → api.<cluster_name>.<base_domain>
<IP>  → master-0.<cluster_name>.<base_domain>
<IP>  → master-1.<cluster_name>.<base_domain>
<IP>  → master-2.<cluster_name>.<base_domain>
```

#### Required Network Ports

| Port      | Protocol | Source        | Destination   | Purpose                    |
|-----------|----------|---------------|---------------|----------------------------|
| 6443      | TCP      | All nodes     | Masters       | Kubernetes API             |
| 22623     | TCP      | Bootstrap, Workers | Masters  | Machine Config Server      |
| 2379-2380 | TCP      | Masters       | Masters       | etcd peer communication    |
| 9000-9999 | TCP/UDP  | All nodes     | All nodes     | Host-level services        |
| 10250     | TCP      | Masters       | Workers       | Kubelet API                |
| 4789      | UDP      | All nodes     | All nodes     | VXLAN (OVN-Kubernetes)     |
| 6081      | UDP      | All nodes     | All nodes     | Geneve (OVN-Kubernetes)    |
| 500/4500  | UDP      | All nodes     | All nodes     | IPsec IKE/NAT-T            |
| 80/443    | TCP      | External      | Ingress LB    | Application traffic        |
| 22        | TCP      | Bastion       | All nodes     | SSH (admin access)         |
| 123       | UDP      | All nodes     | NTP Server    | Time synchronization       |

---

### 3. Software & Tool Requirements

```bash
# Required tools on installation machine (Linux/Mac/WSL)
openshift-install   >= 4.x (matches target OCP version)
oc                  >= 4.x (OpenShift CLI)
kubectl             >= 1.25
jq                  >= 1.6
python3             >= 3.8
ssh-keygen          (for SSH key pair generation)

# For cloud providers
aws                 (AWS CLI >= 2.x)      — for AWS IPI
az                  (Azure CLI >= 2.x)    — for Azure IPI
gcloud              (GCloud SDK)          — for GCP IPI
govc                (vSphere CLI)         — for vSphere UPI/IPI
```

#### Download OpenShift Installer

```bash
# Download from Red Hat Console (requires subscription)
# https://console.redhat.com/openshift/install

# Or via CLI
OCP_VERSION=4.14.20
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-install-linux.tar.gz
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux.tar.gz

tar xvf openshift-install-linux.tar.gz
tar xvf openshift-client-linux.tar.gz

sudo mv openshift-install oc kubectl /usr/local/bin/
openshift-install version
oc version
```

---

### 4. Account & Subscription Requirements

- [ ] **Red Hat account** with valid OpenShift subscription
- [ ] **Pull secret** downloaded from [console.redhat.com](https://console.redhat.com/openshift/install)
- [ ] Cloud provider credentials (IAM roles, Service Accounts) if cloud install
- [ ] SSH key pair generated for node access

#### Minimum Cloud IAM Permissions (AWS IPI)

The IAM user/role must have permissions for:

```
EC2: Full access (instances, VPC, security groups, ELB, Route53)
S3: CreateBucket, PutObject, GetObject, DeleteObject
IAM: CreateRole, AttachRolePolicy, CreateInstanceProfile
Route53: ChangeResourceRecordSets, ListHostedZones
ELB: CreateLoadBalancer, CreateTargetGroup, RegisterTargets
```

> Use the managed policy `openshift-install-*` or attach the Red Hat provided IAM policy JSON.

---

### 5. SSH Key Pair Generation

```bash
# Generate a dedicated SSH key for OpenShift nodes
ssh-keygen -t ed25519 -f ~/.ssh/ocp-cluster-key -C "ocp-cluster-admin"

# Start ssh-agent and add key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/ocp-cluster-key

# Export public key content (used in install-config.yaml)
cat ~/.ssh/ocp-cluster-key.pub
```

---

## Installation Methods

| Method       | Platforms                        | Automation Level | Use Case                  |
|--------------|----------------------------------|------------------|---------------------------|
| **IPI**      | AWS, Azure, GCP, vSphere, Baremetal | Fully automated | Quick PoC / Production    |
| **UPI**      | Baremetal, vSphere, Any          | Manual           | Custom infra / Air-gapped |
| **ROSA**     | AWS                              | Managed service  | Fully managed OCP on AWS  |
| **ARO**      | Azure                            | Managed service  | Fully managed OCP on Azure|
| **Assisted** | Any (via Red Hat Console UI)     | Semi-automated   | Guided install            |

---

## Step-by-Step: IPI Installation (AWS)

### Step 1: Prepare Working Directory

```bash
mkdir ~/ocp-install && cd ~/ocp-install
```

### Step 2: Create install-config.yaml

```yaml
# ~/ocp-install/install-config.yaml
apiVersion: v1
baseDomain: example.com                    # Your base domain
metadata:
  name: ocp-cluster                        # Cluster name (becomes api.ocp-cluster.example.com)
platform:
  aws:
    region: us-east-1
    userTags:
      Environment: production
      Team: platform
compute:
  - architecture: amd64
    hyperthreading: Enabled
    name: worker
    platform:
      aws:
        instanceType: m5.2xlarge           # Worker instance type
        rootVolume:
          iops: 4000
          size: 200
          type: io1
    replicas: 3                            # Number of worker nodes
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      instanceType: m5.xlarge             # Master instance type
      rootVolume:
        iops: 4000
        size: 200
        type: io1
  replicas: 3                             # Always 3 for HA
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14                 # Pod CIDR
      hostPrefix: 23
  machineNetwork:
    - cidr: 10.0.0.0/16                   # Node CIDR (must match VPC)
  networkType: OVNKubernetes              # Recommended: OVNKubernetes or OpenShiftSDN
  serviceNetwork:
    - 172.30.0.0/16                       # Service CIDR
pullSecret: '{"auths": ...}'              # Paste your pull secret here
sshKey: |
  ssh-ed25519 AAAA...                     # Paste your public SSH key here
```

> **Important:** Back up `install-config.yaml` before proceeding — it is consumed and deleted during install.

```bash
cp install-config.yaml install-config.yaml.bak
```

### Step 3: Create Manifests (Optional Customization)

```bash
openshift-install create manifests --dir=~/ocp-install

# Optional: Prevent masters from scheduling workloads
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' \
    ~/ocp-install/manifests/cluster-scheduler-02-config.yml
```

### Step 4: Run Installation

```bash
openshift-install create cluster \
  --dir=~/ocp-install \
  --log-level=info
```

The installer will:
1. Create AWS infrastructure (VPC, subnets, security groups, IAM roles)
2. Launch bootstrap node
3. Launch 3 master nodes
4. Bootstrap the control plane
5. Launch worker nodes
6. Destroy bootstrap node
7. Complete cluster operators

**Expected time:** 30–45 minutes

### Step 5: Monitor Installation Progress

```bash
# In another terminal, watch logs
openshift-install wait-for bootstrap-complete \
  --dir=~/ocp-install \
  --log-level=info

# After bootstrap completes, watch cluster operators
openshift-install wait-for install-complete \
  --dir=~/ocp-install \
  --log-level=info
```

### Step 6: Access the Cluster

```bash
# Set KUBECONFIG
export KUBECONFIG=~/ocp-install/auth/kubeconfig

# Verify cluster access
oc get nodes
oc get clusterversion
oc get clusteroperators

# Get console URL and credentials
cat ~/ocp-install/auth/kubeadmin-password
oc whoami --show-console
```

---

## Step-by-Step: UPI Installation

### UPI Overview (Bare Metal / vSphere)

```
Phase 1: Infrastructure Prep
  └── DNS, DHCP, Load Balancer, PXE/DHCP or ISO

Phase 2: Create Ignition Configs
  └── openshift-install create ignition-configs

Phase 3: Boot Nodes with RHCOS
  └── Bootstrap → Masters (wait) → Workers

Phase 4: Approve CSRs & Complete
  └── oc adm certificate approve <csr>
```

### Step 1: Prepare Infrastructure

#### Configure HAProxy Load Balancer

```bash
# /etc/haproxy/haproxy.cfg (on LB node)
global
    log         127.0.0.1 local2
    maxconn     4000

defaults
    log         global
    mode        tcp
    option      tcplog
    retries     3
    timeout     connect 10s
    timeout     client  1m
    timeout     server  1m

# API (port 6443) — Bootstrap + Masters
frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server

backend openshift-api-server
    balance source
    server bootstrap  192.168.1.10:6443 check
    server master-0   192.168.1.11:6443 check
    server master-1   192.168.1.12:6443 check
    server master-2   192.168.1.13:6443 check

# Machine Config Server (port 22623)
frontend machine-config-server
    bind *:22623
    default_backend machine-config-server

backend machine-config-server
    balance source
    server bootstrap  192.168.1.10:22623 check
    server master-0   192.168.1.11:22623 check
    server master-1   192.168.1.12:22623 check
    server master-2   192.168.1.13:22623 check

# Ingress HTTP (port 80)
frontend ingress-http
    bind *:80
    default_backend ingress-http

backend ingress-http
    balance source
    server worker-0   192.168.1.20:80 check
    server worker-1   192.168.1.21:80 check

# Ingress HTTPS (port 443)
frontend ingress-https
    bind *:443
    default_backend ingress-https

backend ingress-https
    balance source
    server worker-0   192.168.1.20:443 check
    server worker-1   192.168.1.21:443 check
```

### Step 2: Generate Ignition Configs

```bash
mkdir ~/upi-install && cd ~/upi-install

# Create install-config.yaml (platform: none for bare metal)
cat > install-config.yaml <<EOF
apiVersion: v1
baseDomain: example.com
metadata:
  name: ocp-cluster
platform:
  none: {}
compute:
  - name: worker
    replicas: 0                      # UPI: workers added manually
controlPlane:
  name: master
  replicas: 3
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  machineNetwork:
    - cidr: 192.168.1.0/24
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
pullSecret: '{"auths": ...}'
sshKey: |
  ssh-ed25519 AAAA...
EOF

# Generate ignition configs
openshift-install create ignition-configs --dir=~/upi-install

ls ~/upi-install/
# bootstrap.ign  master.ign  worker.ign  auth/  metadata.json
```

### Step 3: Serve Ignition Files via HTTP

```bash
# Serve ignition files from a web server reachable by all nodes
cp ~/upi-install/*.ign /var/www/html/ignition/
chmod 644 /var/www/html/ignition/*.ign
```

### Step 4: Boot RHCOS Nodes

#### Download RHCOS ISO / PXE

```bash
OCP_VERSION=4.14.20
# ISO for manual boot
curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/${OCP_VERSION}/rhcos-live.x86_64.iso

# Or kernel/initramfs for PXE
curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/${OCP_VERSION}/rhcos-live-kernel-x86_64
curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/${OCP_VERSION}/rhcos-live-initramfs.x86_64.img
curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/${OCP_VERSION}/rhcos-live-rootfs.x86_64.img
```

#### Boot Each Node with Ignition

```bash
# Kernel arguments for PXE (bootstrap node example)
coreos.inst.install_dev=/dev/sda
coreos.inst.ignition_url=http://192.168.1.100:8080/ignition/bootstrap.ign

# For masters:
coreos.inst.ignition_url=http://192.168.1.100:8080/ignition/master.ign

# For workers:
coreos.inst.ignition_url=http://192.168.1.100:8080/ignition/worker.ign
```

**Boot order:** Bootstrap → Masters (wait for bootstrap complete) → Workers

### Step 5: Wait for Bootstrap Complete

```bash
export KUBECONFIG=~/upi-install/auth/kubeconfig

openshift-install wait-for bootstrap-complete \
  --dir=~/upi-install \
  --log-level=info

# Once complete, remove bootstrap from load balancer
# Edit haproxy.cfg and remove bootstrap server entries
systemctl reload haproxy
```

### Step 6: Approve Worker CSRs

```bash
# Workers generate Certificate Signing Requests — must be manually approved
watch oc get csr

# Approve all pending CSRs
oc get csr -o go-template='{{range .items}}{{if not .status.certificate}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | \
  xargs oc adm certificate approve

# Repeat until all workers are Ready
oc get nodes
```

### Step 7: Complete Installation

```bash
openshift-install wait-for install-complete \
  --dir=~/upi-install \
  --log-level=info
```

---

## Post-Installation Steps

### 1. Configure Authentication (Replace kubeadmin)

#### Configure HTPasswd Identity Provider

```bash
# Create htpasswd file
htpasswd -c -B -b /tmp/htpasswd admin SecurePassword123!
htpasswd -B -b /tmp/htpasswd developer DevPassword456!

# Create secret
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=/tmp/htpasswd \
  -n openshift-config

# Apply OAuth configuration
cat <<EOF | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: htpasswd_provider
      mappingMethod: claim
      type: HTPasswd
      htpasswd:
        fileData:
          name: htpasswd-secret
EOF

# Grant admin role
oc adm policy add-cluster-role-to-user cluster-admin admin

# Test new credentials
oc login -u admin -p SecurePassword123!

# Remove kubeadmin (ONLY after verifying your admin works!)
oc delete secret kubeadmin -n kube-system
```

### 2. Configure Persistent Storage

#### Option A: AWS EBS (Cloud)

```bash
# Verify default storage class
oc get storageclass

# AWS EBS is automatically configured on AWS IPI installs
# Set it as default if needed
oc patch storageclass gp2 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### Option B: NFS (On-Premises)

```bash
# Deploy NFS provisioner
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.1.200 \
  --set nfs.path=/exports/openshift \
  --set storageClass.defaultClass=true
```

#### Option C: OpenShift Data Foundation (ODF)

```bash
# Install ODF Operator via OperatorHub
oc create namespace openshift-storage

cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  targetNamespaces:
    - openshift-storage
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: odf-operator
  namespace: openshift-storage
spec:
  channel: stable-4.14
  name: odf-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### 3. Configure Internal Image Registry

```bash
# For clusters without default storage (UPI/bare metal)
# Use emptyDir (non-persistent — for dev only)
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"storage":{"emptyDir":{}},"managementState":"Managed"}}'

# For production — use PVC backed storage
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: nfs-client
EOF

oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"storage":{"pvc":{"claim":"registry-storage"}},"managementState":"Managed"}}'
```

### 4. Configure Monitoring Stack

```bash
# Customize Prometheus retention and storage
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    prometheusK8s:
      retention: 15d
      volumeClaimTemplate:
        spec:
          storageClassName: gp2
          resources:
            requests:
              storage: 50Gi
    alertmanagerMain:
      volumeClaimTemplate:
        spec:
          storageClassName: gp2
          resources:
            requests:
              storage: 10Gi
EOF
```

### 5. Configure Ingress / Router Certificate

```bash
# Create TLS secret with your wildcard certificate
oc create secret tls apps-wildcard-cert \
  --cert=wildcard.apps.fullchain.pem \
  --key=wildcard.apps.key \
  -n openshift-ingress

# Patch ingress operator to use your cert
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type=merge \
  --patch='{"spec":{"defaultCertificate":{"name":"apps-wildcard-cert"}}}'
```

### 6. Configure API Server Certificate

```bash
# Create secret with your API cert
oc create secret tls api-cert \
  --cert=api.fullchain.pem \
  --key=api.key \
  -n openshift-config

# Patch apiserver
oc patch apiserver cluster \
  --type=merge \
  --patch='{"spec":{"servingCerts":{"namedCertificates":[{"names":["api.ocp-cluster.example.com"],"servingCertificate":{"name":"api-cert"}}]}}}'
```

### 7. Set Up Cluster Logging (Optional)

```bash
# Install Logging Operator
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  channel: stable
  name: cluster-logging
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Create ClusterLogging instance
cat <<EOF | oc apply -f -
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  managementState: Managed
  logStore:
    type: elasticsearch
    retentionPolicy:
      application:
        maxAge: 7d
      infra:
        maxAge: 7d
      audit:
        maxAge: 7d
    elasticsearch:
      nodeCount: 3
      storage:
        storageClassName: gp2
        size: 200G
      resources:
        requests:
          memory: 8Gi
      redundancyPolicy: SingleRedundancy
  visualization:
    type: kibana
    kibana:
      replicas: 1
  collection:
    type: fluentd
EOF
```

### 8. Configure RBAC and Project Defaults

```bash
# Create project template with default quotas
cat <<EOF | oc apply -f -
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: project-request
  namespace: openshift-config
objects:
  - apiVersion: project.openshift.io/v1
    kind: Project
    metadata:
      name: \${PROJECT_NAME}
  - apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: default-quota
      namespace: \${PROJECT_NAME}
    spec:
      hard:
        requests.cpu: "4"
        requests.memory: 8Gi
        limits.cpu: "8"
        limits.memory: 16Gi
        pods: "20"
parameters:
  - name: PROJECT_NAME
  - name: PROJECT_DISPLAYNAME
  - name: PROJECT_DESCRIPTION
  - name: PROJECT_ADMIN_USER
  - name: PROJECT_REQUESTING_USER
EOF

# Set as default project template
oc patch project.config.openshift.io/cluster \
  --type=merge \
  --patch='{"spec":{"projectRequestTemplate":{"name":"project-request"}}}'
```

### 9. Configure Cluster Autoscaler (Cloud Installs)

```bash
# Enable cluster autoscaler
cat <<EOF | oc apply -f -
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  resourceLimits:
    maxNodesTotal: 20
  scaleDown:
    enabled: true
    delayAfterAdd: 10m
    unneededTime: 5m
---
apiVersion: autoscaling.openshift.io/v1beta1
kind: MachineAutoscaler
metadata:
  name: worker-us-east-1a
  namespace: openshift-machine-api
spec:
  minReplicas: 2
  maxReplicas: 10
  scaleTargetRef:
    apiVersion: machine.openshift.io/v1beta1
    kind: MachineSet
    name: ocp-cluster-worker-us-east-1a
EOF
```

### 10. Enable Operator Hub & Marketplace

```bash
# Verify OperatorHub is enabled
oc get operatorhub cluster -o yaml

# Add custom catalog source (air-gapped)
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: custom-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: registry.example.com/catalog:latest
  displayName: Custom Catalog
  publisher: Internal
EOF
```

---

## Verification Checklist

### Cluster Health Verification

```bash
# 1. Check cluster version and operators
oc get clusterversion
oc get clusteroperators

# All operators should show: AVAILABLE=True, PROGRESSING=False, DEGRADED=False
# Example output:
# NAME                                       AVAILABLE   PROGRESSING   DEGRADED
# authentication                             True        False         False
# cloud-credential                           True        False         False
# console                                    True        False         False
# dns                                        True        False         False
# etcd                                       True        False         False
# ingress                                    True        False         False
# ...

# 2. Verify all nodes are Ready
oc get nodes
# All nodes: STATUS=Ready

# 3. Check etcd health
oc rsh -n openshift-etcd etcd-master-0 \
  etcdctl member list \
  --cacert=/etc/kubernetes/static-pod-resources/etcd-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-certs/etcd-peer-master-0.crt \
  --key=/etc/kubernetes/static-pod-resources/etcd-certs/secrets/etcd-all-certs/etcd-peer-master-0.key \
  --endpoints=https://localhost:2379

# 4. Check all pods are running
oc get pods --all-namespaces | grep -v Running | grep -v Completed

# 5. Verify console access
oc whoami --show-console

# 6. Check API server
oc cluster-info

# 7. Verify image registry
oc get configs.imageregistry.operator.openshift.io cluster -o yaml | grep managementState

# 8. Test application deployment
oc new-project test-deploy
oc new-app nginx
oc expose service nginx
oc get route nginx
curl http://$(oc get route nginx -o jsonpath='{.spec.host}')
oc delete project test-deploy

# 9. Check storage
oc get storageclass
oc get pv

# 10. Validate monitoring
oc get pods -n openshift-monitoring
oc get route -n openshift-monitoring
```

### Security Verification

```bash
# Verify kubeadmin is removed
oc get secret kubeadmin -n kube-system 2>&1 | grep "not found"

# Check OAuth configuration
oc get oauth cluster -o yaml

# Verify TLS certificates
oc get secret -n openshift-ingress | grep tls
oc get apiserver cluster -o yaml | grep namedCertificates -A 5

# Check SCC (Security Context Constraints)
oc get scc

# Verify network policies
oc get networkpolicy --all-namespaces

# Check audit logging
oc get apiservers cluster -o yaml | grep audit -A 10
```

---

## Troubleshooting

### Common Issues and Fixes

#### Bootstrap Fails / Times Out

```bash
# SSH to bootstrap and check logs
ssh -i ~/.ssh/ocp-cluster-key core@<bootstrap-ip>

# On bootstrap node:
journalctl -b -f -u release-image.service -u bootkube.service

# Check bootstrap complete
sudo journalctl -b -u bootkube.service | tail -50
```

#### Masters Not Joining

```bash
# Check etcd pods
oc get pods -n openshift-etcd

# Check node logs
oc get nodes
oc describe node master-0

# Check machine config
oc get mcp
oc get mc
```

#### Worker CSRs Stuck Pending

```bash
# Batch approve all pending CSRs (run multiple times if needed)
for csr in $(oc get csr -o name | grep Pending 2>/dev/null); do
  oc adm certificate approve "$csr"
done
```

#### Cluster Operator Degraded

```bash
# Identify degraded operator
oc get co | grep -v "True.*False.*False"

# Get operator details
oc describe co <operator-name>

# Check operator pod logs
oc get pods -n openshift-<operator-namespace>
oc logs -n openshift-<operator-namespace> <operator-pod>
```

#### DNS Resolution Failures

```bash
# Test DNS from inside cluster
oc run dns-test --image=busybox --rm -it -- nslookup kubernetes.default

# Check cluster DNS operator
oc get pods -n openshift-dns
oc logs -n openshift-dns <coredns-pod>

# Verify DNS config
oc get dns cluster -o yaml
```

#### Certificate Expired / Clock Skew

```bash
# Check node time sync
for node in $(oc get nodes -o name); do
  echo "=== $node ==="
  oc debug $node -- chroot /host timedatectl status 2>/dev/null
done

# Approve pending CSRs after time fix
oc get csr | grep Pending | awk '{print $1}' | xargs oc adm certificate approve
```

---

## Quick Reference

### Essential Commands

```bash
# Cluster status
oc get clusterversion
oc get co                              # cluster operators
oc get nodes                          # node status
oc get mcp                            # machine config pools

# Common admin operations
oc adm top nodes                      # node resource usage
oc adm top pods --all-namespaces      # pod resource usage
oc adm drain <node> --ignore-daemonsets --delete-emptydir-data
oc adm uncordon <node>

# Upgrades
oc adm upgrade                        # check available upgrades
oc adm upgrade --to=4.14.25          # upgrade to specific version

# Certificates
oc adm certificate approve <csr>

# Debug
oc debug node/<node-name>            # shell into node
oc adm must-gather                   # collect diagnostics
oc adm inspect co/<operator-name>   # inspect operator
```

### Key File Locations (Post-Install)

| File                              | Purpose                        |
|-----------------------------------|--------------------------------|
| `./auth/kubeconfig`               | Admin kubeconfig               |
| `./auth/kubeadmin-password`       | Initial admin password         |
| `./metadata.json`                 | Cluster metadata               |
| `./bootstrap.ign`                 | Bootstrap ignition (UPI)       |
| `./master.ign`                    | Master ignition (UPI)          |
| `./worker.ign`                    | Worker ignition (UPI)          |
| `/etc/kubernetes/`                | Kubernetes configs (on nodes)  |
| `/var/log/pods/`                  | Pod logs (on nodes)            |

---

## Resources

- [OpenShift 4.x Documentation](https://docs.openshift.com/container-platform/latest/)
- [Red Hat Console — Cluster Creation](https://console.redhat.com/openshift/install)
- [OpenShift Mirror — Downloads](https://mirror.openshift.com/pub/openshift-v4/)
- [RHCOS Downloads](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/)
- [OpenShift Sizing Calculator](https://access.redhat.com/articles/4128421)
- [Network Requirements](https://docs.openshift.com/container-platform/latest/installing/installing_bare_metal/installing-bare-metal.html#installation-network-user-infra_installing-bare-metal)

---

*Document covers OpenShift Container Platform 4.12 through 4.16+. Always consult the version-specific documentation for the exact release you are deploying.*
