# 🛡️ Domain 6: Reliability & WAF (Operational Excellence) (12%)
### Google Professional Cloud Architect 2026 | Deep-Dive Study Guide (6 of 6)

> **Exam Weight**: ~12% (~6 questions) | **Focus**: Building resilient, observable, and self-healing systems aligned with Well-Architected Framework (WAF) & SRE principles | **Key Skills**: HA vs. DR trade-offs, auto-scaling/healing, data consistency patterns, observability-driven reliability, recovery planning

---

## 📖 Table of Contents

1. [What the Exam Actually Tests](#1-what-the-exam-actually-tests)
2. [Compute Resilience & Autohealing](#2-compute-resilience--autohealing)
3. [Network Availability & Load Balancing](#3-network-availability--load-balancing)
4. [Container Health & Cloud Run HA Architecture](#4-container-health--cloud-run-ha-architecture)
5. [Recovery Planning & RPO/RTO Framework](#5-recovery-planning--rportto-framework)
6. [Data Protection, Failover & State Management](#6-data-protection-failover--state-management)
7. [Dynamic Scaling Patterns](#7-dynamic-scaling-patterns)
8. [Data Integrity Under Load](#8-data-integrity-under-load)
9. [Maintenance & Patching Strategies](#9-maintenance--patching-strategies)
10. [Observability for Reliability](#10-observability-for-reliability)
11. [Modulator's Deep-Dive Decision Frameworks](#11-modulators-deep-dive-decision-frameworks)
12. [Exam Traps & Tricks](#12-exam-traps--tricks)
13. [Mnemonics & Memory Hacks](#13-mnemonics--memory-hacks)
14. [Practice Checkpoint (10 Scenarios)](#14-practice-checkpoint-10-scenarios)
15. [2025–2026 Changes & Curated YouTube Playlist](#15-20252026-changes--curated-youtube-playlist)

---

## 1️⃣ What the Exam Actually Tests

### 🔍 The Real Skill Being Assessed
```
Domain 6 does NOT test:
❌ Memorizing exact timeout values for health checks or probes
❌ Step-by-step console UI paths for enabling backups or autoscaling
❌ Raw SRE formulas without architectural context

Domain 6 DOES test:
✅ Designing HA vs. DR architectures with clear RPO/RTO alignment and cost trade-offs
✅ Selecting appropriate load balancers (Global vs. Regional) based on traffic patterns and failover requirements
✅ Configuring container health probes (Liveness vs. Readiness) for GKE resilience
✅ Implementing auto-scaling patterns that balance cold starts, overprovisioning, and cost
✅ Handling data integrity challenges: out-of-order streams (Dataflow), hotspotting (Bigtable/Cloud Storage)
✅ Designing maintenance and patching strategies that minimize downtime (live migration, rolling updates)
✅ Applying WAF Reliability & Operational Excellence pillars through observability, SLOs, and automated recovery
✅ Choosing replication strategies (synchronous vs. asynchronous) based on consistency vs. latency requirements
```

---

## 2️⃣ Compute Resilience & Autohealing

### 🖥️ MIG Resilience Patterns
```yaml
MIG Location Strategy:
✅ Zonal MIG: Single zone, lower cost, tighter latency bounds
   → Use for: Stateless batch, dev/test, zone-redundant multi-MIG setups
   → Limitation: Zone failure = complete outage

✅ Regional MIG: Spans 3 zones in a region, automatic distribution
   → Use for: Production stateless workloads requiring high availability
   → Benefit: Survives single-zone failure without manual intervention

Autohealing & Automatic Restart:
✅ Autohealing: Health check detects unresponsive VM → deletes & recreates instance
   → Triggers: HTTP/TCP health check failure threshold exceeded
   → Best practice: Pair with startup scripts that rejoin clusters/restore state

✅ Automatic Restart: GCP automatically restarts VMs after host maintenance or hardware failure
   → Enabled by default; disable only for specific batch/stateless-ephemeral workloads
   → Does NOT survive zone-wide outages (requires multi-zone/region design)

Active-Standby Architectural Model:
✅ Pattern: Primary region runs full capacity; standby region runs minimal capacity (1-2 nodes)
   → Failover: DNS/LB switch + standby scales up via predefined SOPs/runbooks
   → Cost optimization: Use preemptible/spot in standby; scale only during DR drills/failover
   → RTO: 30-60 mins; RPO: Async replication lag dependent
✅ Exam cue: "Cost-effective DR for stateful apps" → Active-standby + automated scaling + async replication
```

---

## 3️⃣ Network Availability & Load Balancing

### 🌐 Global vs. Regional Load Balancers
```yaml
Global Load Balancer (External HTTP(S), SSL Proxy, TCP Proxy):
✅ Single anycast IP; routes to closest healthy backend globally
✅ Cross-region failover: Automatically shifts traffic if primary region degrades
✅ Use cases: User-facing web apps, APIs, global SaaS platforms
✅ Session affinity: CLIENT_IP, COOKIE, HEADER, GENERATED_COOKIE
✅ Health checks: Global LB requires HTTP(S) or legacy HTTP health checks; backend service level

Regional Load Balancer (Internal HTTP(S), Internal TCP/UDP):
✅ Regional scope; traffic stays within VPC/region
✅ Use cases: Microservice communication, internal APIs, data tier access
✅ High availability: Distributes across zones within region; no automatic cross-region failover
✅ Exam cue: "Internal service communication within single region" → Regional LB; "Global users + cross-region DR" → Global LB
```

### 🔗 Multi-Region Connectivity for HA/DR
```yaml
Multi-Region Cloud VPN (HA VPN):
✅ Two tunnels per VPN gateway; redundant IKEv2 connections
✅ Multi-region design: Deploy HA VPN gateways in different regions → survive single-region failure
✅ Use case: Cost-effective hybrid connectivity with ~99.99% SLA

High Availability Interconnect (Dedicated):
✅ Dual connected architecture: 2+ connections to different Google PoPs
✅ Redundant Cloud Routers in active-active BGP configuration
✅ Use case: Enterprise-grade hybrid with low latency, high throughput, ~99.999% SLA
✅ Exam cue: "Mission-critical hybrid with zero single point of failure" → HA Interconnect + redundant routers + BGP
```

---

## 4️⃣ Container Health & Cloud Run HA Architecture

### 🐳 GKE Probe Strategy
```yaml
Liveness Probe:
• Purpose: Detect if container is crashed/deadlocked
• Action: Restart container if probe fails
• Configuration: exec, httpGet, tcpSocket
• Exam trap: Don't use for slow-starting apps; will cause restart loops

Readiness Probe:
• Purpose: Determine if container is ready to receive traffic
• Action: Remove from Service endpoints if fails; re-add when passes
• Use case: Warm-up caches, establish DB connections before accepting requests
• Exam tip: Separate readiness from liveness for graceful degradation

Startup Probe (GKE 1.18+):
• Purpose: Delay liveness checks for slow-starting containers
• Action: Disables liveness until startup probe succeeds
• Use case: Legacy apps, heavy initialization, large model loading
```

### ☁️ Cloud Run High Availability Architecture
```
Cloud Run is inherently regional (multi-zone), but true HA requires cross-region design:
✅ Single Region: Automatic zone distribution, handles zone failure transparently
✅ Multi-Region HA:
   • Deploy identical Cloud Run services in 2+ regions
   • Use Global External HTTP(S) Load Balancer with backend services per region
   • Configure health checks + failover priorities (primary/backup)
   • Use Cloud DNS or Global LB regional failover routing policies
✅ Exam cue: "Cloud Run HA across regions" → Multi-region deployments + Global LB + regional backend failover
```

---

## 5️⃣ Recovery Planning & RPO/RTO Framework

### 📐 RPO & RTO Definition & Mapping
```yaml
RPO (Recovery Point Objective): Maximum acceptable data loss measured in time
   • Determines replication strategy: Sync (RPO≈0) vs. Async (RPO>0) vs. Backups (RPO=backup interval)

RTO (Recovery Time Objective): Maximum acceptable downtime measured in time
   • Determines failover mechanism: Auto-failover (mins) vs. Manual/Scripted (hours) vs. Restore from backup (days)

Modulator's HA vs. DR Guidance:
• HA (High Availability): Failing over within a region/zones seamlessly. Focus: Load balancers, multi-zone MIGs, auto-failover DBs, synchronous replication. Goal: Minutes or zero downtime.
• DR (Disaster Recovery): Plan for when an entire region or provider fails. Focus: Cross-region async replication, active-standby/active-active, backup & DR service, DNS failover. Goal: Hours to minutes, with acceptable data loss.

Exam Decision Matrix:
| RPO/RTO Requirement | Architecture Pattern | Cost Impact |
|---------------------|----------------------|-------------|
| RPO=0, RTO<5min     | Active-Active + Spanner/Multi-region BigQuery + Global LB | Highest |
| RPO<15min, RTO<1hr  | Active-Standby + Async DB replication + Standby MIGs + Auto-failover scripts | Medium-High |
| RPO=24hr, RTO=4hr   | Backup & Restore + Backup & DR Service + Runbook automation | Lowest |
```

---

## 6️⃣ Data Protection, Failover & State Management

### 🗄️ Cloud SQL Availability & Backup Strategies
```yaml
High Availability Configuration:
✅ Zonal HA: Synchronous replication to standby in different zone
✅ Automatic failover: ~few minutes; virtual IP moves to standby
✅ Read replicas: Async replication to other zones/regions; scale reads, offload backups
✅ Exam cue: "Zero data loss within region" → HA config + sync replication; "Global reads" → Cross-region async replicas

Backup & Replication Lag Management:
✅ Automated backups: Daily snapshots; enable Point-In-Time Recovery (PITR) for continuous transaction logs
✅ Replication lag monitoring: Cloud Monitoring metric `cloudsql.googleapis.com/replication/replica_lag`
✅ Alerting: Trigger if lag > threshold; prevent failover during high lag to avoid data loss
✅ Exam cue: "Safe failover during replica sync" → Monitor lag + delay switchover until lag < RPO threshold

Cloud Storage Data Management:
✅ Versioning: Preserve previous versions of objects; protects against accidental deletion/overwrite
✅ Object Retention & Bucket Lock: WORM compliance; prevent deletion until retention period expires
✅ Lifecycle Policies: Transition to Nearline/Coldline/Archive; delete old versions after grace period
✅ Exam cue: "Protect against accidental deletion + compliance" → Versioning + retention lock + lifecycle policies
```

### 🔄 State & Session Management (App Engine)
```yaml
App Engine is stateless by design:
✅ State externalization: Use Memorystore (Redis) for session caching, Cloud SQL/Datastore for persistent state
✅ Memcache (Legacy concept): Distributed in-memory caching; eventual consistency; not a source of truth
✅ Session affinity: Generated cookie or client IP if state must stick to instance (avoid if possible)
✅ Exam cue: "App Engine stateful requirement" → Externalize to Memorystore/Cloud SQL; avoid local filesystem/session state
```

---

## 7️⃣ Dynamic Scaling Patterns

### 📈 Scaling Strategies Across Services
```yaml
GKE Autoscaling & Overprovisioning:
✅ Cluster Autoscaler: Adds nodes when pods pending due to resources
✅ HPA (Horizontal Pod Autoscaler): Scales pod replicas based on CPU/memory/custom metrics
✅ Overprovisioning: Deploy dummy "pause" pods with low priority; Cluster Autoscaler pre-provisions capacity
   → Benefit: Eliminates cold-start/node boot delay for sudden traffic spikes
✅ Exam cue: "Fast scale-up for unpredictable spikes" → HPA + overprovisioning with low-priority pause pods

Cloud Run Autoscaling & Cold Start Management:
✅ Configuration: min_instances, max_instances, concurrency, CPU allocation
✅ Cold start mitigation:
   • min_instances=1: Keep one instance warm (eliminates cold start; costs ~$7/month)
   • Provisioned concurrency (if available): Pre-warm multiple instances
   • Keep-alive requests: Scheduled pings to maintain warmth (less reliable than min_instances)
✅ Exam cue: "Latency-sensitive API with idle periods" → min_instances=1 + concurrency tuning + CPU always allocated

Compute Engine Autoscaling (MIGs):
✅ Metrics: CPU utilization, HTTP load balancing capacity, Cloud Monitoring custom metrics
✅ Scale-down policy: Time-based (wait before terminating) + predictive scaling for known patterns
✅ Exam cue: "Steady baseline + predictable daily spikes" → Schedule scaling + CPU/custom metric autoscaling
```

---

## 8️⃣ Data Integrity Under Load

### 🌊 Stream Processing & Hotspotting Prevention
```yaml
Dataflow: Out-of-Order Data Handling:
✅ Watermarks: Estimate of "completeness" for event-time data; guides when to emit results
✅ Triggers:
   • Early trigger: Emit partial results before watermark (low latency)
   • On-time trigger: Emit when watermark passes window end
   • Late trigger: Handle data arriving after watermark
✅ Allowed lateness: Time window to accept late data; discard or route to side output after
✅ Exam cue: "Real-time dashboard with eventual consistency" → Early + On-time triggers + allowed lateness

Bigtable Hotspotting Prevention:
✅ Problem: Sequential row keys (timestamps, IDs) route all writes to single node → throughput bottleneck
✅ Solutions:
   • Field promotion: Move highly variable field to front of key
   • Salting: Add random prefix to distribute writes across nodes
   • Time series reversal: Reverse timestamp for chronological reads with distributed writes
✅ Exam cue: "High-throughput IoT telemetry ingestion" → Reversed timestamp or salted row keys

Cloud Storage Hotspotting Prevention:
✅ Modern GCS handles high QPS automatically, but exam still tests legacy anti-patterns
✅ Avoid: Sequential naming (timestamps, auto-increment IDs) for high write workloads
✅ Use: Hash prefixes, reverse timestamps, or sharded directory structures for massive parallel writes
✅ Exam cue: "Millions of small files written concurrently" → Prefix sharding + avoid sequential naming
```

---

## 9️⃣ Maintenance & Patching Strategies

### 🔧 Compute Engine & MIG Maintenance
```yaml
On-Host Maintenance & Live Migration:
✅ Standard VMs: Live migration for host maintenance, hardware upgrades, security patches (~zero downtime)
✅ Exceptions: Local SSDs, GPUs, or specific machine types → Terminate & restart (configure automatic restart)
✅ Maintenance windows: Schedule node pool updates in GKE/CE during low-traffic periods

MIG Update Policies:
✅ Rolling update: Replace instances with new template gradually
✅ Key parameters:
   • maxSurge: Extra instances created during update (speed)
   • maxUnavailable: Instances allowed to be down (availability)
   • Minimal action: REPLACE (new instance) vs. REFRESH (metadata/script update)
✅ Health check validation: Only mark new instances healthy after passing checks
✅ Exam cue: "Zero-downtime OS/app update" → maxSurge=1, maxUnavailable=0, health check validation, SUBSTITUTE method
```

---

## 🔟 Observability for Reliability

### 📊 Monitoring & Logging for SRE/WAF
```yaml
Cloud Monitoring Uptime Checks:
✅ External probing from multiple global locations (HTTP, HTTPS, TCP, PING)
✅ Use cases: SLA validation, regional outage detection, user-experience simulation
✅ Alerting: Integrate with notification channels; trigger DR runbooks on sustained failures
✅ Exam cue: "Proactively detect regional outage before users report" → Uptime checks + multi-region probing + alerting

Cloud Logging Exports & Sinks:
✅ Sink patterns:
   • Route to BigQuery: Long-term analysis, post-mortem, audit compliance, SLA reporting
   • Route to Cloud Storage: Immutable archive, regulatory retention
   • Exclude verbose/debug logs: Reduce cost & noise in production sinks
✅ Post-mortem & Reliability:
   • Structured logging with trace IDs for request correlation
   • Export admin activity + data access logs for security/audit trails
   • Log-based metrics for custom reliability SLOs (e.g., error rate, latency percentiles)
✅ Exam cue: "Post-incident analysis + compliance audit" → Logging sink to BigQuery/Storage + structured logs + retention policies
```

---

## 1️⃣1️⃣ Modulator's Deep-Dive Decision Frameworks

### 🧭 Framework 1: HA vs. DR Selection
```
Ask: "Is the failure scope zonal/regional or cross-provider/global?"
├─► Zonal/Regional failure expected → HIGH AVAILABILITY
│   ├─► Compute: Regional MIGs or GKE multi-node pools
│   ├─► Network: Regional/Global LB with zonal backends
│   ├─► Data: Cloud SQL HA (sync standby in diff zone), Memorystore HA
│   └─► Goal: Automatic failover, minutes/zero downtime, low RPO
│
└─► Region-wide or provider failure expected → DISASTER RECOVERY
    ├─► Compute: Active-standby MIGs in secondary region, warm/cold standby
    ├─► Network: DNS failover or Global LB regional priority routing
    ├─► Data: Async cross-region replicas, Backup & DR Service, periodic replication
    └─► Goal: Scripted/automated switchover, hours to minutes RTO, acceptable RPO

💡 Exam Rule: "Seamless failover within region" = HA. "Survive region loss" = DR. Don't over-engineer HA with DR costs, or under-engineer DR with HA tools.
```

### 🧭 Framework 2: Liveness vs. Readiness Probe Selection
```
Scenario: "Container is running but database connection pool isn't initialized yet."
✅ Use Readiness Probe: Removes pod from Service until DB connection ready. Traffic routed elsewhere. No restart.
❌ Don't use Liveness: Would restart the pod repeatedly, causing cascading failures.

Scenario: "Container is deadlocked and not responding to HTTP checks."
✅ Use Liveness Probe: Kubernetes kills and restarts the container to recover.
❌ Don't use Readiness: Pod would stay "Ready" in endpoints but actually broken, serving errors to users.

💡 Exam Rule: Liveness = "Am I crashed?" → Restart. Readiness = "Am I ready for traffic?" → Route/Unroute.
```

### 🧭 Framework 3: Data Consistency for Global Resiliency
```
Requirement: "Global low-latency access with strong consistency and minimal RPO."
✅ Choose: Cloud Spanner
   • Synchronous cross-region replication
   • Strong consistency, horizontal scale, automatic sharding
   • RPO ≈ 0, high availability across regions

Requirement: "Regional ACID with acceptable async replication lag for global reads."
✅ Choose: Cloud SQL + Cross-Region Read Replicas
   • Async replication; RPO depends on network lag
   • Cost-effective for regional primary + global read scaling
   • Not suitable for strict strong consistency globally

💡 Exam Rule: Global strong consistency + low RPO → Spanner. Regional ACID + async global reads → Cloud SQL replicas. Never use async replication for "zero data loss global consistency" requirements.
```

---

## 1️⃣2️⃣ Exam Traps & Tricks

### 🚫 Top 10 Domain 6 Exam Traps
| Trap | What It Looks Like | Why It's Wrong | How to Avoid |
|------|-------------------|----------------|--------------|
| **The HA/DR Confusion** | Suggests active-standby DR for zonal fault tolerance | Overkill; violates cost/reliability trade-off | Match scope: Zone/region failure → HA; Region/provider failure → DR |
| **The Probe Swap** | Uses liveness probe for slow startup; readiness for deadlock | Causes restart loops or serves broken traffic | Liveness = crash recovery; Readiness = traffic routing; Startup = delay liveness |
| **The Async=RPO0 Fallacy** | Claims async cross-region replication meets "zero data loss" | Async always has replication lag; RPO>0 | For RPO≈0 globally, choose Spanner or multi-region sync services |
| **The Cold Start Ignorance** | Sets min_instances=0 for latency-critical APIs | First request experiences 1-3s cold start penalty | Use min_instances≥1 or provisioned concurrency for latency-sensitive paths |
| **The Hotspot Anti-Pattern** | Sequential timestamps/IDs for Bigtable/GCS high-write workloads | Routes all traffic to single node/shard → throughput collapse | Use salting, field promotion, or reversed timestamps for write distribution |
| **The Global/Regional LB Mismatch** | Uses regional LB for global user traffic with cross-region failover | No automatic cross-region routing; fails global requirements | Global users + cross-region DR → Global LB; Internal single-region → Regional LB |
| **The Manual Patching at Scale** | Suggests SSH + apt-get for 100+ VMs | Doesn't scale; violates operational excellence pillar | Use OS Config API, automated patch deployments, or MIG rolling updates |
| **The Versioning Misapplication** | Enables Cloud Storage versioning without lifecycle policies | Unbounded cost growth from accumulating old versions | Pair versioning with lifecycle rules to transition/delete old versions after retention period |
| **The Session Affinity Dependency** | Relies on cookie affinity to maintain application state | Violates stateless cloud-native design; breaks on scaling/failover | Externalize state to Memorystore/Cloud SQL; design stateless whenever possible |
| **The Uptime Check Misconfiguration** | Single-region uptime check for global SLA validation | Misses regional outages affecting specific user segments | Configure multi-region uptime checks with geographic distribution |

---

## 1️⃣3️⃣ Mnemonics & Memory Hacks

### 🧠 Acronyms to Memorize
```
HA vs DR Trade-off → "S.C.A.L.E."
S - Scope: Zone/region (HA) vs. Region/provider (DR)
C - Consistency: Sync (HA) vs. Async (DR)
A - Automation: Auto-failover (HA) vs. Runbook/scripted (DR)
L - Latency/Cost: Lower (HA) vs. Higher complexity/cost (DR)
E - Expectation: Minutes/zero downtime vs. Hours to minutes RTO

Probe Purpose → "P.R.O.B.E."
P - Purpose: Detect failure or readiness?
R - Readiness: Remove from traffic if not ready
O - Operational: Startup delays liveness for slow init
B - Broken: Liveness restarts crashed containers
E - Endpoint: Readiness controls Service endpoints; Liveness controls container lifecycle

Hotspotting Prevention → "S.A.L.T."
S - Salt: Add random prefix to distribute writes
A - Avoid: Sequential timestamps/auto-increment keys
L - Layout: Reverse timestamps for time-series reads
T - Throughput: Test write patterns before production deployment

Reliability Observability → "M.A.P.S."
M - Monitoring: Uptime checks, SLO burn rate, custom metrics
A - Alerts: Thresholds tied to error budgets, not raw values
P - Post-mortem: Logging exports to BigQuery/Storage for analysis
S - Sinks: Structured logs, trace correlation, exclusion filters for cost
```

---

## 1️⃣4️⃣ Practice Checkpoint (10 Scenarios)

### 🔹 Scenario Set A: Compute, Network & Container Resilience (4)

**Scenario A1**:
```
A production API requires 99.95% availability within a single region. Workload is stateless. Requirements:
• Survive zone failure automatically
• Zero downtime during instance updates
• Minimal configuration overhead

Question: Which compute architecture BEST meets requirements?
A) Zonal MIG with manual scaling and blue-green deployments
B) Regional MIG with autohealing, maxSurge=1/maxUnavailable=0 rolling update policy, and health checks
C) Active-standby MIGs across two regions with DNS failover
D) Single Compute Engine instance with automatic restart enabled

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: B**
**Why**: Regional MIG automatically distributes instances across 3 zones, surviving zone failure. Autohealing replaces unhealthy instances. Rolling update with maxSurge=1/maxUnavailable=0 ensures zero downtime during template updates. Minimal overhead vs multi-region or manual patterns.
**Why not others**: A (zonal fails zone failure; manual scaling adds overhead), C (multi-region is DR, not HA; overkill/costly for single-region requirement), D (single instance = no HA; violates availability target).
**WAF Pillar**: Reliability (zone redundancy, autohealing, zero-downtime updates)
</details>
```

**Scenario A2** *(Modulator Focus: HA vs DR)*:
```
Company requires RPO=0, RTO<5min for global transaction processing. Data must be strongly consistent across US and Europe.
Question: Which data architecture is MOST appropriate?
A) Cloud SQL primary in us-central1 with async cross-region replica in europe-west1
B) Cloud Spanner multi-region instance with synchronous replication across continents
C) Cloud Storage dual-region with object versioning and lifecycle policies
D) Memorystore Redis with active-active configuration across regions

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: B**
**Why**: Cloud Spanner provides synchronous cross-region replication with strong consistency, meeting RPO≈0 and RTO<5min requirements globally. Designed specifically for global ACID workloads.
**Why not others**: A (async replication has lag → RPO>0; violates requirement), C (object storage, not transactional), D (Memorystore doesn't support global active-active with strong consistency; Redis replication is async).
**Exam Rule**: Global strong consistency + zero data loss = Spanner. Never async for RPO=0 globally.
</details>
```

**Scenario A3** *(Probe Selection)*:
```
GKE pod runs a legacy app that takes 3 minutes to initialize database connections and warm caches. During init, HTTP endpoint returns 503.
Question: Which probe configuration prevents premature restarts and traffic routing?
A) Liveness probe checking /health every 10s
B) Readiness probe checking /ready every 5s; Liveness probe disabled
C) Startup probe checking /init for 3 minutes, then Liveness probe on /health every 10s
D) Readiness and Liveness probes both checking /ready every 5s

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: C**
**Why**: Startup probe delays liveness checks until app is fully initialized, preventing restart loops during slow startup. Once startup succeeds, liveness monitors for crashes. Readiness can independently control traffic routing.
**Why not others**: A (liveness will restart pod before 3min init completes), B (liveness needed for crash recovery; readiness alone doesn't restart dead containers), D (same readiness for both probes mixes concerns; liveness should check for crash/deadlock).
</details>
```

**Scenario A4** *(LB Selection)*:
```
Internal microservices in us-east1 communicate via REST APIs. Requirements:
• Load balance across 3 pods in different zones
• Keep traffic within VPC; no public internet exposure
• Support HTTP path-based routing

Question: Which load balancer is MOST appropriate?
A) External Global HTTP(S) Load Balancer
B) Internal Regional HTTP(S) Load Balancer
C) TCP/SSL Proxy Load Balancer
D) Network Load Balancer (L4)

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: B**
**Why**: Internal Regional HTTP(S) LB operates within a single region/VPC, supports path-based L7 routing, and distributes traffic across zones without public exposure. Matches internal microservice requirements.
**Why not others**: A (global + external internet exposure), C (L4 proxy, no HTTP path routing), D (L4 network LB, no L7 path-based routing).
</details>
```

---

### 🔹 Scenario Set B: Data, Scaling & Observability (6)

**Scenario B1** *(Cloud Storage Hotspotting)*:
```
Application writes 10,000 small JSON files per second to Cloud Storage. Objects are named `logs/YYYY-MM-DD/HH-MM-SS-deviceID.json`. Users report throttling and high latency.
Question: Which change BEST resolves the performance issue?
A) Enable Cloud Storage transfer acceleration and increase chunk size
B) Add a random hash prefix or reverse the timestamp in object names
C) Migrate objects to Bigtable for better write throughput
D) Increase the number of Cloud Storage buckets to 100

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: B**
**Why**: Sequential timestamps cause hotspotting by directing writes to the same storage partition. Hash prefixes or reversed timestamps distribute writes evenly across partitions, resolving throttling.
**Why not others**: A (transfer acceleration is for cross-region download speed, not write hotspotting), C (Bigtable is for structured KV/time-series, not object files), D (multiple buckets adds complexity; single bucket with proper key distribution is sufficient).
</details>
```

**Scenario B2** *(Cloud Run Cold Start)*:
```
Latency-sensitive payment processing API deployed on Cloud Run experiences 1.5s delay on first request after 5 minutes of inactivity. SLO: p95 < 300ms.
Question: Which configuration change BEST meets the SLO?
A) Set max_instances to 100 and concurrency to 1
B) Set min_instances=1 and keep CPU allocated during request processing
C) Disable automatic scaling and run 24/7 fixed capacity
D) Migrate to App Engine Flexible environment

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: B**
**Why**: min_instances=1 keeps an instance warm, eliminating cold start penalty. Keeping CPU allocated ensures background tasks complete and reduces initialization time. Cost-effective for low-traffic latency-sensitive APIs.
**Why not others**: A (max_instances/concurrency don't prevent cold starts), C (fixed 24/7 wastes cost during idle periods), D (App Engine Flexible has slower cold starts than Cloud Run gen2; adds overhead).
</details>
```

**Scenario B3** *(Dataflow Out-of-Order)*:
```
Streaming pipeline ingests IoT sensor data. Network delays cause events to arrive up to 5 minutes out of order. Dashboard requires near-real-time updates with eventual consistency.
Question: Which Dataflow windowing/trigger pattern is MOST appropriate?
A) Fixed window with on-time trigger only; discard late data
B) Sliding window with early trigger every 30s, on-time trigger at watermark, allowed lateness 5m
C) Global window with single final trigger after 10 minutes
D) Session window with 1-minute gap duration

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: B**
**Why**: Early trigger provides near-real-time dashboard updates. On-time trigger emits final results when watermark passes. Allowed lateness (5m) captures delayed events for eventual consistency. Matches requirements precisely.
**Why not others**: A (discards late data → incomplete results), C (10-min delay violates near-real-time requirement), D (session windows for user activity, not time-series IoT telemetry).
</details>
```

**Scenario B4** *(Backup & DR Service)*:
```
Enterprise runs 200 VMs across production, staging, and dev. Requirements:
• Centralized backup management with policy-based retention
• Immutable backup storage for compliance
• Automated restore testing to validate RTO
• Cost-optimized lifecycle for older backups

Question: Which solution BEST meets requirements?
A) Manual gsutil snapshots to Cloud Storage with lifecycle policies
B) Compute Engine scheduled snapshots with Cloud Scheduler triggers
C) Backup and DR service with policy-based retention, immutable vaults, and automated restore testing
D) Third-party backup agent installed on each VM managing local storage

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: C**
**Why**: Backup & DR service provides centralized, policy-driven backup management across projects. Supports immutable storage (WORM), automated restore validation for RTO testing, and lifecycle optimization. Scales to 200+ VMs natively.
**Why not others**: A/B (manual/scheduled snapshots lack centralized governance, restore testing, and compliance vaults), D (third-party agents add management overhead; violates "centralized" requirement).
</details>
```

**Scenario B5** *(Cloud SQL Failover)*:
```
Cloud SQL instance shows replica lag of 45 seconds. Primary region experiences zone failure. Automated failover is configured. RPO requirement: ≤30 seconds.
Question: What should the architecture do BEFORE failing over?
A) Immediately promote replica regardless of lag
B) Wait until replication lag drops below 30 seconds, then promote
C) Restore from last backup instead of promoting replica
D) Switch read traffic to replica but keep primary as write master

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: B**
**Why**: Promoting a replica with 45s lag violates RPO≤30s requirement. Architecture must monitor replication lag metric and delay failover until lag < RPO threshold to ensure acceptable data loss.
**Why not others**: A (violates RPO requirement), C (backup restore takes hours → violates RTO), D (keeps broken primary as write master → data corruption/outage).
</details>
```

**Scenario B6** *(Observability & Uptime Checks)*:
```
Global SaaS platform needs to detect regional outages within 2 minutes and alert SRE team before user reports. SLO: 99.99% availability.
Question: Which observability configuration BEST meets requirements?
A) Cloud Monitoring dashboard showing internal CPU utilization
B) Multi-region Cloud Monitoring uptime checks with alerting policy tied to error budget burn rate
C) Cloud Logging export of 5xx errors to BigQuery with daily report generation
D) Single-region HTTP health check on load balancer backend

<details>
<summary>✅ Answer & Explanation</summary>
**Correct: B**
**Why**: Multi-region uptime checks probe from external locations, detecting regional outages independent of internal metrics. Burn rate alerting ties failures to error budget, aligning with SRE principles and enabling proactive response before user reports.
**Why not others**: A (internal metrics don't reflect user experience or regional outages), C (daily report is too slow for 2-min detection), D (single-region check misses regional/user-facing failures).
</details>
```

---

## 1️⃣5️⃣ 2025–2026 Changes & Curated YouTube Playlist

### 🔄 What's New in Domain 6 for PCA v6.1
```
✅ Added/Emphasized:
• Backup and DR managed service patterns for centralized, policy-driven recovery
• Cloud Run multi-region HA with Global LB failover routing
• GKE startup probes + overprovisioning for fast scale-up
• SRE-aligned reliability: error budget burn rate alerting, multi-window SLO monitoring
• Dataflow watermark/trigger patterns for out-of-order stream integrity
• Explicit HA vs. DR scope definition in exam scenarios

❌ De-emphasized:
• Manual snapshot/backup scripting (focus on managed Backup & DR, PITR, automation)
• Legacy Memcache as primary state store (emphasize Memorystore/Cloud SQL externalization)
• Raw SLO calculation math (focus on architectural alignment with SLOs/burn rates)
```

### 📚 Updated Resource Recommendations
| Video Title | Channel | Duration | Why It's Critical |
|-------------|---------|----------|-------------------|
| HA vs DR: When to Use Which Pattern | Google Cloud Tech | 19 min | Clear scope boundaries, RPO/RTO mapping, cost trade-offs |
| GKE Health Probes: Liveness vs Readiness vs Startup | Google Cloud Tech | 16 min | Probe selection decision tree, anti-pattern avoidance |
| Cloud Spanner for Global Resiliency | Google Cloud Tech | 21 min | Synchronous replication, strong consistency, low RPO design |
| Dataflow Windowing & Triggers Deep Dive | Google Cloud Tech | 22 min | Watermarks, early/on-time/late triggers, allowed lateness |
| Cloud Monitoring Uptime Checks & SRE Alerting | Google Cloud Tech | 18 min | Multi-region probing, burn rate alerts, error budget alignment |

### 🔁 Recommended Viewing Order
```
Week 1: Core Reliability Concepts
✓ Day 1: "HA vs DR Patterns" + "GKE Health Probes"
✓ Day 2: "Cloud Spanner for Global Resiliency" + "Cloud Monitoring Uptime Checks"
✓ Day 3: "Dataflow Windowing & Triggers" + take notes using decision frameworks

Week 2: Advanced Patterns & Practice
✓ Day 1: Review modulator's HA/DR, Probe, Consistency frameworks
✓ Day 2: Sketch Cloud Run multi-region HA, Cloud SQL failover with lag monitoring
✓ Day 3: "Domain 6 Practice: Live Solve" → attempt scenarios, watch explanations

Week 3: Final Review
✓ Day 1: Update trap avoidance list; focus on Async≠RPO0, Probe mix-ups, LB scope
✓ Day 2: Teach one HA/DR scenario to study pod using stakeholder summary format
✓ Day 3: Final checklist review; confidence check on WAF Reliability pillar alignment
```

---

> ✅ **Domain 6 Completion Checklist**
> - [ ] Watch all 5 curated videos; take notes using HA/DR, Probe, Consistency frameworks
> - [ ] Complete all 10 practice scenarios without peeking at answers
> - [ ] Create personal "Reliability Decision Cheat Sheet" for exam day (HA vs DR, Probe selection, LB scope)
> - [ ] Sketch multi-region architecture with Global LB + Backup & DR + RPO/RTO mapping
> - [ ] Teach one SRE/Uptime scenario to cohort using error budget burn rate explanation
> - [ ] Final review of 2026 updates: Backup & DR, startup probes, async consistency limits

---
