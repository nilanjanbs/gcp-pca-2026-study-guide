# ☁️ GCP Networking — PCA 2026 Exam Study Guide
> **Target:** Google Cloud Professional Cloud Architect (PCA) Exam — 2026  
> **Domain:** Networking  
> **Topics:** VPC, Cloud VPN, Cloud Interconnect, Cloud Load Balancing, NEGs

---

## Table of Contents
1. [Cloud VPC — Introduction](#1-cloud-vpc--introduction)
2. [Cloud VPC — IP Addresses](#2-cloud-vpc--ip-addresses)
3. [Cloud VPC — Subnet Creation Tips](#3-cloud-vpc--subnet-creation-tips)
4. [Cloud VPC — IP Range Overlap](#4-cloud-vpc--ip-range-overlap)
5. [Cloud VPC — Connecting to Other Networks](#5-cloud-vpc--connecting-to-other-networks)
6. [Cloud VPC — Project Isolation Techniques](#6-cloud-vpc--project-isolation-techniques)
7. [Cloud VPC — Private Google Access](#7-cloud-vpc--private-google-access)
8. [Cloud VPC — Serverless VPC Access](#8-cloud-vpc--serverless-vpc-access)
9. [Cloud VPC — Direct VPC Egress](#9-cloud-vpc--direct-vpc-egress)
10. [Cloud VPC — Predefined Roles](#10-cloud-vpc--predefined-roles)
11. [Cloud VPN — Introduction & Core Concepts](#11-cloud-vpn--introduction--core-concepts)
12. [Cloud VPN — Resolving Overlapping Networks](#12-cloud-vpn--resolving-overlapping-networks)
13. [Cloud VPN — Gateway Types](#13-cloud-vpn--gateway-types)
14. [Cloud VPN — Multi-Region Configurations](#14-cloud-vpn--multi-region-configurations)
15. [Cloud Interconnect — Introduction](#15-cloud-interconnect--introduction)
16. [Cloud Interconnect — Dedicated Interconnect Use Cases](#16-cloud-interconnect--dedicated-interconnect-use-cases)
17. [Cloud Interconnect — Avoiding IP Overlap](#17-cloud-interconnect--avoiding-ip-overlap)
18. [Cloud Interconnect — Colocation Facilities](#18-cloud-interconnect--colocation-facilities)
19. [Cloud Interconnect — High Availability Use Cases](#19-cloud-interconnect--high-availability-use-cases)
20. [Cloud Load Balancing — Introduction](#20-cloud-load-balancing--introduction)
21. [Cloud Load Balancing — OSI Layers](#21-cloud-load-balancing--osi-layers)
22. [Cloud Load Balancing — Regional vs. Global](#22-cloud-load-balancing--regional-vs-global)
23. [Cloud Load Balancing — Health Checks](#23-cloud-load-balancing--health-checks)
24. [Cloud Load Balancing — Session Affinity](#24-cloud-load-balancing--session-affinity)
25. [Cloud Load Balancing — IoT Use Case](#25-cloud-load-balancing--iot-use-case)
26. [Cloud Load Balancing — Network Endpoint Groups (NEGs)](#26-cloud-load-balancing--network-endpoint-groups-negs)

---

## 1. Cloud VPC — Introduction

### What is Cloud VPC?
A **Virtual Private Cloud (VPC)** in GCP is a **globally distributed, software-defined network** that provides connectivity for GCP resources. Unlike traditional on-prem networks or other clouds, GCP VPCs are **global by default** — a single VPC spans all regions.

### Key Characteristics

| Property | GCP VPC Behavior |
|----------|-----------------|
| **Scope** | Global — one VPC spans all regions |
| **Subnets** | Regional — each subnet exists in one region |
| **Routing** | Automatic within VPC; custom routes for on-prem |
| **Firewall** | Stateful, applied at VM-level (not subnet boundary) |
| **IPv4** | Supported (internal + external) |
| **IPv6** | Supported on dual-stack subnets (GA) |
| **Default VPC** | Auto-created per project — avoid in prod |

### VPC vs. Traditional Network

```
Traditional (on-prem):
    Region A Network ←→ Peering/Routing ←→ Region B Network

GCP VPC:
    One VPC
        ├── Subnet: northamerica-northeast1 (10.10.0.0/20)
        ├── Subnet: us-central1 (10.20.0.0/20)
        └── Subnet: europe-west1 (10.30.0.0/20)
    → VMs in different regions communicate via VPC internal routing automatically
```

### Auto Mode vs. Custom Mode VPC

| | Auto Mode | Custom Mode |
|--|-----------|------------|
| **Subnet creation** | Automatic (one per region, /20) | Manual — you define |
| **IP ranges** | Fixed predefined ranges | You choose |
| **Flexibility** | Low | High |
| **IP overlap risk** | High (if connecting to other networks) | Manageable |
| **Production use** | ❌ Avoid | ✅ Recommended |
| **Convertible** | Yes → Custom (one-way, no reverse) | No |

> 🎯 **PCA Exam Catch:** Auto mode VPC uses `10.128.0.0/9` range which overlaps with many on-prem and cloud networks. **Always use Custom mode** for production. This is a very common exam scenario — "company wants to connect on-prem to GCP but VPN fails" → answer involves auto mode overlap.

### Default VPC
- Auto-created in every new project
- Uses auto mode (problematic for hybrid connectivity)
- Pre-configured with firewall rules: `allow-internal`, `allow-ssh`, `allow-rdp`, `allow-icmp`
- **Disable with Org Policy:** `constraints/compute.skipDefaultNetworkCreation`

### Creating a Custom VPC

```bash
# Create custom mode VPC
gcloud compute networks create prod-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# Create subnet in the VPC
gcloud compute networks subnets create prod-subnet-ca \
  --network=prod-vpc \
  --region=northamerica-northeast1 \
  --range=10.10.0.0/20
```

---

## 2. Cloud VPC — IP Addresses

### Types of IP Addresses in GCP

| Type | Description | Persistence |
|------|-------------|-------------|
| **Internal (Private) IPv4** | RFC 1918 addresses from subnet range | Ephemeral or static |
| **External (Public) IPv4** | Routable internet addresses | Ephemeral or static (reserved) |
| **Internal IPv6** | ULA range from subnet | Assigned per subnet |
| **External IPv6** | Global unicast IPv6 | Assigned per subnet |
| **Alias IP** | Secondary IP ranges on a NIC | Used for GKE pods |

### Ephemeral vs. Static IP

| | Ephemeral | Static (Reserved) |
|--|-----------|------------------|
| **Persistence** | Released when VM stops | Persists until explicitly released |
| **Cost** | Free (when in use) | Charged when reserved but unattached |
| **Use case** | Stateless workloads | Load balancer frontends, NAT, VPN gateways |
| **Assignment** | Automatic | Manual reservation then assignment |

```bash
# Reserve a static external IP
gcloud compute addresses create prod-lb-ip \
  --region=northamerica-northeast1 \
  --ip-version=IPV4

# Reserve global static IP (for global LB)
gcloud compute addresses create prod-global-ip \
  --global \
  --ip-version=IPV4

# List reserved IPs
gcloud compute addresses list
```

### Internal IP Assignment

- Each VM gets a **primary internal IP** from its subnet range
- **Alias IP ranges** allow VMs to own additional internal IPs (used by GKE for pod IPs)
- Internal IPs are **persistent across reboots** (released only when VM is deleted or NIC changed)

### GKE and Alias IPs

```
GKE Node (VM):
    Primary IP: 10.10.0.5/20  (from node subnet)
    Alias IP range: 10.100.0.0/24  (pod CIDR — from secondary range)
        ├── Pod 1: 10.100.0.1
        ├── Pod 2: 10.100.0.2
        └── Pod N: 10.100.0.N
```

> 🎯 **PCA Exam Catch:** GKE requires **secondary IP ranges** on subnets for pod and service CIDRs. Not planning these ranges leads to exhaustion as clusters grow. Always pre-plan pod and service CIDR sizes.

### IPv6 on GCP VPC
- **Dual-stack subnets** support both IPv4 and IPv6
- External IPv6: Global unicast `/96` assigned per subnet
- Internal IPv6: ULA range from `fd20::/20`
- GKE supports dual-stack clusters (IPv4 + IPv6)

---

## 3. Cloud VPC — Subnet Creation Tips

### Subnet Design Principles

```
1. Size appropriately — subnets can EXPAND but not shrink
2. Plan for GKE secondary ranges upfront
3. Avoid IP ranges that overlap with on-prem or other clouds
4. Use consistent CIDR conventions across environments
5. Consider future growth (VPC peering, Shared VPC, hybrid)
```

### Subnet Expansion
Subnets can be expanded (CIDR widened) but **never shrunk**.

```bash
# Expand subnet from /24 to /20
gcloud compute networks subnets expand-ip-range prod-subnet-ca \
  --region=northamerica-northeast1 \
  --prefix-length=20
```

> 🎯 **PCA Exam Catch:** You **cannot reduce** a subnet's IP range once set. If you need a smaller range, you must delete and recreate (with service disruption). Always plan subnets larger than current needs.

### Secondary Ranges (for GKE)

```bash
# Create subnet with primary range + two secondary ranges
gcloud compute networks subnets create gke-subnet \
  --network=prod-vpc \
  --region=northamerica-northeast1 \
  --range=10.10.0.0/20 \
  --secondary-range=pod-range=10.100.0.0/16 \
  --secondary-range=svc-range=10.200.0.0/20
```

### GKE IP Planning Formula

```
Nodes:     Primary subnet range    (e.g., /20 = 4094 node IPs)
Pods:      Secondary pod range     (e.g., /16 = 65536 pod IPs, 110 per node by default)
Services:  Secondary service range (e.g., /20 = 4094 service cluster IPs)

For 100 nodes × 110 pods each = 11,000 pod IPs needed → /20 secondary is NOT enough → use /16
```

### Private Google Access on Subnets

```bash
# Enable Private Google Access on subnet
gcloud compute networks subnets update prod-subnet-ca \
  --region=northamerica-northeast1 \
  --enable-private-ip-google-access
```

### Subnet Scope Rules

| Rule | Detail |
|------|--------|
| A subnet is regional | Cannot span regions |
| A VM's primary NIC must be in a subnet | True |
| A VM can have multiple NICs | Yes — max 8 NICs, each in a different VPC |
| Subnets in same VPC can communicate | Yes — via VPC routing automatically |
| Subnets across VPCs cannot communicate | Requires VPC Peering or Shared VPC |

---

## 4. Cloud VPC — IP Range Overlap

### Why Overlap Matters
IP range overlap prevents **VPC Peering**, **Cloud VPN**, and **Cloud Interconnect** from working correctly. When two networks have overlapping CIDR ranges, routing becomes ambiguous and GCP will reject the peering or routing configuration.

### Overlap Scenarios

```
Scenario 1 — VPC Peering Rejection:
    VPC-A: 10.0.0.0/8
    VPC-B: 10.1.0.0/16   ← overlaps with VPC-A
    → Peering CANNOT be established

Scenario 2 — VPN Routing Conflict:
    GCP VPC subnet: 192.168.1.0/24
    On-prem network: 192.168.1.0/24  ← same range
    → Traffic never leaves GCP — routed locally

Scenario 3 — Auto Mode VPC Problem:
    Auto mode VPC uses 10.128.0.0/9
    On-prem uses any 10.x.x.x range → highly likely overlap
```

### RFC 1918 Private Ranges (Exam Must-Know)

```
10.0.0.0/8       → 16.7M addresses (largest, most common for GCP)
172.16.0.0/12    → 1M addresses
192.168.0.0/16   → 65K addresses (common for on-prem/home networks)
```

### Detecting Overlap

```bash
# Check all subnet ranges in a VPC
gcloud compute networks subnets list \
  --filter="network:prod-vpc" \
  --format="table(name,region,ipCidrRange,secondaryIpRanges)"
```

### Overlap Prevention Strategy

```
Org-wide CIDR Allocation Plan:

Production VPCs:    10.0.0.0/12   (10.0.0.0 – 10.15.255.255)
Staging VPCs:       10.16.0.0/12  (10.16.0.0 – 10.31.255.255)
Development VPCs:   10.32.0.0/12  (10.32.0.0 – 10.47.255.255)
On-premises:        172.16.0.0/12 (separate RFC range entirely)

Within Production:
    Identity Platform:  10.0.0.0/16
    Payments:           10.1.0.0/16
    Shared VPC:         10.2.0.0/16
```

> 🎯 **PCA Exam Catch:** A question will describe a company that "cannot establish VPC peering" or "VPN tunnel is up but traffic doesn't flow." The root cause is almost always IP overlap. The fix: **redesign subnets** (expand one network, re-IP the other) or use a **NVA (Network Virtual Appliance)** to NAT between overlapping ranges.

### Resolving Overlap Without Re-IPing
When re-IPing is not feasible, use **network address translation at the boundary**:
- Deploy a **Cloud Router with custom NAT** or a VM-based NAT/firewall appliance
- This is complex and should be flagged as a design flaw — not recommended as permanent

---

## 5. Cloud VPC — Connecting to Other Networks

### Connection Options Overview

| Method | Use Case | Bandwidth | Latency | SLA |
|--------|---------|-----------|---------|-----|
| **Cloud VPN (HA VPN)** | On-prem/other cloud, encrypted | Up to 3 Gbps per tunnel | Variable (internet) | 99.99% |
| **Dedicated Interconnect** | High bandwidth, private, on-prem DC | 10 Gbps or 100 Gbps per link | Low, consistent | 99.99% |
| **Partner Interconnect** | Dedicated via service provider | 50 Mbps – 50 Gbps | Low | 99.9% / 99.99% |
| **Cross-Cloud Interconnect** | GCP ↔ AWS/Azure directly | 10 Gbps or 100 Gbps | Low | 99.99% |
| **VPC Peering** | GCP project ↔ GCP project | VPC-native | Very low | N/A |
| **Shared VPC** | Within org, shared network | VPC-native | Very low | N/A |

### Decision Framework

```
Need to connect on-prem to GCP?
    → < 1.5 Gbps, cost-sensitive, encrypted:  Cloud VPN (HA VPN)
    → > 1 Gbps, consistent, private:           Dedicated Interconnect
    → No colocation nearby:                     Partner Interconnect
    → Multi-cloud (AWS/Azure ↔ GCP):           Cross-Cloud Interconnect

Need to connect GCP projects?
    → Same org, share a network:               Shared VPC (preferred)
    → Different orgs or shared services:        VPC Peering
    → Transitive routing needed:               Cannot use Peering — use Shared VPC or NVA
```

### Transitive Routing Limitation

```
VPC-A ←peered→ VPC-B ←peered→ VPC-C

VPC-A CANNOT reach VPC-C via VPC-B (no transitive peering)

Solution: 
    Option 1 — Peer VPC-A ↔ VPC-C directly
    Option 2 — Use Shared VPC (all projects share one host VPC)
    Option 3 — Deploy NVA in VPC-B to route between A and C
```

> 🎯 **PCA Exam Catch:** VPC Peering is **non-transitive**. If the exam describes 3+ VPCs needing full connectivity, the answer is **Shared VPC** or direct peering between all pairs, NOT a chain of peerings.

---

## 6. Cloud VPC — Project Isolation Techniques

### Why Isolate at the Project Level?
Each GCP project has its own VPC, IAM boundary, quotas, and billing. Project isolation is GCP's primary security boundary.

### Isolation Techniques Compared

| Technique | Isolation Level | Network Sharing | Use Case |
|-----------|----------------|----------------|---------|
| **Separate Projects** | Full (billing, IAM, quotas) | None by default | Env separation (dev/staging/prod) |
| **Shared VPC** | IAM-level | Shared network (subnets) | Multi-project, shared network admin |
| **VPC Peering** | Full project isolation | Selective cross-VPC routes | Shared services, partner projects |
| **GKE Namespaces** | App-level | Same cluster network | Multi-tenant within one cluster |
| **VPC Service Controls** | API-level data perimeter | Prevents data exfil | Regulated data (BQ, GCS, Spanner) |

### Shared VPC Architecture

```
Host Project: shared-vpc-host-prod
    └── VPC: prod-shared-vpc
            ├── Subnet: prod-gke-subnet     (shared with identity-prod)
            ├── Subnet: prod-backend-subnet  (shared with payments-prod)
            └── Subnet: prod-data-subnet     (shared with data-prod)

Service Projects:
    ├── identity-platform-prod   → uses prod-gke-subnet
    ├── payments-prod            → uses prod-backend-subnet
    └── data-platform-prod       → uses prod-data-subnet
```

```bash
# Enable Shared VPC on host project
gcloud compute shared-vpc enable HOST_PROJECT_ID

# Attach service project to host
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
  --host-project=HOST_PROJECT_ID

# Grant subnet-level IAM to service project SA
gcloud compute networks subnets add-iam-policy-binding prod-gke-subnet \
  --region=northamerica-northeast1 \
  --member="serviceAccount:SERVICE_PROJECT_SA" \
  --role="roles/compute.networkUser" \
  --project=HOST_PROJECT_ID
```

### Shared VPC IAM Roles

| Role | Who Needs It | Purpose |
|------|-------------|---------|
| `roles/compute.xpnAdmin` | Host project admin | Enable/configure Shared VPC |
| `roles/compute.networkUser` | Service project SAs, users | Use subnets from host project |
| `roles/compute.networkViewer` | Operators | Read network config |
| `roles/compute.securityAdmin` | Firewall managers | Manage firewall rules in host |

> 🎯 **PCA Exam Catch:** In Shared VPC, **firewall rules live in the host project**, not service projects. Network admins manage firewalls centrally. Service project owners cannot bypass host-project firewall rules.

### VPC Service Controls (Complement to VPC Isolation)
Creates an **API-level security perimeter** around GCP managed services:
- Even if someone has IAM access, VPC-SC can block API calls from outside the perimeter
- Prevents data exfiltration from BigQuery, GCS, Spanner, etc.
- Works alongside VPC network isolation — both layers apply

---

## 7. Cloud VPC — Private Google Access

### What is Private Google Access (PGA)?
PGA allows VM instances **without external IP addresses** to reach **Google APIs and services** (e.g., GCS, BigQuery, PubSub, Secret Manager) using Google's internal network — without traversing the public internet.

### How PGA Works

```
VM (no external IP, only internal: 10.10.0.5)
    → Sends request to storage.googleapis.com (142.250.x.x)
    → VPC routing intercepts — detects it's a Google API address
    → Routes via Google's internal infrastructure
    → No internet egress, no NAT required
```

### Enabling PGA

```bash
# Enable on a subnet
gcloud compute networks subnets update prod-subnet-ca \
  --region=northamerica-northeast1 \
  --enable-private-ip-google-access

# Verify
gcloud compute networks subnets describe prod-subnet-ca \
  --region=northamerica-northeast1 \
  --format="get(privateIpGoogleAccess)"
```

### Private Google Access Variants

| Variant | Accesses | Use Case |
|---------|---------|---------|
| **Private Google Access** | Google APIs (GCS, BQ, etc.) | VMs without external IP |
| **Private Google Access for on-prem** | Google APIs from on-prem via VPN/Interconnect | Hybrid workloads |
| **Private Service Connect** | Google APIs via private endpoint IP | Custom IP, DNS control |
| **VPC-SC + PGA** | Google APIs within a security perimeter | Regulated environments |

### Private Service Connect (PSC)
More advanced than PGA — creates a **private endpoint** with a custom internal IP for Google APIs:

```bash
# Create PSC endpoint for Google APIs
gcloud compute addresses create psc-api-endpoint \
  --global \
  --purpose=PRIVATE_SERVICE_CONNECT \
  --addresses=10.10.10.10 \
  --network=prod-vpc

gcloud compute forwarding-rules create psc-api-rule \
  --global \
  --network=prod-vpc \
  --address=psc-api-endpoint \
  --target-google-apis-bundle=all-apis
```

> 🎯 **PCA Exam Catch:** PGA does NOT require a Cloud NAT. It's entirely separate. If a VM needs to reach the internet (e.g., apt-get updates), it needs **Cloud NAT**. If it only needs Google APIs (GCS, BQ), it only needs **PGA**. These are different solutions for different problems.

### PGA vs. Cloud NAT vs. External IP

| | PGA | Cloud NAT | External IP |
|--|-----|-----------|-------------|
| **Reaches Google APIs** | ✅ Yes | ✅ Yes (via internet) | ✅ Yes |
| **Reaches internet** | ❌ No | ✅ Yes | ✅ Yes |
| **Security** | Best | Good | Less secure |
| **Cost** | Free | Charged per hour + data | Charged when idle |
| **On-prem reachable** | Requires VPN/IC config | No | Yes |

---

## 8. Cloud VPC — Serverless VPC Access

### What is Serverless VPC Access?
A managed service that allows **serverless products** (Cloud Run, Cloud Functions, App Engine Standard) to connect to **internal VPC resources** (private VMs, Cloud SQL private IP, Memorystore, GKE internal services) without exposing those resources publicly.

### Serverless Products That Use It

| Product | VPC Access Method |
|---------|-----------------|
| Cloud Run (fully managed) | Serverless VPC Access connector OR Direct VPC Egress |
| Cloud Functions (1st/2nd gen) | Serverless VPC Access connector |
| App Engine Standard | Serverless VPC Access connector |
| App Engine Flexible | Has VPC access natively (in VPC) |

### How Serverless VPC Access Works

```
Cloud Run service (serverless, no VPC)
    → Traffic to 10.10.0.x (private IP)
    → Routed through VPC Access Connector (a managed VM group in your VPC)
    → Connector forwards traffic into your VPC
    → Reaches private Cloud SQL, Memorystore, internal VM
```

### Creating a VPC Access Connector

```bash
# Create connector
gcloud compute networks vpc-access connectors create prod-connector \
  --region=northamerica-northeast1 \
  --network=prod-vpc \
  --range=10.8.0.0/28 \
  --min-instances=2 \
  --max-instances=10 \
  --machine-type=e2-standard-4

# Attach to Cloud Run service
gcloud run services update my-service \
  --region=northamerica-northeast1 \
  --vpc-connector=prod-connector \
  --vpc-egress=private-ranges-only
```

### VPC Egress Settings

| Setting | Traffic Routed Through VPC |
|---------|--------------------------|
| `private-ranges-only` | Only RFC 1918 + internal traffic | ← Default, recommended
| `all-traffic` | All outbound traffic (including internet) |

---

## 9. Cloud VPC — Direct VPC Egress

### What is Direct VPC Egress?
A newer (GA 2024) capability for **Cloud Run** that allows services to connect directly to a VPC **without a VPC Access Connector**. The Cloud Run service gets a **network interface directly in your VPC subnet**.

### Direct VPC Egress vs. VPC Access Connector

| | VPC Access Connector | Direct VPC Egress |
|--|---------------------|------------------|
| **Architecture** | Managed connector VMs in between | Cloud Run gets NIC in your subnet |
| **Cost** | Connector VMs billed continuously | Only billed when requests run |
| **Throughput** | Up to 1 Gbps per connector | Higher, direct |
| **IP usage** | Connector CIDR (/28 min) | IPs from your subnet |
| **Setup** | Requires connector resource | Configured directly on Cloud Run |
| **Availability** | All regions | Growing region support |

```bash
# Configure Cloud Run with Direct VPC Egress
gcloud run services update my-service \
  --region=northamerica-northeast1 \
  --network=prod-vpc \
  --subnet=prod-subnet-ca \
  --vpc-egress=all-traffic
```

> 🎯 **PCA Exam Catch:** For new Cloud Run deployments needing VPC access, **Direct VPC Egress is preferred** over Connector for cost and performance. However, for Cloud Functions and App Engine Standard, **VPC Access Connector is still required**. Know which product uses which method.

---

## 10. Cloud VPC — Predefined Roles

### Key VPC IAM Roles

| Role | Purpose | Typical Assignee |
|------|---------|-----------------|
| `roles/compute.networkAdmin` | Full VPC management: create networks, subnets, firewalls, routes | Network/Platform team |
| `roles/compute.networkUser` | Use (attach to) subnets, not manage them | Service project SAs, developers |
| `roles/compute.networkViewer` | Read-only view of network resources | Auditors, SRE read-only |
| `roles/compute.securityAdmin` | Manage firewall rules and SSL certs | Security team |
| `roles/compute.xpnAdmin` | Enable and manage Shared VPC | Org/Platform admin |
| `roles/vpcaccess.admin` | Manage VPC Access connectors | Platform team |
| `roles/vpcaccess.user` | Use VPC Access connectors | Cloud Run/Functions SAs |

### Shared VPC Role Assignment Pattern

```
Host Project:
    Network Admin group → roles/compute.networkAdmin
    Security team      → roles/compute.securityAdmin

Service Projects:
    App SA             → roles/compute.networkUser (on specific subnets in host)
    GKE SA             → roles/compute.networkUser (on GKE subnet in host)
```

---

## 11. Cloud VPN — Introduction & Core Concepts

### What is Cloud VPN?
Cloud VPN creates an **encrypted IPsec tunnel** between your GCP VPC and an external network (on-prem or another cloud) over the public internet.

### VPN Types

| Type | Tunnels | SLA | Dynamic Routing | Use Case |
|------|---------|-----|----------------|---------|
| **HA VPN** | 2 interfaces, 4 tunnels | 99.99% | Required (BGP) | Production |
| **Classic VPN** | 1 interface | 99.9% | Static or dynamic | Legacy — avoid for new |

> 🎯 **PCA Exam Catch:** **Always recommend HA VPN** for new architectures. Classic VPN is legacy and only gets 99.9% SLA. The exam will test whether you know the difference and when each is appropriate.

### HA VPN Architecture

```
GCP VPC
    └── HA VPN Gateway (2 external IPs — Interface 0 and Interface 1)
            ├── Tunnel 1 → On-prem VPN Gateway Interface 0
            ├── Tunnel 2 → On-prem VPN Gateway Interface 0 (redundant)
            ├── Tunnel 3 → On-prem VPN Gateway Interface 1
            └── Tunnel 4 → On-prem VPN Gateway Interface 1 (redundant)

→ 4 tunnels total for 99.99% SLA
→ Cloud Router + BGP on each tunnel for dynamic routing
```

### Creating HA VPN

```bash
# Create HA VPN gateway
gcloud compute vpn-gateways create prod-ha-vpn-gw \
  --network=prod-vpc \
  --region=northamerica-northeast1

# Create Cloud Router (BGP)
gcloud compute routers create prod-cloud-router \
  --network=prod-vpc \
  --region=northamerica-northeast1 \
  --asn=65001

# Create external VPN gateway (peer)
gcloud compute external-vpn-gateways create on-prem-gw \
  --interfaces=0=PEER_IP_0,1=PEER_IP_1

# Create tunnels
gcloud compute vpn-tunnels create tunnel-1 \
  --peer-external-gateway=on-prem-gw \
  --peer-external-gateway-interface=0 \
  --region=northamerica-northeast1 \
  --ike-version=2 \
  --shared-secret=SECRET \
  --router=prod-cloud-router \
  --vpn-gateway=prod-ha-vpn-gw \
  --vpn-gateway-interface=0
```

### BGP and Cloud Router

Cloud Router uses **BGP (Border Gateway Protocol)** to dynamically exchange routes between GCP VPC and on-prem:
- Learns on-prem routes → installs in VPC routing table
- Advertises GCP subnet routes → on-prem learns them
- Route failover is **automatic** when BGP session drops

```
BGP Session:
    GCP ASN:    65001
    On-prem ASN: 65002
    GCP BGP IP:  169.254.0.1/30
    Peer BGP IP: 169.254.0.2/30
    (Link-local IPs, always 169.254.x.x for VPN BGP)
```

> 🎯 **PCA Exam Catch:** BGP uses **link-local IPs (169.254.x.x)** for the session, not subnet IPs. If the exam asks about BGP peer IPs in VPN, the answer is always in the `169.254.0.0/16` range.

### VPN Bandwidth
- Each tunnel: up to **3 Gbps** (aggregate)
- Multiple tunnels = ECMP (Equal Cost Multi-Path) load balancing
- For > 3 Gbps sustained: use **Dedicated Interconnect** instead

---

## 12. Cloud VPN — Resolving Overlapping Networks

### The Problem
VPN tunnels carry routes between networks. If the GCP subnet range **overlaps** with the on-prem range, GCP routes traffic locally instead of through the tunnel.

### Detection

```bash
# List VPN tunnel routes
gcloud compute routes list \
  --filter="network=prod-vpc" \
  --format="table(name,destRange,nextHopVpnTunnel)"

# Check BGP advertised routes
gcloud compute routers get-status prod-cloud-router \
  --region=northamerica-northeast1
```

### Resolution Options

| Option | When to Use | Complexity |
|--------|------------|------------|
| **Re-IP one network** | At design time, before deployment | Low (if early enough) |
| **Use NAT at VPN boundary** | Post-deployment, can't re-IP | High |
| **Policy-based routing** | Specific traffic only | Medium |
| **NVA (Network Virtual Appliance)** | Complex NAT requirements | High |

### Best Practice: Pre-Plan CIDRs
```
Before creating any VPN:
1. Document on-prem IP ranges (all of them, including planned)
2. Document GCP ranges (current + planned growth + GKE pod/svc CIDRs)
3. Ensure zero overlap
4. Document in a CMDB or IPAM tool (e.g., NetBox, Infoblox)
```

---

## 13. Cloud VPN — Gateway Types

### GCP VPN Gateway Types

| Gateway Type | Used For | Interface Count |
|-------------|---------|----------------|
| **HA VPN Gateway** | New GCP-side HA VPN deployments | 2 interfaces |
| **Classic VPN Gateway** | Legacy; being deprecated | 1 interface |
| **External VPN Gateway** | Represents your on-prem/peer device | 1–4 interfaces |

### Peer Gateway Types (On-Prem/Other Cloud)

| Scenario | External Gateway Interfaces | Tunnels Needed |
|----------|---------------------------|---------------|
| Single peer device, 1 IP | 1 | 2 (for HA) |
| Single peer device, 2 IPs | 2 | 4 (for 99.99%) |
| Two peer devices, 1 IP each | 2 | 4 (for 99.99%) |
| AWS VGW (Virtual Private Gateway) | 2 | 4 |

### HA VPN to AWS

```
AWS VGW (Virtual Private Gateway)
    ├── Tunnel A: IP 1.2.3.4  ←→ GCP HA VPN Interface 0
    ├── Tunnel B: IP 5.6.7.8  ←→ GCP HA VPN Interface 0
    ├── Tunnel C: IP 1.2.3.4  ←→ GCP HA VPN Interface 1
    └── Tunnel D: IP 5.6.7.8  ←→ GCP HA VPN Interface 1
→ 4 tunnels, BGP on each, 99.99% SLA achievable
```

> 🎯 **PCA Exam Catch:** For 99.99% SLA on HA VPN, you need **4 tunnels** (2 from each HA VPN interface to 2 peer interfaces). Just 2 tunnels gives 99.9%. The exam will test this tunnel count.

---

## 14. Cloud VPN — Multi-Region Configurations

### Why Multi-Region VPN?
- Regional disaster recovery for VPN connectivity
- Minimize latency for on-prem users in different geographies
- Comply with data residency (separate VPN per region)

### Multi-Region HA VPN Architecture

```
On-Premises DC — Toronto
    └── VPN Gateway Toronto
            ├── Tunnels → GCP HA VPN Gateway (northamerica-northeast1)
            └── Tunnels → GCP HA VPN Gateway (northamerica-northeast2) [DR]

On-Premises DC — London
    └── VPN Gateway London
            └── Tunnels → GCP HA VPN Gateway (europe-west2)
```

```bash
# Cloud Router per region (one per VPN region)
gcloud compute routers create prod-router-ca \
  --network=prod-vpc \
  --region=northamerica-northeast1 \
  --asn=65001

gcloud compute routers create prod-router-ca-dr \
  --network=prod-vpc \
  --region=northamerica-northeast2 \
  --asn=65001
```

### Global vs. Regional Dynamic Routing

```bash
# Set VPC to global dynamic routing mode
# (Cloud Router learns routes globally, not just in its region)
gcloud compute networks update prod-vpc \
  --bgp-routing-mode=global
```

| Routing Mode | Behavior |
|-------------|---------|
| `regional` | Cloud Router only advertises/learns routes in its region |
| `global` | Cloud Router learns routes from all regions in the VPC |

> 🎯 **PCA Exam Catch:** If on-prem should reach GCP resources in **any region** via a single VPN in one region, set `--bgp-routing-mode=global`. With regional mode, on-prem can only reach the VPN gateway's local region subnets.

---

## 15. Cloud Interconnect — Introduction

### What is Cloud Interconnect?
Cloud Interconnect provides **high-bandwidth, low-latency, private** connections between your on-prem network and GCP — without using the public internet.

### Interconnect Types

| Type | Who Provides Physical Link | Bandwidth | Colocation Required |
|------|--------------------------|-----------|-------------------|
| **Dedicated Interconnect** | You (via GCP colocation) | 10 Gbps or 100 Gbps per link | ✅ Yes |
| **Partner Interconnect** | Service Provider | 50 Mbps – 50 Gbps | ❌ No |
| **Cross-Cloud Interconnect** | Google (GCP ↔ AWS/Azure) | 10 Gbps or 100 Gbps | ❌ No |

### When to Choose Each

```
Dedicated Interconnect:
    ✅ You have equipment in a Google colocation facility
    ✅ Need > 10 Gbps consistently
    ✅ Lowest latency, full control
    ✅ SLA: 99.99% (with redundant links)

Partner Interconnect:
    ✅ No colocation presence
    ✅ Need 50 Mbps – 10 Gbps
    ✅ Existing relationship with supported provider
    ✅ SLA: 99.9% (single) or 99.99% (redundant)

Cross-Cloud Interconnect:
    ✅ Multi-cloud architecture (GCP + AWS or Azure)
    ✅ Need high bandwidth between clouds
    ✅ Private connectivity, not over public internet
```

### Interconnect vs. VPN Decision

| Factor | HA VPN | Dedicated Interconnect |
|--------|--------|----------------------|
| **Cost** | Lower | Higher |
| **Bandwidth** | Up to ~3 Gbps per tunnel | 10–200 Gbps |
| **Latency** | Internet-variable | Consistent, low |
| **Encryption** | IPsec (built-in) | Not encrypted (add MACsec or VPN over IC) |
| **Setup time** | Hours | Weeks (physical provisioning) |
| **SLA** | 99.99% (HA VPN) | 99.99% (redundant) |

> 🎯 **PCA Exam Catch:** Dedicated Interconnect traffic is **NOT encrypted by default**. If the exam requirement includes data encryption in transit over Interconnect, the answer is to add **MACsec** (at Layer 2) or run **VPN over Interconnect** (IPsec tunnel carried over the IC link).

---

## 16. Cloud Interconnect — Dedicated Interconnect Use Cases

### Primary Use Cases

| Use Case | Why Dedicated Interconnect |
|---------|--------------------------|
| Large data migrations to GCP (PB-scale) | 100 Gbps link avoids internet throttling |
| Real-time data streaming on-prem → BigQuery | Consistent low latency, high throughput |
| Hybrid application with on-prem DB + GCP compute | Low-latency SQL queries across boundary |
| Disaster recovery with sync replication | Consistent bandwidth for Spanner/Bigtable sync |
| Financial services real-time trading systems | Sub-millisecond predictability, no jitter |
| Healthcare — DICOM imaging from on-prem PACS | Large file transfers, latency-sensitive |

### Dedicated Interconnect — Link Speeds

| Circuit Speed | Use Case |
|--------------|---------|
| 10 Gbps | Standard enterprise workloads |
| 100 Gbps | Bulk data migration, media/broadcast |

Multiple circuits can be **bundled (LACP)** for more bandwidth:
- 8× 10 Gbps = 80 Gbps aggregate
- 2× 100 Gbps = 200 Gbps aggregate

### VLAN Attachments (Interconnect Attachments)

A VLAN attachment (also called **interconnect attachment**) connects a Dedicated Interconnect circuit to a specific **VPC and Cloud Router**:

```bash
# Create VLAN attachment
gcloud compute interconnects attachments dedicated create prod-ic-attachment \
  --region=northamerica-northeast1 \
  --router=prod-cloud-router \
  --interconnect=prod-dedicated-ic \
  --bandwidth=BPS_10G \
  --vlan=100 \
  --candidate-subnets=169.254.0.0/29
```

---

## 17. Cloud Interconnect — Avoiding IP Overlap

### Same Problem as VPN — Bigger Consequences
With Dedicated Interconnect, IP overlap means high-bandwidth traffic is misrouted silently. The consequences are more severe because organizations assume the dedicated circuit is working correctly.

### IP Planning for Interconnect

```
Must ensure no overlap between:
1. All GCP VPC subnets (including secondary ranges for GKE)
2. On-prem network ranges (including all branch offices, DCs, VPNs)
3. Any peered VPCs
4. Cloud Router BGP link-local ranges (169.254.x.x)
5. Future planned expansion ranges
```

### BGP Summary Routes
Instead of advertising individual subnets, use **BGP route summarization** to reduce complexity and avoid route table explosion:

```
Instead of advertising: 
    10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24 ... 10.0.255.0/24

Advertise summary: 10.0.0.0/16
→ Fewer BGP routes, easier management, same effect
```

```bash
# Advertise custom range from Cloud Router
gcloud compute routers update-bgp-peer prod-cloud-router \
  --peer-name=on-prem-peer \
  --region=northamerica-northeast1 \
  --advertisement-mode=custom \
  --set-advertisement-ranges=10.0.0.0/16,10.10.0.0/16
```

---

## 18. Cloud Interconnect — Colocation Facilities

### What is Colocation?
Colocation (colo) is a data center where multiple organizations house their equipment. Google has **Dedicated Interconnect PoP (Point of Presence)** locations inside colocation facilities.

### How Dedicated Interconnect Requires Colocation

```
Your Equipment (in the colo)
    ↕ Cross-connect (physical cable in the colo — you arrange/pay)
Google's Equipment (at the colo PoP)
    ↕ Google's network
GCP Region (may be nearby or connected via Google's backbone)
```

- You do NOT need to be in the same building as the GCP region data center
- You DO need to be in a colo that has a Google Interconnect PoP
- If no nearby PoP → use **Partner Interconnect** (provider handles the physical link)

### Finding Colocation Facilities

```bash
# List available colocation facilities
gcloud compute interconnects locations list \
  --format="table(name,city,description)"
```

Key Canadian Interconnect locations:
- Markham, ON (near Toronto)
- Montreal, QC

### Colocation Checklist

| Item | Action |
|------|--------|
| PoP location selection | Choose colo with Google PoP nearest your DC |
| Cross-connect ordering | Order with the colo facility |
| LOA (Letter of Authorization) | Get from Google console after ordering IC |
| Circuit provisioning | 4–6 weeks typical lead time |
| Redundant links | Order 2+ circuits in separate PoP locations |

---

## 19. Cloud Interconnect — High Availability Use Cases

### HA Requirements for Interconnect

| SLA Level | Configuration Required |
|-----------|----------------------|
| 99.9% | 1 Dedicated IC link + 1 VLAN attachment |
| 99.99% | 2 IC links in **different metro areas** + 2 VLAN attachments |

### 99.99% HA Architecture

```
On-Premises DC
    ├── Router A → Cross-connect → Google PoP: Toronto-Markham
    │                                  └── VLAN Attachment 1 → Cloud Router → prod-vpc
    └── Router B → Cross-connect → Google PoP: Montreal
                                       └── VLAN Attachment 2 → Cloud Router → prod-vpc

→ Two different metropolitan PoP locations = protection against metro-level failure
→ Two different on-prem routers = protection against router failure
→ BGP active-active or active-passive routing
```

### Active-Active vs. Active-Passive Failover

| Mode | BGP Configuration | Traffic | Failover |
|------|-----------------|---------|---------|
| **Active-Active** | Equal MED/cost on both links | ECMP across both links | Automatic, seamless |
| **Active-Passive** | Higher MED on standby link | All on primary; standby idle | Automatic on primary failure |

```bash
# Set MED for active-passive (higher = less preferred)
gcloud compute routers update-bgp-peer prod-cloud-router \
  --peer-name=ic-peer-montreal \
  --region=northamerica-northeast1 \
  --advertised-route-priority=100   # Lower = more preferred (primary)

gcloud compute routers update-bgp-peer prod-cloud-router \
  --peer-name=ic-peer-toronto \
  --region=northamerica-northeast1 \
  --advertised-route-priority=200   # Higher = less preferred (standby)
```

> 🎯 **PCA Exam Catch:** For 99.99% Interconnect SLA, the two links **must be in different metropolitan areas** (different cities), not just different PoPs in the same city. Same-city redundancy only achieves 99.9%.

---

## 20. Cloud Load Balancing — Introduction

### What is Cloud Load Balancing?
Cloud Load Balancing is a **fully managed, software-defined** load balancing service that distributes traffic across backends globally or regionally. It is **not appliance-based** — no VMs to manage, auto-scales instantly.

### Key Differentiators from Traditional LB

| Traditional | GCP Cloud Load Balancing |
|-------------|--------------------------|
| Hardware appliances or LB VMs | Fully managed, Google's Andromeda SDN |
| Manual scaling | Instant auto-scale (single IP, global) |
| Regional only | Global single-IP (Anycast) |
| Separate SSL termination | Built-in SSL, managed certificates |
| Separate WAF | Integrated Cloud Armor (WAF + DDoS) |

### Load Balancer Types Overview

| LB Type | Layer | Global/Regional | Traffic Type |
|---------|-------|----------------|-------------|
| **Global External HTTP(S)** | L7 | Global | HTTP, HTTPS |
| **Regional External HTTP(S)** | L7 | Regional | HTTP, HTTPS |
| **External TCP Proxy** | L4 | Global | TCP (non-HTTP) |
| **External SSL Proxy** | L4 | Global | SSL/TLS |
| **External Network (TCP/UDP)** | L4 | Regional | TCP, UDP, ESP, ICMP |
| **Internal HTTP(S)** | L7 | Regional | HTTP, HTTPS (internal) |
| **Internal TCP/UDP** | L4 | Regional | TCP, UDP (internal) |
| **Internal TCP Proxy** | L4 | Regional | TCP (internal) |
| **Cross-Region Internal** | L7 | Global (internal) | HTTP, HTTPS (internal, multi-region) |

---

## 21. Cloud Load Balancing — OSI Layers

### Layer 4 vs. Layer 7 Load Balancing

| | Layer 4 (Transport) | Layer 7 (Application) |
|--|--------------------|--------------------|
| **Routes based on** | IP, TCP/UDP port | HTTP host, path, headers, cookies |
| **SSL termination** | Pass-through or terminate | Terminate and inspect |
| **Content awareness** | No | Yes |
| **Performance** | Higher (less processing) | Slightly more overhead |
| **Use cases** | Non-HTTP TCP/UDP, gaming, MQTT | Web apps, APIs, microservices |
| **GCP examples** | Network LB, TCP Proxy LB | HTTP(S) LB, Internal HTTP(S) LB |

### Why Layer Matters for Architecture Decisions

```
HTTPS web application → L7 (HTTP(S) LB)
    ✅ Path-based routing (/api → backend A, /static → Cloud Storage)
    ✅ Host-based routing (api.example.com → one backend, www.example.com → another)
    ✅ Header-based routing
    ✅ Cloud Armor WAF integration
    ✅ Managed SSL certificates

TCP game server (port 25565) → L4 (Network LB or TCP Proxy)
    ✅ Low overhead
    ✅ Preserves client IP (Network LB)
    ✅ Handles non-HTTP protocols

IoT MQTT (port 8883) → L4 (Network LB or Internal TCP/UDP)
    ✅ Protocol-agnostic
    ✅ Low latency

gRPC microservices → L7 (HTTP(S) LB with HTTP/2)
    ✅ HTTP/2 multiplexing
    ✅ gRPC health checks
```

> 🎯 **PCA Exam Catch:** **gRPC uses HTTP/2**, so it uses the **HTTP(S) Load Balancer** (L7), not a TCP LB. If you see a gRPC use case in the exam, always choose HTTP(S) LB.

---

## 22. Cloud Load Balancing — Regional vs. Global

### Global Load Balancers

- Use **Anycast IP** — single IP address, traffic routed to nearest Google PoP
- Google's network carries traffic from the PoP to the backend (Premium Tier networking)
- Backends can span **multiple regions** — automatic failover between regions

```
User in Tokyo → Anycast IP → Google Tokyo PoP
    → nearest healthy backend (asia-northeast1) → response
    → if asia-northeast1 backends fail → failover to us-central1
```

**Global LB products:**
- Global External HTTP(S) LB (classic and modern)
- External TCP Proxy LB
- External SSL Proxy LB
- Cross-Region Internal HTTP(S) LB

### Regional Load Balancers

- Single region — backends must be in the same region
- Use **regional (Premium or Standard Tier)** external IP
- Standard Tier networking: traffic uses internet routing (lower cost, higher latency)

**Regional LB products:**
- Regional External HTTP(S) LB
- External Network (TCP/UDP) LB
- Internal HTTP(S) LB
- Internal TCP/UDP LB
- Internal TCP Proxy LB

### Premium vs. Standard Network Tier

| | Premium Tier | Standard Tier |
|--|-------------|--------------|
| **Routing** | Google's backbone from PoP to backend | Internet routing to GCP region |
| **LB types** | Global LBs | Regional LBs only |
| **Latency** | Lower, consistent | Higher, variable |
| **Cost** | Higher | Lower |
| **Use case** | Production, global users | Cost-optimized, single-region |

### Path-Based and Host-Based Routing (URL Maps)

```yaml
# URL map structure
Host: api.example.com
    /v1/*    → backend-service: api-v1-backend
    /v2/*    → backend-service: api-v2-backend
    /static/* → backend-bucket: static-assets-bucket

Host: admin.example.com
    /*       → backend-service: admin-backend
```

```bash
# Create URL map with path rules
gcloud compute url-maps create prod-url-map \
  --default-service=default-backend-service

gcloud compute url-maps add-path-matcher prod-url-map \
  --path-matcher-name=api-paths \
  --default-service=api-v1-backend \
  --path-rules=/v2/*=api-v2-backend,/static/*=static-bucket
```

> 🎯 **PCA Exam Catch:** Path-based routing requires an **L7 (HTTP(S)) load balancer**. If a question asks to route `/api` to one backend and `/web` to another, the answer is always Global or Regional HTTP(S) LB with URL map rules — never a TCP/Network LB.

---

## 23. Cloud Load Balancing — Health Checks

### What are Health Checks?
Health checks probe backend instances/services to determine if they are healthy enough to receive traffic. Unhealthy backends are automatically removed from rotation.

### Health Check Types

| Type | Protocol | Use Case |
|------|---------|---------|
| HTTP | HTTP GET to a path | Web apps |
| HTTPS | HTTPS GET | Secure web apps |
| HTTP/2 | HTTP/2 GET | gRPC services |
| TCP | TCP connect | Non-HTTP services |
| SSL | SSL handshake | SSL services |
| gRPC | gRPC health protocol | gRPC microservices |

### Health Check Configuration

```bash
# Create HTTP health check
gcloud compute health-checks create http prod-http-hc \
  --port=8080 \
  --request-path=/healthz \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3
```

### Health Check Parameters

| Parameter | Description | Recommendation |
|-----------|-------------|----------------|
| `check-interval` | How often to probe | 10–30s (shorter = faster detection, more load) |
| `timeout` | How long to wait for response | < check-interval |
| `healthy-threshold` | Consecutive successes to mark healthy | 2 |
| `unhealthy-threshold` | Consecutive failures to mark unhealthy | 3 |
| `request-path` | HTTP path probed | `/healthz`, `/health`, `/_ah/health` |

### Health Check Firewall Rules
Health checks come from specific Google IP ranges. You MUST allow them:

```bash
# Allow GCP health check probes
gcloud compute firewall-rules create allow-health-checks \
  --network=prod-vpc \
  --allow=tcp:8080 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=backend-servers
```

> 🎯 **PCA Exam Catch:** Health checks come from **`130.211.0.0/22` and `35.191.0.0/16`**. If you don't add these firewall rules, health checks fail and backends are marked unhealthy — even if the app is running perfectly. This is one of the most common LB misconfiguration scenarios in the exam.

### Health Check vs. Autohealing (GCE MIG)

| | LB Health Check | MIG Autohealing |
|--|----------------|----------------|
| **Purpose** | Remove from LB traffic | Recreate failed VMs |
| **Scope** | Traffic routing | VM lifecycle |
| **Action on failure** | Stop sending traffic | Delete + recreate VM |
| **Both together?** | ✅ Yes — use both |

---

## 24. Cloud Load Balancing — Session Affinity

### What is Session Affinity?
Session affinity (also called **sticky sessions**) ensures that requests from the **same client** are consistently routed to the **same backend instance**.

### Session Affinity Types

| Type | Based On | Use Case |
|------|---------|---------|
| `NONE` | No affinity — pure round-robin | Stateless apps |
| `CLIENT_IP` | Client's IP address | Simple stickiness |
| `CLIENT_IP_PORT` | Client IP + source port | Finer stickiness |
| `GENERATED_COOKIE` | LB-generated cookie | Stateful web sessions |
| `HEADER_FIELD` | Custom HTTP header | Microservices with tenant header |
| `HTTP_COOKIE` | Named HTTP cookie | App-controlled sessions |

### Configuring Session Affinity

```bash
# Set cookie-based session affinity
gcloud compute backend-services update prod-backend \
  --global \
  --session-affinity=GENERATED_COOKIE \
  --affinity-cookie-ttl=3600
```

### Session Affinity Trade-offs

| Factor | No Affinity | With Affinity |
|--------|------------|---------------|
| **Load distribution** | Even | Uneven (hot backends possible) |
| **State handling** | Stateless required | Stateful apps possible |
| **Resilience** | Client retries to any backend | Client must re-establish if backend fails |
| **Scalability** | Better | Can limit scale-out benefit |

> 🎯 **PCA Exam Catch:** Session affinity is a **best-effort** feature in GCP — it is **not guaranteed**. If a backend is removed (failed health check, scale-in), existing sessions are redistributed. The **right architecture is stateless backends with session state in Memorystore (Redis)** — not session affinity as the primary solution.

### Recommended Pattern for Stateful Sessions

```
Client → HTTP(S) LB (no session affinity)
    → Backend (stateless app)
        → Memorystore (Redis) for session storage
            ← All backends read/write the same session store
```

---

## 25. Cloud Load Balancing — IoT Use Case

### IoT Traffic Characteristics
- **High connection count** (millions of devices)
- **Long-lived TCP connections** (MQTT over TCP/TLS)
- **Low-bandwidth, high-frequency** small messages
- **Geographically distributed** devices
- **Protocol variety:** MQTT (port 8883), CoAP (UDP), HTTPS

### GCP IoT Load Balancing Architecture

```
IoT Devices (global)
    ↓ MQTT over TLS (port 8883)
Global External TCP Proxy LB  ← Single Anycast IP worldwide
    ↓ (SSL termination at LB)
Backend: MQTT Broker pool (GCE MIG or GKE)
    ↓
Cloud Pub/Sub (message ingestion)
    ↓
Dataflow / BigQuery / Cloud IoT processing
```

### Why Global TCP Proxy LB for IoT MQTT

| Requirement | Why TCP Proxy LB |
|-------------|-----------------|
| Non-HTTP protocol (MQTT) | L4 TCP LB handles any TCP protocol |
| Global low-latency | Anycast IP routes to nearest PoP |
| TLS termination | TCP Proxy LB handles SSL/TLS |
| Millions of connections | GCP scales transparently |
| Client IP visibility | Use `--proxy-header=PROXY_V1` for PROXY protocol |

> 🎯 **PCA Exam Catch:** IoT with MQTT = **TCP Proxy LB** (not HTTP(S) LB). The exam will try to confuse you into choosing HTTP(S) LB. MQTT is not HTTP — use L4.

### Cloud IoT Core (Deprecated)
> Cloud IoT Core was deprecated and shut down in August 2023. Exam questions referencing it are outdated. Current recommended IoT architecture: use **MQTT brokers (HiveMQ, EMQX on GKE) + Pub/Sub** directly.

---

## 26. Cloud Load Balancing — Network Endpoint Groups (NEGs)

### What are NEGs?
Network Endpoint Groups are **collections of endpoints** (IP:port pairs) that serve as backends for Cloud Load Balancing. They provide more granular control than instance groups.

### NEG Types

| NEG Type | Endpoints | Use Case |
|---------|-----------|---------|
| **Zonal NEG** | GCE VMs or GKE pods (IP:port) | Container-native load balancing |
| **Internet NEG** | External IP:port | Routing to external backends |
| **Serverless NEG** | Cloud Run, Cloud Functions, App Engine | LB in front of serverless |
| **Private Service Connect NEG** | PSC endpoint | Accessing published services via PSC |
| **Hybrid Connectivity NEG** | On-prem endpoints (IP:port) | Extend LB to on-prem backends |

### Container-Native Load Balancing (Zonal NEG + GKE)

Traditional LB with GKE instance groups load balances at the **node level** — traffic goes to a node then iptables routes to a pod (double-hop):

```
Client → LB → GKE Node (iptables NAT) → Pod
```

With **Container-Native LB (Zonal NEG)**, the LB routes directly to **pod IPs**:

```
Client → LB → Pod (direct, no node NAT)
```

```bash
# GKE container-native LB via Kubernetes Service annotation
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP  # Not NodePort — NEG handles it
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

> 🎯 **PCA Exam Catch:** Container-native LB with NEGs requires **VPC-native (alias IP) GKE clusters** — not routes-based clusters. Always enable `--enable-ip-alias` when creating GKE clusters intended for container-native LB.

### Serverless NEG

Enables **HTTP(S) LB in front of Cloud Run**, Cloud Functions, or App Engine:

```bash
# Create serverless NEG for Cloud Run
gcloud compute network-endpoint-groups create cloud-run-neg \
  --region=northamerica-northeast1 \
  --network-endpoint-type=serverless \
  --cloud-run-service=my-cloud-run-service

# Create backend service using the NEG
gcloud compute backend-services create cloud-run-backend \
  --global \
  --load-balancing-scheme=EXTERNAL_MANAGED

gcloud compute backend-services add-backend cloud-run-backend \
  --global \
  --network-endpoint-group=cloud-run-neg \
  --network-endpoint-group-region=northamerica-northeast1
```

**Why use Serverless NEG instead of Cloud Run's built-in URL?**
- Centralized SSL termination and managed certificates
- **Cloud Armor** (WAF + DDoS) protection on Cloud Run
- **Path-based routing** across multiple Cloud Run services
- **CDN** (Cloud CDN) in front of Cloud Run
- Single ingress IP for multiple services

### Hybrid Connectivity NEG

Routes LB traffic to **on-prem or other cloud backends** over VPN or Interconnect:

```bash
# Create hybrid NEG with on-prem endpoint
gcloud compute network-endpoint-groups create hybrid-neg \
  --network-endpoint-type=NON_GCP_PRIVATE_IP_PORT \
  --zone=northamerica-northeast1-a \
  --network=prod-vpc

# Add on-prem endpoint
gcloud compute network-endpoint-groups update hybrid-neg \
  --zone=northamerica-northeast1-a \
  --add-endpoint="ip=172.16.10.5,port=8080"
```

### Internet NEG

Route LB traffic to **external (internet) backends** not on GCP:

```bash
# Create Internet NEG
gcloud compute network-endpoint-groups create ext-neg \
  --network-endpoint-type=INTERNET_FQDN_PORT \
  --global

# Add FQDN endpoint
gcloud compute network-endpoint-groups update ext-neg \
  --global \
  --add-endpoint="fqdn=api.thirdparty.com,port=443"
```

Use case: Migrate traffic gradually from on-prem/third-party to GCP using the same LB frontend.

---

## PCA Exam Quick-Reference: Networking Decision Framework

```
Q: On-prem connection method?
    < 1.5 Gbps, budget-sensitive → HA VPN
    > 1 Gbps, dedicated, colocation available → Dedicated Interconnect
    No colocation, need > VPN → Partner Interconnect
    AWS/Azure ↔ GCP → Cross-Cloud Interconnect

Q: VPC strategy?
    Always custom mode in production
    Disable default VPC via Org Policy (compute.skipDefaultNetworkCreation)
    Plan CIDRs upfront — subnets cannot shrink

Q: GKE networking?
    Alias IP (VPC-native) clusters always
    Plan pod CIDR (/16 minimum for large clusters)
    Container-native LB via Zonal NEG for direct pod routing

Q: Load balancer type?
    HTTP/HTTPS, global multi-region → Global External HTTP(S) LB
    HTTP/HTTPS, single region → Regional External HTTP(S) LB
    TCP non-HTTP, global → External TCP Proxy LB
    UDP/ESP/ICMP → External Network (TCP/UDP) LB (regional)
    Internal microservices → Internal HTTP(S) LB
    gRPC → HTTP(S) LB (HTTP/2)
    MQTT/IoT → TCP Proxy LB

Q: Serverless (Cloud Run) needs WAF / CDN / path routing?
    → Serverless NEG + Global HTTP(S) LB + Cloud Armor

Q: Private access to Google APIs without internet?
    VMs → Private Google Access (enable on subnet)
    Serverless → VPC Access Connector or Direct VPC Egress
    Custom IP/DNS → Private Service Connect

Q: VPN not routing traffic despite tunnel being UP?
    → IP range overlap — check subnet CIDRs vs. on-prem ranges

Q: Dedicated Interconnect not encrypted?
    → Add MACsec or run VPN-over-Interconnect
```

---

## PCA 2026 — High-Frequency Exam Traps: Networking

| Trap | Correct Answer |
|------|---------------|
| Auto mode VPC for hybrid connectivity | Never — use Custom mode |
| Classic VPN for new production deployments | Use HA VPN (99.99% SLA) |
| 2 tunnels for 99.99% HA VPN | Need 4 tunnels (2 per HA VPN interface) |
| Same-city Interconnect for 99.99% | Needs different metro areas |
| HTTP(S) LB for MQTT/IoT | TCP Proxy LB — MQTT is not HTTP |
| HTTP(S) LB for gRPC | ✅ Correct — gRPC is HTTP/2 |
| Session affinity as primary session solution | Use Memorystore (Redis) for session state |
| Health check fails despite app running | Add firewall rule for 130.211.0.0/22, 35.191.0.0/16 |
| Cloud Run needs WAF | Serverless NEG + HTTP(S) LB + Cloud Armor |
| Container-native LB on routes-based GKE | Requires VPC-native (alias IP) cluster |
| VPC peering for 3+ VPC full mesh | Use Shared VPC — peering is non-transitive |
| Dedicated Interconnect traffic is encrypted | Not by default — add MACsec or VPN-over-IC |
| PGA replaces Cloud NAT for internet access | PGA = Google APIs only; Cloud NAT = internet |
| bgp-routing-mode=regional for multi-region on-prem | Use global routing mode for on-prem to reach all regions |
| VPC Access Connector for new Cloud Run | Consider Direct VPC Egress (newer, cost-efficient) |

---

*Study Guide Version: 1.0 | Aligned to PCA Exam 2026 Syllabus | Domain: Networking*

# 📘 PCA Networking Enrichment Addendum (2026 Exam Focus)
> *Append this to your existing `gcp-networking-pca2026-study-guide.md`. All sections are mapped to PCA scoring domains, include 2026 exam updates, and highlight high-frequency test patterns.*

---

## 🔍 Missing & High-Yield Topics (PCA 2026)

### 1. Cloud DNS & Routing Policies
| Feature | Exam Focus | PCA Catch |
|--------|------------|-----------|
| **Zone Types** | Public, Private, Peering, Forwarding | Private zones override public zones for VPC-internal resolution |
| **Routing Policies** | Standard, Failover, Geo, Latency, Weighted, Primary/Backup | ❌ Never use DNS Failover for production HA. It doesn't check backend health. Use LB health checks instead. |
| **Split-Horizon DNS** | Same FQDN resolves differently internally vs externally | Use for zero-trust architectures: internal users → private LB IP, external → public LB IP |
| **DNS Peering** | Resolve queries across VPCs without internet | Requires peered VPC + authorization; non-transitive |

🎯 **Exam Decision Tree**: 
```
Need traffic routing + health checks + auto-failover? → Cloud Load Balancing
Need name resolution only? → Cloud DNS
Need geo/latency routing WITHOUT load balancer? → Cloud DNS Routing Policy (rarely optimal)
```

### 2. Cloud NAT & Egress Optimization
| Feature | Exam Focus | PCA Catch |
|--------|------------|-----------|
| **Architecture** | Regional, highly available, scales automatically | No single point of failure; managed by Google |
| **Endpoint Independent Mapping (EIM)** | Preserves client port across destinations | Required for symmetric protocols (some VPNs, gaming) |
| **Logging** | Enable via Cloud NAT log config | Use for egress auditing, compliance, anomaly detection |
| **Egress Cost Optimization** | Cloud NAT + Cloud CDN + Premium/Standard tier selection | Egress from GCP → Internet is billed. Use CDN to cache at edge, reducing origin egress |

💡 **Exam Tip**: Cloud NAT replaces legacy `nat-gw` VMs. Always recommend Cloud NAT for new designs. It supports IPv4 only; IPv6 egress requires external IPs or dual-stack LB.

### 3. Network Connectivity Center (NCC) — *New GA 2024/2025*
| Feature | Exam Focus | PCA Catch |
|--------|------------|-----------|
| **Hub-and-Spoke Model** | Central routing hub connects VPCs, on-prem, SD-WAN | Solves transitive routing limitation of VPC peering |
| **Inter-Region Routing** | Automatic route exchange across regions | Requires `bgp-routing-mode=global` on VPCs |
| **SD-WAN Integration** | Partner Interconnect + NCC for branch connectivity | Replaces complex VPN mesh architectures |
| **Route Aggregation** | Summarizes routes to prevent table explosion | Critical for large enterprises (>50 VPCs) |

🎯 **Exam Catch**: If a question describes "multiple VPCs need full mesh connectivity" or "transitive routing between on-prem and cloud", the answer is **NCC** or **Shared VPC**, NOT VPC peering chains.

### 4. Hierarchical Firewall Policies & Firewall Insights
| Feature | Exam Focus | PCA Catch |
|--------|------------|-----------|
| **Policy Scope** | Organization → Folder → Project → VPC | Evaluated top-down; lower priority number = higher precedence |
| **Rule Evaluation** | Allow/Deny with priority 0-65535 | First matching rule wins; implicit deny at end |
| **Firewall Insights** | AI-driven recommendations to remove unused rules | Use for least-privilege compliance audits |
| **Logging** | Enable per-rule for audit trails | Required for compliance (HIPAA, PCI, SOC2) |

⚠️ **Exam Trap**: Hierarchical firewall policies **override** VPC-level rules. If a question mentions "centralized firewall management across org", choose Hierarchical Policies + `roles/compute.securityAdmin`.

### 5. Private Service Connect (PSC) — Deep Dive
| Component | Exam Focus | PCA Catch |
|-----------|------------|-----------|
| **Consumer Endpoint** | Creates private IP in your VPC for external services | No internet/NAT required; DNS resolves to private IP |
| **Producer Service** | Publishes service (Google API, third-party SaaS, internal LB) | Requires forwarding rule + PSC service attachment |
| **DNS Integration** | Automatic resolution or custom private zone | Use `private-dns-zone` for seamless internal access |
| **Use Cases** | Vertex AI, BigQuery, third-party APIs, SaaS | Replaces VPC peering for managed services; avoids IP overlap |

🎯 **Exam Catch**: PSC ≠ PGA. PGA uses Google's internal routing for standard APIs. PSC gives you a **custom private IP** and DNS control for specific services. Use PSC when you need predictable IPs or third-party service integration.

### 6. Route Propagation & BGP Advertisement Modes
| Mode | Behavior | Exam Use |
|------|----------|----------|
| `default` | Advertises all subnet CIDRs in the region/VPC | Simple setups; may cause routing bloat |
| `custom` | Advertises only specified ranges via Cloud Router | Required for overlap avoidance, summarization, hybrid routing |
| **Route Priority** | Lower number = higher preference. BGP=100, Static=0-200 | Use static routes (priority 0-50) for override scenarios |

💡 **Exam Tip**: Always pair Cloud Router with `custom` advertisement mode in hybrid environments to control what on-prem sees. Prevents accidental route leaks.

---

## 🧩 Advanced PCA Decision Frameworks

### 🌐 Connectivity Choice Matrix
| Requirement | Recommended Solution | Why It Wins |
|-------------|----------------------|-------------|
| `<1.5 Gbps, encrypted, budget` | HA VPN (4 tunnels) | 99.99% SLA, IPsec, BGP dynamic routing |
| `>2 Gbps, low latency, colo` | Dedicated Interconnect + MACsec/VPN-over-IC | 10-200 Gbps, consistent, private |
| `Multi-cloud (AWS/Azure)` | Cross-Cloud Interconnect | Direct, private, 10/100 Gbps |
| `VPC-to-VPC, same org` | Shared VPC | Centralized network mgmt, no peering limits |
| `VPC-to-VPC, different orgs` | VPC Peering or NCC | Peering for simple; NCC for transitive/complex |
| `Managed service access` | Private Service Connect | Private IP, DNS control, no overlap risk |

### 📡 Traffic Management: LB vs DNS vs Routing
| Goal | Best Service | Limitation |
|------|--------------|------------|
| Path/host routing + health checks | Cloud HTTP(S) LB | L7 only; cannot route raw TCP/UDP by path |
| Geo/latency-based routing | Cloud DNS Routing Policies | No health checks; DNS TTL delays failover |
| Global anycast + auto-failover | Global External LB | Premium tier cost; requires backend in GCP |
| Internal microservice routing | Internal HTTP(S) LB + NEGs | Regional scope; requires VPC-native GKE |

### 💰 Egress Cost Optimization Checklist
- [ ] Cache static content at edge with **Cloud CDN**
- [ ] Use **Standard Tier** for single-region, non-critical workloads
- [ ] Route inter-region traffic via **VPC internal routing** (free within same network)
- [ ] Avoid egress to internet for internal API calls; use **PGA/PSC**
- [ ] Monitor egress with **Network Intelligence Center → Topology & Dashboards**

---

## ⚠️ 2026 Exam Traps & Anti-Patterns

| Trap in Question | Correct Approach | Why |
|------------------|------------------|-----|
| "Use DNS failover for HA web app" | Use Global HTTP(S) LB + health checks | DNS doesn't monitor backend health; TTL causes delayed failover |
| "VPC peering chain for 4 projects" | Use Shared VPC or NCC | Peering is non-transitive; chains break routing |
| "Cloud NAT for IPv6 egress" | Use dual-stack LB or external IPv6 | Cloud NAT is IPv4-only |
| "Static route priority 500 overrides BGP" | BGP routes use priority 100; static must be `<100` to win | Route priority is inverse: lower number = higher preference |
| "Hierarchical firewall deny at org level" | Org policies override project rules | Evaluate top-down; first match wins |
| "PGA for internet access" | Use Cloud NAT or external IP | PGA only reaches Google APIs/services |
| "Direct VPC egress for Cloud Functions" | Use Serverless VPC Access Connector | Direct egress is Cloud Run-only (2024+ GA) |

---

## 📝 How to Integrate This Addendum

1. **Append** the `Missing & High-Yield Topics` section after `26. Cloud Load Balancing — Network Endpoint Groups (NEGs)`.
2. **Insert** the `Advanced PCA Decision Frameworks` table before the `PCA Exam Quick-Reference` section.
3. **Replace** the existing `High-Frequency Exam Traps` table with the expanded `2026 Exam Traps & Anti-Patterns` table.
4. **Use** the `Egress Cost Optimization Checklist` and `Traffic Management` matrix as quick-reference during practice exams.
5. **Cross-Reference** with your security/compute guides: NCC ↔ Shared VPC, PSC ↔ IAM/Identity, Hierarchical Firewalls ↔ VPC-SC.

---

## ✅ Quick Reference Additions (Print-Friendly)

```
DNS vs LB: 
  DNS = Name resolution only. LB = Traffic routing + health checks + SSL/WAF.

NAT vs PGA vs PSC:
  NAT = Internet egress (IPv4)
  PGA  = Google APIs over internal routing
  PSC  = Custom private IP for managed/third-party services

Routing Priority:
  Lower number = Higher preference
  BGP default = 100 | Static = 0-200 | Hierarchical FW = evaluated top-down

VPC Peering Limits:
  Non-transitive | No overlapping CIDRs | Max 25 peerings per VPC
  → Use NCC or Shared VPC for complex topologies

Cloud NAT:
  Regional | Highly available | Endpoint Independent Mapping (default)
  Logs available | Replaces legacy nat-gw VMs

2026 Focus Areas:
  NCC hub-spoke | Direct VPC egress (Cloud Run) | Hierarchical FW policies
  PSC for AI/SaaS | IPv6 dual-stack GA | Egress cost optimization patterns
```

---
*This enrichment aligns with the official Google Cloud PCA exam guide, 2024-2026 service GA announcements, and real candidate feedback. Always validate architecture decisions against current [GCP Networking Documentation](https://cloud.google.com/networking).*
