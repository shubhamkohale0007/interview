# OpenShift — Expose Non-HTTP/SNI Applications
### Interview Questions, Answers & Key Points | Easy English | 8+ Years Experience

---

## Table of Contents

1. [Important Concepts to Remember](#1-important-concepts-to-remember)
2. [Load Balancer Services — Questions](#2-load-balancer-services--questions)
3. [MetalLB — Questions](#3-metallb--questions)
4. [Multus Secondary Networks — Questions](#4-multus-secondary-networks--questions)
5. [NetworkAttachmentDefinition — Questions](#5-networkattachmentdefinition--questions)
6. [Hands-On / Practical Questions](#6-hands-on--practical-questions)
7. [Scenario-Based Questions](#7-scenario-based-questions)
8. [Quick Reference Cheatsheet](#8-quick-reference-cheatsheet)

---

## 1. Important Concepts to Remember

> These are the most important points from the chapter. Read these first.

---

### Key Point 1 — When to Use What

```
HTTP services     → Use Ingress or Route (best option, always prefer this)
Non-HTTP services → Use LoadBalancer Service or Multus Secondary Network
```

> **Remember:** Ingress and Routes ONLY work with HTTP/HTTPS. For everything else (SSH, RTSP, PostgreSQL, etc.) you need LoadBalancer or Multus.

---

### Key Point 2 — Kubernetes Service Types

| Service Type | Used For | Accessible From |
|---|---|---|
| **ClusterIP** | Internal communication only | Inside cluster only |
| **NodePort** | External access via node IP + port | Outside cluster |
| **LoadBalancer** | External access with dedicated IP | Outside cluster |

> **Remember:** LoadBalancer is the cleanest way to expose non-HTTP services externally.

---

### Key Point 3 — MetalLB Facts

- MetalLB is used when cluster is **NOT on a cloud provider** (bare metal or on hypervisor)
- MetalLB has **two modes**: Layer 2 and BGP
- MetalLB assigns IPs from a pool you define called **IPAddressPool**
- If IP pool runs out → new services stay in **`<pending>`** state
- Installed via **Operator Lifecycle Manager (OLM)**

---

### Key Point 4 — Multus CNI Key Facts

- Multus allows pods to have **more than one network interface**
- Default cluster network (OVN-Kubernetes) stays on `eth0`
- Extra networks are added as `net1`, `net2`, etc.
- Extra networks are defined using **NetworkAttachmentDefinition** resource
- Pods use the annotation `k8s.v1.cni.cncf.io/networks` to join extra networks
- NetworkAttachmentDefinitions are **namespaced** (only pods in same namespace can use them)

---

### Key Point 5 — Network Attachment Types

| Type | What it does |
|---|---|
| **Host Device** | Gives a node's physical NIC directly to ONE pod |
| **Bridge** | Multiple pods share one virtual bridge (like a switch) |
| **IPVLAN** | Uses fewer MAC addresses — good for networks with MAC limits |
| **MACVLAN** | Each pod gets its own MAC address |

> **Remember:** Host Device = exclusive — only ONE pod at a time can use it.

---

### Key Point 6 — IPAM (IP Address Management)

IPAM controls how pods get IP addresses on the secondary network.

| IPAM Type | How it works |
|---|---|
| **DHCP** | Pod gets IP from a DHCP server automatically |
| **Static** | You hardcode the IP address in the config |

---

### Key Point 7 — Important Annotations

```yaml
# Add a pod to a secondary network
k8s.v1.cni.cncf.io/networks: custom

# Check network status of a pod (auto-filled by Multus)
k8s.v1.cni.cncf.io/network-status
```

---

## 2. Load Balancer Services — Questions

---

**Q1. Why can't we use Ingress or Route to expose a PostgreSQL or SSH service?**

**A:**
Ingress and Routes work only with **HTTP and HTTPS** protocols. They use a feature called **virtual hosting** — they look at the request hostname to decide where to send traffic.

Protocols like PostgreSQL, SSH, RTSP (video streaming), etc. do NOT have virtual hosting. So you cannot use Ingress or Route for them.

For non-HTTP services, you must use:
- **LoadBalancer Service** — gives dedicated IP address to the service
- **Multus Secondary Network** — attaches pod directly to an external network

> **Interview tip:** Always say "Ingress uses virtual hosting of HTTP" — that shows deep understanding.

---

**Q2. What is a LoadBalancer Service in Kubernetes? How is it different from ClusterIP?**

**A:**

| ClusterIP | LoadBalancer |
|---|---|
| Only accessible inside the cluster | Accessible from outside the cluster |
| Gets a cluster-internal IP | Gets an external IP from a load balancer provider |
| Used for internal pod-to-pod communication | Used to expose services to external users |
| Default service type | Requires a load balancer component (cloud or MetalLB) |

LoadBalancer service gives your service a **public IP address** so external clients can connect.

---

**Q3. How do you create a LoadBalancer service using the command line?**

**A:**
```bash
# Using oc expose (imperative way)
oc expose deployment/virtual-rtsp-1 \
  --type=LoadBalancer \
  --target-port=8554

# Check the external IP assigned
oc get services
```

Or using a YAML file (declarative way):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-lb
  namespace: example
spec:
  ports:
  - port: 1234
    protocol: TCP
    targetPort: 1234
  selector:
    name: example
  type: LoadBalancer   # This line makes it a LoadBalancer service
```

---

**Q4. After creating a LoadBalancer service, how do you find the external IP?**

**A:**
```bash
# Method 1 — get services
oc get services
# Look at the EXTERNAL-IP column

# Method 2 — using jsonpath for exact IP
oc get service example-lb \
  -o jsonpath="{.status.loadBalancer.ingress}"
# Output: [{"ip":"192.168.50.20"}]
```

---

**Q5. What does it mean when the EXTERNAL-IP of a LoadBalancer service shows `<pending>`?**

**A:**
`<pending>` means the service is waiting for an IP address to be assigned.

This happens when:
- **No more IPs are available** in the IP address pool (most common reason)
- The load balancer component (MetalLB) is not installed or not configured
- The IP pool (IPAddressPool) is exhausted

> **Real example from the exercise:** The classroom had only 2 IPs (192.168.50.20 and 192.168.50.21). When a third LoadBalancer service was created, it showed `<pending>` because no IPs were left.

**Solution:** Delete an existing service to free up an IP, or add more IPs to the pool.

---

**Q6. After exposing a service with LoadBalancer, what should you verify?**

**A:**
1. **Check the external IP is assigned** — not `<pending>`
   ```bash
   oc get services
   ```
2. **Test connectivity to the IP and port**
   ```bash
   nc -vz 192.168.50.20 8554
   ```
3. **Test using the actual protocol client** — connect with the real application (e.g., media player, psql, ssh)
4. **Verify load balancing** — if multiple pods, confirm traffic is distributed
5. **Use network tools** if needed — `ping`, `traceroute`, `nc`

> **Important:** Just because the IP shows up does not mean the service is working. Always test with the actual client.

---

## 3. MetalLB — Questions

---

**Q7. What is MetalLB and when do you use it?**

**A:**
MetalLB is a **load balancer component for bare metal clusters** (clusters not running on a cloud provider).

**When to use MetalLB:**
- Your cluster runs on physical servers (bare metal)
- Your cluster runs on hypervisors (VMs) without a cloud provider
- You need LoadBalancer services but have no cloud provider (no AWS, GCP, Azure)

**When NOT to use MetalLB:**
- Your cluster is on AWS → use AWS NLB/ALB
- Your cluster is on GCP → use Google Cloud Load Balancer
- Your cluster is on Azure → use Azure Load Balancer

Cloud providers have their own built-in load balancer integration.

---

**Q8. What are the two modes of MetalLB? What is the difference?**

**A:**

| Mode | How it works | Use case |
|---|---|---|
| **Layer 2** | One node answers ARP requests for the service IP, then forwards traffic | Simpler, works in most networks |
| **BGP** | Uses Border Gateway Protocol to advertise routes to the service IP | Complex networks, data centers with BGP routers |

Layer 2 is simpler and more common in lab/on-premise setups. BGP gives better load balancing and works at scale in data center environments.

---

**Q9. How do you install MetalLB on OpenShift?**

**A:**
MetalLB is installed using the **Operator Lifecycle Manager (OLM)**:
1. Go to OpenShift Console → OperatorHub
2. Search for "MetalLB"
3. Install the MetalLB Operator
4. After installing, create the MetalLB custom resource to start it
5. Create an **IPAddressPool** resource to define the IP ranges MetalLB can assign
6. Create an **L2Advertisement** resource (for Layer 2 mode)

---

**Q10. What is IPAddressPool in MetalLB?**

**A:**
IPAddressPool is a **MetalLB custom resource** that defines which IP addresses MetalLB can give to LoadBalancer services.

Example:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.50.20-192.168.50.21   # Only 2 IPs available
```

> **Important:** If you define only 2 IPs, you can only have 2 LoadBalancer services at a time. The third will stay in `<pending>`.

---

## 4. Multus Secondary Networks — Questions

---

**Q11. What is Multus CNI and why do we use it?**

**A:**
Multus CNI is a **container network interface plugin** that allows pods to have **multiple network interfaces**.

By default, every pod has only one network interface (`eth0`) connected to the cluster pod network.

With Multus, a pod can have:
- `eth0` → connected to the default cluster network (OVN-Kubernetes)
- `net1` → connected to a custom/secondary network
- `net2` → connected to another network (if needed)

**Why use it:**
- Connect a pod directly to an external network (without going through the cluster network)
- Better performance — dedicated bandwidth for specific traffic
- Better security — isolate certain traffic on a separate network
- Simplify outgoing traffic control

---

**Q12. What are the benefits of using a secondary network for a database pod?**

**A:**
Benefits:

1. **Network isolation** — the database is only reachable from the specific secondary network, not from the public internet
2. **Security** — clients not connected to that network cannot access the database
3. **Performance** — dedicated network bandwidth, no sharing with cluster traffic
4. **No need for a service** — the pod gets a direct IP on the secondary network
5. **Direct access** — external clients connect directly to the pod IP on the secondary network

> **From the exercise:** The PostgreSQL database was accessible from the `utility` machine (which was on the 192.168.51.0/24 network) but NOT from the `workstation` machine (which had no route to that network). This is perfect network isolation.

---

**Q13. What is a NetworkAttachmentDefinition? Write an example.**

**A:**
A NetworkAttachmentDefinition (NAD) is a **Kubernetes custom resource** that describes how to attach a pod to a secondary network.

Example for a host-device type with static IP:
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: custom                  # Name used in pod annotation
spec:
  config: |-
    {
      "cniVersion": "0.3.1",
      "name": "custom",         # Must match metadata.name
      "type": "host-device",    # Network attachment type
      "device": "ens4",         # Node network interface to use
      "ipam": {
        "type": "static",
        "addresses": [
          {"address": "192.168.51.10/24"}   # Static IP for the pod
        ]
      }
    }
```

---

**Q14. How do you attach a pod to a secondary network?**

**A:**
Add the annotation `k8s.v1.cni.cncf.io/networks` to the pod template in the deployment:

```yaml
spec:
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: custom   # Name of the NetworkAttachmentDefinition
      labels:
        app: my-app
    spec:
      containers:
      ...
```

After the pod starts, Multus automatically:
- Creates a new network interface (`net1`) in the pod
- Assigns the IP address from the NetworkAttachmentDefinition config

---

**Q15. How do you verify that a pod is successfully connected to a secondary network?**

**A:**
Check the `k8s.v1.cni.cncf.io/network-status` annotation on the pod:

```bash
oc get pod <pod-name> \
  -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'
```

Expected output showing two networks:
```json
[{
    "name": "ovn-kubernetes",
    "interface": "eth0",
    "ips": ["10.8.0.92"],
    "default": true
},{
    "name": "non-http-multus/custom",
    "interface": "net1",
    "ips": ["192.168.51.10"],
    "mac": "52:54:00:01:33:0a"
}]
```

- `eth0` → default cluster network
- `net1` → secondary network (Multus added this)

> **Tip:** The periods in the annotation name must be escaped with `\` in JSONPath.

---

**Q16. NetworkAttachmentDefinitions are namespaced. What does this mean?**

**A:**
It means a NetworkAttachmentDefinition created in namespace `non-http-multus` can **only be used by pods in that same namespace**.

Pods in a different namespace cannot use it.

If you need the same secondary network in multiple namespaces, you must create a NetworkAttachmentDefinition in each namespace.

---

## 5. NetworkAttachmentDefinition — Questions

---

**Q17. What is the difference between host-device and bridge network attachment types?**

**A:**

| | Host Device | Bridge |
|---|---|---|
| **What it does** | Gives a physical NIC directly to a pod | Multiple pods share a virtual bridge |
| **Pods that can use it** | Only ONE pod at a time | Many pods at the same time |
| **Use case** | Single pod needs exclusive network access | Multiple pods need to communicate on the same network |
| **Example** | Database pod that needs exclusive access to ens4 | Multiple microservices on a private VLAN |

> **Important:** If you use host-device with a Deployment that has `replicas: 2`, the second pod will FAIL because only one pod can own the NIC.

> **From the exercise:** The database used host-device because "this application requires exclusive access to the database data. Only one pod must be running at a time" — that's why they also used the `Recreate` deployment strategy.

---

**Q18. What is the difference between MACVLAN and IPVLAN?**

**A:**

| | MACVLAN | IPVLAN |
|---|---|---|
| **MAC address** | Each pod gets its OWN MAC address | All pods share the parent interface MAC address |
| **Number of MACs** | More MAC addresses used | Fewer MAC addresses used |
| **When to use** | Standard environments | Networks with MAC address limits |
| **Example** | Most on-premise setups | Cloud environments, some switches limit MACs per port |

> **Simple rule:** If the network has a limit on MAC addresses per port → use IPVLAN. Otherwise → MACVLAN works fine.

---

**Q19. What is IPAM in a NetworkAttachmentDefinition?**

**A:**
IPAM = **IP Address Management**. It controls how the pod gets an IP address on the secondary network.

Two common types:

**Static IPAM — you define the IP:**
```json
"ipam": {
  "type": "static",
  "addresses": [
    {"address": "192.168.51.10/24"}
  ]
}
```

**DHCP IPAM — IP assigned automatically:**
```json
"ipam": {
  "type": "dhcp"
}
```

> **Use static** when you need a predictable, fixed IP (e.g., database servers).
> **Use DHCP** when IPs are managed by an existing DHCP server on that network.

---

**Q20. Can you also define secondary networks in the cluster network operator instead of creating a NetworkAttachmentDefinition separately?**

**A:**
Yes. You can edit the cluster Network operator configuration directly:

```yaml
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks:
  - name: custom
    namespace: non-http-multus
    rawCNIConfig: |-
      {
        "cniVersion": "0.3.1",
        "name": "custom",
        "type": "host-device",
        "device": "ens4",
        "ipam": {
          "type": "static",
          "addresses": [
            {"address": "192.168.51.10/24"}
          ]
        }
      }
    type: Raw
```

Both methods achieve the same result. The standalone NetworkAttachmentDefinition approach is more flexible for namespace-level management.

---

## 6. Hands-On / Practical Questions

---

**Q21. Write the full steps to expose a video stream (RTSP on port 8554) using a LoadBalancer service.**

**A:**
```bash
# Step 1: Create the deployment
oc apply -f virtual-rtsp-1.yaml

# Step 2: Wait for pod to be ready
oc get pods -w

# Step 3: Create LoadBalancer service
oc expose deployment/virtual-rtsp-1 \
  --type=LoadBalancer \
  --target-port=8554

# Step 4: Get the external IP
oc get services

# Step 5: Test connectivity
nc -vz 192.168.50.20 8554

# Step 6: Test with real client
totem rtsp://192.168.50.20:8554/stream
```

---

**Q22. Write the full steps to connect a PostgreSQL pod to a secondary network using Multus.**

**A:**

**Step 1: Create the NetworkAttachmentDefinition (as admin)**
```yaml
# network-attachment-definition.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: custom
spec:
  config: |-
    {
      "cniVersion": "0.3.1",
      "name": "custom",
      "type": "host-device",
      "device": "ens4",
      "ipam": {
        "type": "static",
        "addresses": [
          {"address": "192.168.51.10/24"}
        ]
      }
    }
```
```bash
oc create -f network-attachment-definition.yaml
```

**Step 2: Add annotation to the deployment**
```yaml
spec:
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: custom
```
```bash
oc apply -f deployment.yaml
```

**Step 3: Verify the pod has two network interfaces**
```bash
oc get pod <pod-name> \
  -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'
```

**Step 4: Test connectivity from a machine on the secondary network**
```bash
psql -h 192.168.51.10 -U user sample -c 'SELECT 1;'
```

---

**Q23. How do you check the network interfaces of an OpenShift node?**

**A:**
```bash
# Run ip addr on the node using oc debug
oc debug node/master01 -- chroot /host ip addr

# Run ip route to see routing table
oc debug node/master01 -- chroot /host ip route
```

The `chroot /host` part is needed because the debug pod starts in a container. `chroot /host` gives access to the actual host filesystem and commands.

---

**Q24. How do you check if two machines can communicate with each other?**

**A:**
```bash
# Basic connectivity check
ping 192.168.51.10

# Port-level connectivity check
nc -vz 192.168.51.10 5432

# Trace the network path
traceroute 192.168.51.10

# View routing table (to understand if route exists)
ip route
```

> **From the exercise:** The `workstation` had no route to `192.168.51.0/24`, so `ping 192.168.51.10` failed. The `utility` machine had `eth2` on that network, so it could connect.

---

## 7. Scenario-Based Questions

---

**Q25. You have 3 video stream applications, each needing a LoadBalancer service. After creating the third service, it shows `<pending>`. What do you do?**

**A:**
1. **Confirm the issue** — the IP pool is exhausted
   ```bash
   oc get services
   # Third service shows <pending> in EXTERNAL-IP column
   ```

2. **Check MetalLB IP pool** — how many IPs are configured?
   ```bash
   oc get ipaddresspool -n metallb-system -o yaml
   ```

3. **Options to fix:**

   **Option A** — Delete an unused service to free an IP:
   ```bash
   oc delete service/virtual-rtsp-1
   # Now the third service gets the released IP
   ```

   **Option B** — Expand the IP pool (requires admin access):
   - Edit the IPAddressPool to add more IP addresses

   **Option C** — Rethink the design:
   - Can multiple cameras share one IP on different ports?
   - Can we use a different approach like Multus?

---

**Q26. A developer says "I added the secondary network annotation to my pod but the pod still doesn't have the net1 interface." How do you troubleshoot?**

**A:**
Step by step:

1. **Check if NetworkAttachmentDefinition exists in the correct namespace:**
   ```bash
   oc get networkattachmentdefinition -n <namespace>
   ```

2. **Check the annotation spelling is correct:**
   ```bash
   oc get pod <pod-name> -o yaml | grep -A2 annotations
   # Must be: k8s.v1.cni.cncf.io/networks: custom
   ```

3. **Check pod events for errors:**
   ```bash
   oc describe pod <pod-name>
   # Look for Multus-related errors in Events section
   ```

4. **Check if the node interface (ens4) is available:**
   ```bash
   oc debug node/master01 -- chroot /host ip addr
   ```

5. **Check if another pod is already using the host-device interface:**
   - Host-device type allows only ONE pod at a time
   - If another pod is using ens4, this pod will fail

6. **Check Multus logs:**
   ```bash
   oc logs -n openshift-multus -l app=multus
   ```

---

**Q27. Why would you choose Multus secondary network over a LoadBalancer service for a database?**

**A:**

| Reason | LoadBalancer | Multus Secondary Network |
|---|---|---|
| **Security** | Service is reachable from outside cluster on a public IP | Only reachable from machines on the specific secondary network |
| **IP address needed** | Uses IP from MetalLB pool | Uses IP directly from the secondary network |
| **Network isolation** | No isolation — anyone who knows the IP can try to connect | Strong isolation — must be physically on the same network |
| **Protocol** | Works for any TCP/UDP protocol | Works for any protocol |
| **Use case** | Public-facing services | Private, secure, isolated services |

**For a database:**
- You usually do NOT want it exposed publicly
- Multus gives it a private IP on an internal/management network
- Only machines physically on that network can access it
- Much more secure than a LoadBalancer service

---

**Q28. Your team wants to expose both HTTP (port 80) and non-HTTP (port 5432 PostgreSQL) services. What approach do you use for each?**

**A:**

| Service | Protocol | Approach | Reason |
|---|---|---|---|
| Web application | HTTP/HTTPS | **Route or Ingress** | HTTP supports virtual hosting, can share one IP for many apps |
| PostgreSQL database | TCP (non-HTTP) | **LoadBalancer or Multus** | No virtual hosting, needs dedicated IP or secondary network |

Implementation:
```bash
# For web app — use Route
oc expose service/web-app
# OR create an Ingress resource

# For database — use LoadBalancer
oc expose deployment/database \
  --type=LoadBalancer \
  --target-port=5432

# OR — for better security — use Multus
# Create NetworkAttachmentDefinition
# Add annotation to deployment
```

> **Always prefer Route/Ingress for HTTP** — it is simpler and shares IP addresses efficiently.

---

## 8. Quick Reference Cheatsheet

```bash
# ─── LOAD BALANCER SERVICE ────────────────────────────────────
# Create LoadBalancer service
oc expose deployment/<name> --type=LoadBalancer --target-port=<port>

# Get external IP
oc get services
oc get service <name> -o jsonpath="{.status.loadBalancer.ingress}"

# Test port connectivity
nc -vz <external-ip> <port>

# ─── MULTUS SECONDARY NETWORK ─────────────────────────────────
# Create NetworkAttachmentDefinition
oc create -f network-attachment-definition.yaml

# List NetworkAttachmentDefinitions
oc get networkattachmentdefinition

# Check pod secondary network status
oc get pod <pod-name> \
  -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'

# Debug node network interfaces
oc debug node/<node-name> -- chroot /host ip addr

# ─── TROUBLESHOOTING ──────────────────────────────────────────
# Check pod events
oc describe pod <pod-name>

# Check Multus logs
oc logs -n openshift-multus -l app=multus

# Check MetalLB IP pool
oc get ipaddresspool -n metallb-system

# Test connectivity
ping <ip>
nc -vz <ip> <port>
traceroute <ip>
ip route

# ─── KEY YAML SNIPPETS ────────────────────────────────────────
# LoadBalancer Service
spec:
  type: LoadBalancer
  ports:
  - port: 8554
    targetPort: 8554
  selector:
    name: my-app

# Multus annotation on pod
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: custom

# NetworkAttachmentDefinition (host-device, static IP)
spec:
  config: |-
    {
      "cniVersion": "0.3.1",
      "name": "custom",
      "type": "host-device",
      "device": "ens4",
      "ipam": {
        "type": "static",
        "addresses": [{"address": "192.168.51.10/24"}]
      }
    }
```

---

## Summary Table — Most Asked Interview Points

| Topic | Key Point |
|---|---|
| Ingress / Route | Only for HTTP/HTTPS. Cannot expose TCP services. |
| LoadBalancer service | Gives dedicated external IP. Needs MetalLB on bare metal. |
| MetalLB | For clusters without cloud provider. Uses IPAddressPool for IP ranges. |
| `<pending>` IP | IP pool is full. Delete a service or expand the pool. |
| Multus | Gives pods extra network interfaces (net1, net2...). |
| NetworkAttachmentDefinition | Defines how to connect pod to secondary network. Namespaced. |
| host-device type | Exclusive — only ONE pod can use the NIC at a time. |
| IPAM static | Fixed IP. Good for databases, predictable services. |
| IPAM dhcp | Dynamic IP from DHCP server. |
| MACVLAN vs IPVLAN | MACVLAN = own MAC per pod. IPVLAN = shared MAC, fewer MACs used. |
| Pod annotation | `k8s.v1.cni.cncf.io/networks: <nad-name>` to join secondary network. |
| Network status annotation | Auto-filled by Multus. Shows all interfaces and IPs. |
| Security benefit of Multus | Only machines on the same physical network can access the pod. |

---

*28 Questions | OpenShift Chapter 5 — Expose Non-HTTP/SNI Applications*
*Easy English | 8+ Years Experience Level | Generated: 2026-06-29*
