# Google Cloud Professional Cloud Architect (PCA) Exam Study Guide
## Backup, Disaster Recovery & Migration Strategy Domain

> **Target Certification**: Google Cloud Professional Cloud Architect (PCA)  
> **Focus Area**: Backup/DR Design, RTO/RPO, Migration Center, Shared VPC, Phased Migration  
> **Last Updated**: April 2026  
> **Format**: Markdown (.md)

---

## 📋 Table of Contents

1. [Exam Overview & Domain Weighting](#exam-overview--domain-weighting)
2. [PCA Exam Tips for Backup/DR & Migration Questions](#pca-exam-tips-for-backupdr--migration-questions)
3. [Backup & Disaster Recovery Fundamentals](#backup--disaster-recovery-fundamentals)
4. [RTO/RPO: Definitions, Trade-offs & Design](#rtorpo-definitions-trade-offs--design)
5. [DR Strategy Patterns on Google Cloud](#dr-strategy-patterns-on-google-cloud)
6. [Migration Center & Assessment Tools](#migration-center--assessment-tools)
7. [Shared VPC: Architecture & Governance](#shared-vpc-architecture--governance)
8. [Phased Migration Strategies](#phased-migration-strategies)
9. [Scenario-Based Practice Quizzes](#scenario-based-practice-quizzes)
10. [Quick Reference Tables](#quick-reference-tables)

---

## Exam Overview & Domain Weighting

### PCA Exam Domain Structure (2026)

| Domain | Approximate Weight | Key Topics for This Guide |
|--------|-------------------|---------------------------|
| **Designing & Planning Cloud Solution Architecture** | **~24-30%** | **DR architecture, RTO/RPO alignment, migration strategy selection, Shared VPC design** [[46]] |
| Managing & Provisioning a Solution Infrastructure | ~18-25% | Backup configuration, Shared VPC implementation, migration tool deployment |
| Designing for Security & Compliance | ~20% | Cross-project IAM in Shared VPC, encrypted backups, compliance during migration |
| Analyzing & Optimizing Technical/Business Processes | ~15% | Cost optimization of DR strategies, migration TCO analysis, backup retention policies |
| Managing Implementation | ~10% | Migration wave execution, cutover planning, rollback procedures |

> ⚠️ **Critical Insight**: Backup/DR and migration questions frequently appear in the **Designing & Planning** domain (~24-30% weight), making this one of the highest-impact topic areas. Expect 8-15 questions (13-25% of exam) to directly test these concepts [[63]].

### Expected Question Distribution for This Domain

```
📊 Approximate Question Count (of 50-60 total):
• RTO/RPO calculation & trade-off analysis: 2-4 questions
• DR strategy selection (pilot light vs. multi-region): 2-3 questions
• Backup configuration & retention policies: 2-3 questions
• Migration Center assessment & tool selection: 2-3 questions
• Shared VPC design & IAM patterns: 2-4 questions
• Phased migration planning & cutover strategy: 2-3 questions

Total: ~12-20 questions (20-33% of exam) may test these topics
```

### Exam Format Reminder
- **Duration**: 2 hours (120 minutes)
- **Questions**: 50-60 multiple-choice/multiple-select (some with case studies)
- **Case Studies**: 20-30% of questions reference fictional company scenarios [[5]]
- **Passing Score**: Not publicly disclosed (scaled scoring)

---

## PCA Exam Tips for Backup/DR & Migration Questions 🎯

### 🧠 Strategic Approach

1. **Always Start with Business Requirements**
   ```
   PCA questions reward solutions aligned with business outcomes:
   
   Step 1: Identify RTO/RPO from scenario (explicit or implied)
   Step 2: Map to appropriate DR pattern (cost vs. recovery trade-off)
   Step 3: Verify compliance/data residency constraints
   Step 4: Select migration approach based on application complexity
   
   Example: "Financial trading platform" → Likely low RTO/RPO → Multi-region active-active
   Example: "Internal reporting tool" → Higher RTO/RPO acceptable → Pilot light or backup/restore
   ```

2. **Master the RTO/RPO → DR Strategy Decision Tree**
   ```
   RTO < 1 hour AND RPO < 15 min → Multi-region active-active or hot standby
   RTO 1-4 hours AND RPO 15 min-1 hour → Warm standby with frequent replication
   RTO 4-24 hours AND RPO 1-24 hours → Pilot light with on-demand scaling
   RTO > 24 hours OR RPO > 24 hours → Backup/restore from cold storage
   
   Exam Tip: Questions often provide business impact statements instead of explicit RTO/RPO:
   • "Cannot lose more than 5 minutes of transactions" → RPO ≤ 5 min
   • "Must be operational within 30 minutes of outage" → RTO ≤ 30 min
   ```

3. **Know the Migration Assessment Workflow** [[35]]
   ```
   Migration Center Process:
   Discovery → Assessment → Planning → Execution → Validation
   
   Key Tools by Workload Type:
   • VMs: Migrate for Compute Engine (formerly Velostrata)
   • Databases: Database Migration Service (DMS)
   • Kubernetes: Migrate for Anthos / GKE migration tools
   • Files: Storage Transfer Service, rsync, third-party tools
   
   Exam Tip: Questions may ask "What is the FIRST step?" → Always Discovery/Assessment
   ```

4. **Shared VPC: Remember the Host/Service Project Model**
   ```
   Host Project:
   • Contains the VPC network resources (subnets, routes, firewalls)
   • Managed by network administrators
   • Grants usage permissions to service projects via IAM
   
   Service Projects:
   • Contain application resources (VMs, GKE, Cloud Run)
   • Attach to host project's subnets
   • Cannot modify network configuration directly
   
   Critical IAM Roles:
   • roles/compute.networkAdmin (host project): Full network control
   • roles/compute.networkUser (service project): Can attach resources to subnets
   ```

5. **Phased Migration: Know the "6 R's" Framework** [[40]]
   ```
   Rehost (lift-and-shift): Minimal changes, fastest migration
   Replatform (lift-tinker-shift): Minor cloud optimizations (managed DB, etc.)
   Refactor (re-architect): Significant changes for cloud-native benefits
   Repurchase: Replace with SaaS solution
   Retire: Decommission unused applications
   Retain: Keep on-premises (hybrid approach)
   
   Exam Tip: Questions often ask "Which approach minimizes risk?" → Rehost with thorough testing
   ```

### 🔍 Question Analysis Framework

```
When you see a Backup/DR/Migration question:
1. IDENTIFY the workload type (stateless app? stateful DB? batch processing?)
2. LOCATE constraints (RTO/RPO? compliance? budget? timeline?)
3. MATCH to appropriate GCP service(s) with cost/performance trade-offs
4. VERIFY data residency and cross-region replication requirements
5. CHECK migration dependencies (network connectivity, IAM, testing strategy)
```

### 🚫 Common Pitfalls to Avoid

| Mistake | Correct Approach |
|---------|-----------------|
| Choosing multi-region DR for low-criticality workloads | Match DR strategy to business impact; avoid over-engineering |
| Confusing RTO with RPO in scenario analysis | RTO = time to recover; RPO = data loss tolerance |
| Using Shared VPC without proper IAM delegation | Grant roles/compute.networkUser to service project teams |
| Planning "big bang" migration for complex environments | Use phased approach with wave planning and rollback procedures |
| Ignoring egress costs in multi-region DR design | Factor in replication and failover traffic costs |
| Assuming Database Migration Service supports all databases | Verify source/target compatibility (Oracle, SQL Server, MySQL, PostgreSQL supported) |

---

## Backup & Disaster Recovery Fundamentals

### Core Concepts (Memorize for Exam!)

```yaml
Backup vs. Disaster Recovery:
  Backup:
    • Purpose: Point-in-time copy of data for restoration
    • Scope: Data-level protection (files, databases, disks)
    • Recovery: Restore individual items or full systems
    • Tools: Persistent Disk snapshots, Cloud SQL backups, Cloud Storage versioning
    
  Disaster Recovery:
    • Purpose: Maintain business continuity during major outages
    • Scope: Application/infrastructure-level protection
    • Recovery: Failover to alternate region/environment
    • Tools: Multi-region deployment, global load balancing, DNS failover

Key Metrics:
  RTO (Recovery Time Objective): Maximum acceptable downtime
  RPO (Recovery Point Objective): Maximum acceptable data loss
  MTTR (Mean Time To Recovery): Average time to restore service
  MTBF (Mean Time Between Failures): Average time between outages
```

### Google Cloud Backup Services Comparison

| Service | Backup Type | Retention Options | Cross-Region | Best For |
|---------|------------|------------------|--------------|----------|
| **Persistent Disk Snapshots** | Block-level disk backup | Custom schedules, up to daily | ✅ Manual copy to another region | GCE VMs, stateful workloads |
| **Cloud SQL Backups** | Automated DB backups + on-demand | 1-365 days, point-in-time recovery | ✅ Cross-region replicas (read-only) | Managed relational databases |
| **Cloud Storage Versioning** | Object-level versioning | Custom lifecycle policies | ✅ Multi-region buckets | Unstructured data, archives |
| **Filestore Backups** | NFS file share snapshots | Custom schedules | ✅ Async replication to another region | Shared file systems, legacy apps |
| **BigQuery Table Snapshots** | Point-in-time table copy | 2-7 days (configurable) | ✅ Cross-region replication | Analytics workloads |

### Critical Exam Concepts

✅ **Snapshot Best Practices**:
```
• Application-consistent snapshots: Quiesce applications before snapshot
• Frequency alignment: Match snapshot interval to RPO requirements
• Retention policy: Balance recovery needs with storage costs
• Cross-region copy: Use gcloud or automation for DR readiness
• Encryption: CMEK for compliance-sensitive backups
```

✅ **Backup Testing Strategy (Often Tested!)**:
```
PCA Exam Expectation: "Backups are useless if not tested"

Testing Approaches:
• Regular restore drills: Validate backup integrity quarterly
• Isolated test environment: Restore to separate project/region
• Automated validation: Script checks for data consistency
• Documentation: Maintain runbooks for restore procedures

Exam Tip: Questions may ask "How to ensure backups are recoverable?" → Regular testing in isolated environment
```

### Sample Exam Question

> **Scenario**: A media company stores user-generated content in Cloud Storage. Requirements:
> - Protect against accidental deletion or corruption
> - Recover individual files within 1 hour (RTO)
> - Maximum data loss of 15 minutes (RPO)
> - Minimize storage costs for historical versions
>
> **Question**: Which backup strategy BEST meets these requirements?
>
> A) Enable Cloud Storage versioning + lifecycle policy to delete versions older than 30 days  
> B) Daily Persistent Disk snapshots of VMs hosting the application  
> C) Cross-region replication to a backup bucket with 7-day retention  
> D) Export data daily to Coldline storage with 1-year retention  
>
> **Answer**: A  
> **Rationale**: Cloud Storage versioning enables point-in-time recovery of individual objects (meets 1-hour RTO). Combined with frequent uploads (every ≤15 min), it satisfies 15-minute RPO. Lifecycle policies control costs by removing old versions. Option B protects VMs, not object storage; C provides regional failover but not granular file recovery; D is for archival, not operational recovery [[23]].

---

## RTO/RPO: Definitions, Trade-offs & Design

### RTO/RPO Deep Dive

```yaml
RTO (Recovery Time Objective):
  Definition: Maximum acceptable duration of service interruption
  Business Impact: Revenue loss, customer dissatisfaction, compliance penalties
  Design Considerations:
    • Automation level: Manual failover vs. automated orchestration
    • Infrastructure readiness: Pre-provisioned vs. on-demand resource creation
    • Data synchronization: Async replication (faster RTO) vs. sync (slower but consistent)
    • Testing frequency: Regular drills reduce actual recovery time

RPO (Recovery Point Objective):
  Definition: Maximum acceptable amount of data loss measured in time
  Business Impact: Lost transactions, data reconciliation effort, regulatory fines
  Design Considerations:
    • Replication frequency: Continuous sync vs. periodic snapshots
    • Write acknowledgment: Async (lower RPO risk) vs. sync (higher latency)
    • Conflict resolution: Last-write-wins vs. application-level merge logic
    • Cost trade-off: More frequent replication = higher egress/storage costs
```

### RTO/RPO → Technical Implementation Mapping

| RTO Target | RPO Target | Recommended Architecture | Approximate Cost Multiplier |
|------------|------------|-------------------------|----------------------------|
| < 15 minutes | < 1 minute | Multi-region active-active + synchronous replication | 3-5x baseline |
| 15-60 minutes | 1-15 minutes | Multi-region active-passive + async replication + automated failover | 2-3x baseline |
| 1-4 hours | 15 min-1 hour | Warm standby in secondary region + frequent snapshots | 1.5-2x baseline |
| 4-24 hours | 1-24 hours | Pilot light + on-demand scaling + daily backups | 1.2-1.5x baseline |
| > 24 hours | > 24 hours | Backup/restore from cold storage + manual provisioning | 1.1-1.3x baseline |

> 💡 **Exam Tip**: Cost multipliers are approximate but useful for trade-off questions. The exam often asks "Which solution provides the BEST balance of cost and recovery objectives?"

### Calculating RTO/RPO from Business Statements

```
Scenario Language → Technical Translation:

"We cannot afford to lose more than 10 minutes of customer orders"
→ RPO ≤ 10 minutes → Requires frequent replication or continuous sync

"The business can tolerate up to 2 hours of downtime during off-peak hours"
→ RTO ≤ 2 hours (with time-of-day consideration) → Warm standby may suffice

"Regulatory requirements mandate recovery within 30 minutes with zero data loss"
→ RTO ≤ 30 min, RPO = 0 → Multi-region active-active with synchronous replication

"Budget constraints limit DR spending to 20% of production infrastructure cost"
→ Cost multiplier ≤ 1.2x → Pilot light or backup/restore strategy
```

### Sample Exam Question

> **Scenario**: An e-commerce platform processes $10K/minute in transactions during peak hours. Business requirements:
> - Cannot lose more than 1 minute of transaction data
> - Must restore service within 10 minutes of regional outage
> - Budget allows DR infrastructure up to 3x production cost
>
> **Question**: Which architecture BEST meets these requirements?
>
> A) Single-region deployment with hourly Persistent Disk snapshots  
> B) Multi-region active-passive with async replication every 5 minutes + automated failover  
> C) Multi-region active-active with synchronous replication + global load balancing  
> D) Pilot light deployment with daily backups + manual failover procedures  
>
> **Answer**: C  
> **Rationale**: RPO ≤ 1 minute requires synchronous or near-continuous replication. RTO ≤ 10 minutes requires pre-provisioned, ready-to-serve infrastructure with automated failover. Multi-region active-active with global load balancing meets both. Budget allows 3x cost, which aligns with active-active multiplier. Option B has 5-minute replication interval (violates RPO); A and D cannot meet RTO/RPO targets [[38]].

---

## DR Strategy Patterns on Google Cloud

### Four Primary DR Patterns (Exam Essentials)

#### Pattern 1: Backup & Restore (Cold DR)
```
Architecture:
Production Region → Periodic Backups → Cold Storage (Cloud Storage Nearline/Coldline)
                              ↓
                    Manual restore to new region during disaster

Characteristics:
• RTO: Hours to days (manual provisioning + restore time)
• RPO: Hours (based on backup frequency)
• Cost: Lowest (pay only for backup storage)
• Complexity: Low (simple backup scripts)

Best For:
• Non-critical internal tools
• Development/test environments
• Workloads with high RTO/RPO tolerance

GCP Implementation:
• Persistent Disk snapshots → Cloud Storage cross-region copy
• Cloud SQL exports → Cloud Storage with lifecycle policies
• Automation: Cloud Functions/Workflows to orchestrate restore
```

#### Pattern 2: Pilot Light (Warm DR)
```
Architecture:
Production Region → Async Replication → Minimal DR Environment (secondary region)
                              ↓
                    Scale up DR environment during disaster

Characteristics:
• RTO: 30 minutes to 4 hours (scale-up time + data sync)
• RPO: 15 minutes to 1 hour (replication lag)
• Cost: Moderate (pay for minimal DR resources + replication)
• Complexity: Medium (automation for scale-up required)

Best For:
• Business-critical applications with moderate recovery requirements
• Stateful applications with manageable replication lag

GCP Implementation:
• Cloud SQL cross-region read replicas (promote to master during failover)
• Persistent Disk async replication + Instance Templates for quick VM provisioning
• Managed Instance Groups with min-instances=1 in DR region
• DNS failover via Cloud DNS with low TTL
```

#### Pattern 3: Warm Standby (Hot DR)
```
Architecture:
Production Region → Async/Near-sync Replication → Fully Provisioned DR Environment
                              ↓
                    Redirect traffic to DR region during disaster

Characteristics:
• RTO: 5-30 minutes (DNS propagation + connection drain)
• RPO: 1-15 minutes (replication frequency)
• Cost: High (pay for full DR infrastructure, often idle)
• Complexity: High (traffic management, data consistency testing)

Best For:
• Customer-facing applications with strict recovery requirements
• Applications where brief downtime causes significant revenue loss

GCP Implementation:
• Global HTTP(S) Load Balancer with backend services in multiple regions
• Cloud SQL cross-region replicas with automated promotion scripts
• Memorystore for Redis with cross-region replication (beta)
• Traffic Director for service mesh-based failover
```

#### Pattern 4: Multi-Region Active-Active
```
Architecture:
Multiple Regions → Synchronous/Near-sync Replication → All Regions Serve Traffic
                              ↓
                    Automatic traffic redistribution during partial outage

Characteristics:
• RTO: < 1 minute (automatic load balancer failover)
• RPO: 0 to < 1 minute (synchronous replication)
• Cost: Highest (2-5x production infrastructure)
• Complexity: Highest (conflict resolution, data consistency, testing)

Best For:
• Mission-critical applications with zero-tolerance for downtime/data loss
• Global applications requiring low-latency access from multiple geographies

GCP Implementation:
• Cloud Spanner for globally consistent, horizontally scalable databases
• Global HTTP(S) Load Balancer with health-based traffic distribution
• Cloud CDN for edge caching with origin failover
• Conflict-free replicated data types (CRDTs) or application-level conflict resolution
```

### DR Strategy Selection Framework (PCA Exam Gold!)

```
When asked "Which DR strategy should we use?", evaluate:

1. Business Impact Analysis:
   • Revenue impact per minute of downtime → Drives RTO target
   • Data loss tolerance (transactions, user data) → Drives RPO target
   • Compliance requirements (HIPAA, PCI, GDPR) → May mandate minimum RTO/RPO

2. Technical Constraints:
   • Application architecture (stateless vs. stateful)
   • Data replication feasibility (sync vs. async, conflict resolution)
   • Dependency mapping (external APIs, third-party services)

3. Cost-Benefit Analysis:
   • DR infrastructure cost vs. potential loss from outage
   • Testing and maintenance overhead for each pattern
   • Egress costs for cross-region replication

4. Operational Readiness:
   • Team expertise with automation and failover procedures
   • Testing frequency and validation processes
   • Documentation and runbook completeness
```

### Sample Exam Question

> **Scenario**: A healthcare provider runs a patient portal with these requirements:
> - HIPAA compliance requires audit trails for all data access
> - Cannot lose more than 5 minutes of patient record updates
> - Must restore service within 30 minutes of regional outage
> - Budget allows DR infrastructure up to 2x production cost
> - Application is stateful with complex database relationships
>
> **Question**: Which DR strategy BEST meets these requirements?
>
> A) Backup & Restore with hourly Cloud SQL exports to Coldline storage  
> B) Pilot Light with Cloud SQL cross-region replica + automated promotion  
> C) Warm Standby with Global Load Balancer + Cloud SQL async replication every 2 minutes  
> D) Multi-Region Active-Active with Cloud Spanner + synchronous replication  
>
> **Answer**: C  
> **Rationale**: RPO ≤ 5 minutes requires replication interval ≤ 5 minutes; async replication every 2 minutes satisfies this. RTO ≤ 30 minutes requires pre-provisioned infrastructure with automated failover; warm standby with global load balancing meets this. Budget allows 2x cost, which aligns with warm standby multiplier. Cloud SQL supports HIPAA compliance with proper configuration. Option A cannot meet RTO/RPO; B has higher RTO due to scale-up time; D exceeds budget (3-5x cost) and Cloud Spanner may be overkill for complex relational data [[38]][[42]].

---

## Migration Center & Assessment Tools

### What is Migration Center?

Migration Center is Google Cloud's unified platform for discovering, assessing, planning, and executing migrations to Google Cloud [[35]].

### Migration Center Workflow

```
┌─────────────────┐
│   DISCOVERY     │
│ • Import inventory from VMware, AWS, Azure, on-prem |
│ • Collect performance metrics (CPU, memory, network) |
│ • Identify dependencies between workloads |
└────────┬────────┘
         ↓
┌─────────────────┐
│   ASSESSMENT    │
│ • TCO analysis: On-prem vs. GCP cost comparison |
│ • Technical fit: Compatibility with GCP services |
│ • Risk scoring: Complexity, dependencies, downtime tolerance |
│ • Recommendation: 6 R's classification per workload |
└────────┬────────┘
         ↓
┌─────────────────┐
│   PLANNING      │
│ • Wave planning: Group workloads by dependency/risk |
│ • Cutover strategy: Big bang vs. phased vs. parallel run |
│ • Testing plan: Validation criteria, rollback procedures |
│ • Timeline: Resource allocation, milestone tracking |
└────────┬────────┘
         ↓
┌─────────────────┐
│   EXECUTION     │
│ • Tool selection based on workload type (see below) |
│ • Automated replication and cutover |
│ • Monitoring and validation during migration |
└────────┬────────┘
         ↓
┌─────────────────┐
│   VALIDATION    │
│ • Post-migration testing: Functionality, performance, security |
│ • Business sign-off: Stakeholder acceptance |
│ • Decommissioning: Retire legacy systems after validation |
└─────────────────┘
```

### Migration Tool Selection by Workload Type

| Workload Type | Recommended Tool | Key Features | Limitations |
|--------------|-----------------|--------------|-------------|
| **VMware VMs** | Migrate for Compute Engine | Agentless replication, minimal downtime, test migrations | Requires VMware vCenter access; Windows/Linux only |
| **Physical Servers** | Migrate for Compute Engine (with agent) | Agent-based replication for bare metal | Requires OS-level agent installation |
| **AWS EC2 / Azure VMs** | Migrate for Compute Engine + Storage Transfer | Cross-cloud replication, network optimization | Requires source cloud API access |
| **Databases (MySQL, PostgreSQL, SQL Server)** | Database Migration Service (DMS) | Continuous replication, minimal downtime, SSL support | Source must be supported version; schema conversion may be needed |
| **Oracle Databases** | Database Migration Service (with limitations) | Homogeneous migration (Oracle→Oracle on GCE) | Heterogeneous migration (Oracle→Cloud SQL) requires manual effort |
| **Kubernetes Workloads** | Migrate for Anthos / GKE migration tools | Container image conversion, config adaptation | Requires containerization of legacy apps |
| **File Data** | Storage Transfer Service, rsync, third-party | Scheduled transfers, bandwidth throttling, verification | Large datasets require planning for transfer time |
| **SaaS Applications** | Repurchase or retain | Evaluate SaaS alternatives on GCP Marketplace | May require business process changes |

### Critical Exam Concepts

✅ **Assessment Phase Best Practices**:
```
• Inventory completeness: Ensure all workloads are discovered before planning
• Dependency mapping: Identify upstream/downstream dependencies to avoid broken migrations
• Performance baselining: Capture peak/average utilization for right-sizing in GCP
• Risk categorization: High-risk workloads (complex, critical) migrate later in waves
• Stakeholder alignment: Document acceptance criteria before execution begins

Exam Tip: Questions may ask "What should be done BEFORE migration planning?" → Complete discovery and assessment
```

✅ **TCO Analysis Components (Often Tested)**:
```
Total Cost of Ownership Comparison:

On-Premises Costs:
• Hardware depreciation (servers, storage, network)
• Data center facilities (power, cooling, space)
• Operations staff (administration, monitoring, patching)
• Licensing (OS, middleware, applications)
• Backup/DR infrastructure and testing

Google Cloud Costs:
• Compute (VMs, GKE, Cloud Run)
• Storage (Persistent Disk, Cloud Storage, databases)
• Network (egress, load balancing, CDN)
• Operations (monitoring, logging, support)
• Migration one-time costs (tools, consulting, testing)

Key Insight: Cloud often shifts CapEx to OpEx; focus on 3-5 year TCO, not just monthly costs
```

### Sample Exam Question

> **Scenario**: A manufacturing company wants to migrate 200+ VMware VMs to Google Cloud. Requirements:
> - Minimize downtime during cutover (< 15 minutes per VM)
> - Validate functionality before decommissioning on-premises
> - Migrate in waves based on business criticality
> - Track migration progress and costs centrally
>
> **Question**: Which Migration Center workflow BEST meets these requirements?
>
> A) Direct cutover using Storage Transfer Service for all VMs simultaneously  
> B) Discovery → Assessment → Wave planning → Migrate for Compute Engine with test migrations → Validation  
> C) Assessment → Immediate execution using Database Migration Service for all workloads  
> D) Planning → Manual VM export/import via Cloud Storage → Post-migration testing  
>
> **Answer**: B  
> **Rationale**: Complete workflow ensures proper discovery, risk-based wave planning, minimal-downtime replication via Migrate for Compute Engine, and validation before decommissioning. Test migrations validate the process before production cutover. Option A lacks assessment and testing; C uses wrong tool for VMs; D is manual and error-prone without proper tooling [[35]].

---

## Shared VPC: Architecture & Governance

### What is Shared VPC?

Shared VPC enables organizations to connect multiple projects to a common VPC network, allowing resources in different projects to communicate securely using internal IPs [[44]].

### Architecture Model

```
┌─────────────────────────────────┐
│        HOST PROJECT             │
│  (Network Administration)       │
│                                 │
│  • VPC network definition       │
│  • Subnets (regional)           │
│  • Routes, firewall rules       │
│  • Cloud Router, Cloud NAT      │
│  • IAM: roles/compute.networkAdmin │
└────────┬────────────────────────┘
         │ Grants subnet usage
         ↓
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ SERVICE PROJECT │ │ SERVICE PROJECT │ │ SERVICE PROJECT │
│ (App Team A)    │ │ (App Team B)    │ │ (App Team C)    │
│                 │ │                 │ │                 │
│ • GCE VMs       │ • GKE clusters    │ • Cloud Run       │
│ • Cloud SQL     │ • Memorystore     │ • Functions       │
│ • Attach to     │ • Attach to       │ • Attach to       │
│   host subnets  │   host subnets    │   host subnets    │
│                 │ │                 │ │                 │
│ IAM:            │ │ IAM:            │ │ IAM:            │
│ roles/compute. │ │ roles/compute. │ │ roles/compute. │
│ networkUser    │ │ networkUser    │ │ networkUser    │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### Critical IAM Roles (Memorize for Exam!)

| Role | Scope | Permissions | Typical Assignee |
|------|-------|------------|-----------------|
| `roles/compute.networkAdmin` | Host project | Full network configuration (create/modify subnets, firewalls, routes) | Network engineering team |
| `roles/compute.networkUser` | Service project (or specific subnet) | Attach resources to subnets; view network config | Application development teams |
| `roles/compute.securityAdmin` | Host project | Manage firewall rules and SSL policies | Security team |
| `roles/compute.loadBalancerAdmin` | Host project | Configure load balancers using shared subnets | Platform engineering team |

✅ **Best Practices for Shared VPC Design**:
```
1. Subnet Strategy:
   • Regional subnets for high availability
   • Separate subnets per environment (prod, stage, dev) or team
   • Use secondary IP ranges for GKE pods/services

2. Firewall Rule Management:
   • Use network tags or service accounts for granular rules
   • Document rule purpose and owner in description field
   • Regularly audit rules for least privilege

3. Service Project Onboarding:
   • Standardized process for granting networkUser role
   • Template for resource deployment in shared subnets
   • Documentation for network troubleshooting

4. Monitoring and Logging:
   • Enable VPC Flow Logs for security auditing
   • Use Cloud Monitoring for network performance metrics
   • Centralize logs in host project for cross-project visibility
```

### Common Exam Scenarios

✅ **Scenario: Multi-Team Application Deployment**
```
Requirement: Three teams need to deploy microservices that communicate via internal IPs, but each team manages their own CI/CD and IAM.

Solution Pattern:
1. Create Shared VPC host project with regional subnets
2. Grant each team's service project roles/compute.networkUser on appropriate subnets
3. Teams deploy GKE/Cloud Run resources attached to shared subnets
4. Use service accounts + IAM for application-layer authorization
5. Implement firewall rules based on service accounts for zero-trust networking

Exam Tip: Questions may ask "How to enable cross-project communication without exposing public IPs?" → Shared VPC + internal load balancing
```

✅ **Scenario: Centralized Network Security**
```
Requirement: Security team must enforce firewall policies across all application projects.

Solution Pattern:
1. Security team holds roles/compute.networkAdmin in host project
2. Define firewall rules in host project using network tags/service accounts
3. Application teams deploy resources with appropriate tags/service accounts
4. Use VPC Service Controls perimeter to prevent data exfiltration
5. Enable Firewall Rules Logging for audit compliance

Exam Tip: Questions may ask "How to ensure consistent security policies across projects?" → Shared VPC with centralized firewall management
```

### Sample Exam Question

> **Scenario**: A financial services company has:
> - Central network team managing security and connectivity
> - Five application teams deploying independently to different projects
> - Requirement: All applications must communicate via internal IPs only
> - Compliance: Firewall rules must be auditable and centrally controlled
>
> **Question**: Which architecture BEST meets these requirements?
>
> A) Each project has its own VPC with VPC Peering and centralized firewall management via third-party tool  
> B) Shared VPC with host project for network resources and service projects for applications + IAM delegation  
> C) Single project with all resources and team-based IAM folders for isolation  
> D) Independent VPCs with Cloud VPN connections and distributed firewall management  
>
> **Answer**: B  
> **Rationale**: Shared VPC enables centralized network management (firewall rules, subnets) in host project while allowing application teams autonomy in service projects. Internal IP communication is native to Shared VPC. IAM roles (networkAdmin/networkUser) enforce separation of duties. Option A adds peering complexity; C lacks project-level isolation for billing/IAM; D requires VPN overhead and distributed management [[44]].

---

## Phased Migration Strategies

### The "6 R's" Migration Framework (Exam Essential) [[40]]

```
1. Rehost ("Lift-and-Shift"):
   • Description: Move VMs/applications to GCP with minimal changes
   • Tools: Migrate for Compute Engine, VM import/export
   • Pros: Fastest migration, minimal application changes
   • Cons: Doesn't leverage cloud-native benefits; may have higher TCO long-term
   • Best For: Time-sensitive migrations, legacy apps with complex dependencies

2. Replatform ("Lift, Tinker, and Shift"):
   • Description: Minor optimizations during migration (managed DB, containerization)
   • Tools: Database Migration Service, Anthos, Cloud Run migration
   • Pros: Some cloud benefits with moderate effort; reduced operational overhead
   • Cons: Requires some application modification; longer timeline than rehost
   • Best For: Applications with clear optimization opportunities (e.g., move to managed DB)

3. Refactor ("Re-architect"):
   • Description: Significant redesign for cloud-native architecture (microservices, serverless)
   • Tools: Cloud Run, GKE, Cloud Functions, Cloud Build
   • Pros: Maximum cloud benefits (scalability, resilience, cost optimization)
   • Cons: Highest effort, longest timeline, requires development resources
   • Best For: Strategic applications where cloud-native benefits justify investment

4. Repurchase:
   • Description: Replace with SaaS or commercial cloud-native solution
   • Tools: GCP Marketplace, third-party SaaS integrations
   • Pros: Fastest path to cloud benefits; vendor manages operations
   • Cons: May require business process changes; vendor lock-in considerations
   • Best For: Commodity applications (CRM, email, collaboration)

5. Retire:
   • Description: Decommission unused or redundant applications
   • Tools: Inventory analysis, stakeholder interviews
   • Pros: Immediate cost savings; reduces migration scope
   • Cons: Requires business approval; may uncover hidden dependencies
   • Best For: Applications with no active users or replaced by newer systems

6. Retain:
   • Description: Keep on-premises or in current environment (hybrid approach)
   • Tools: Cloud Interconnect, Cloud VPN, Anthos for hybrid management
   • Pros: Avoids migration risk for complex/regulated workloads
   • Cons: Ongoing on-premises costs; hybrid complexity
   • Best For: Applications with regulatory constraints, extreme latency requirements, or pending replacement
```

### Wave Planning Methodology

```
Step 1: Group Workloads by Dependency and Risk
   • Wave 0: Foundation (networking, IAM, monitoring, logging)
   • Wave 1: Low-risk, low-dependency workloads (dev/test, internal tools)
   • Wave 2: Medium-risk workloads with manageable dependencies
   • Wave 3: High-risk, business-critical workloads (after validation of earlier waves)
   • Wave 4: Decommissioning of legacy systems (after successful migration)

Step 2: Define Wave Criteria
   • Technical: Dependency mapping, migration complexity, testing requirements
   • Business: Criticality, stakeholder availability, change freeze windows
   • Operational: Team capacity, support coverage, rollback readiness

Step 3: Establish Success Metrics per Wave
   • Technical: Performance benchmarks, error rates, security compliance
   • Business: User acceptance, business process validation, SLA adherence
   • Operational: Support ticket volume, incident response time

Step 4: Plan Rollback Procedures
   • Pre-cutover backups of source systems
   • DNS TTL reduction before cutover for faster rollback
   • Documented rollback steps with estimated time
   • Stakeholder communication plan for rollback scenarios
```

### Cutover Strategies Comparison

| Strategy | Description | Pros | Cons | Best For |
|----------|-------------|------|------|----------|
| **Big Bang** | Migrate all workloads in single cutover window | Simple coordination; clean break from legacy | High risk; extended downtime if issues occur | Small, simple environments; non-critical workloads |
| **Phased** | Migrate workloads in sequential waves | Lower risk per wave; learn and adapt between waves | Longer overall timeline; temporary hybrid complexity | Most enterprise migrations; complex dependencies |
| **Parallel Run** | Run legacy and cloud environments simultaneously; switch traffic gradually | Minimal risk; validate functionality before full cutover | Highest cost (duplicate infrastructure); data sync complexity | Mission-critical applications; regulatory requirements |
| **Pilot** | Migrate representative subset first; refine process before full migration | Validate migration approach; build team confidence | Doesn't migrate production workloads initially | First-time migrations; unproven tooling/processes |

### Sample Exam Question

> **Scenario**: A retail company is migrating 500+ applications to Google Cloud. Requirements:
> - Business-critical e-commerce platform must have zero downtime during migration
> - Less critical internal tools can tolerate brief maintenance windows
> - Migration must be completed within 18 months
> - Team has limited cloud migration experience
>
> **Question**: Which migration strategy BEST balances risk, timeline, and team capability?
>
> A) Big Bang cutover for all applications during annual maintenance window  
> B) Pilot migration of internal tools → Refine process → Phased migration of business apps with parallel run for e-commerce  
> C) Refactor all applications to cloud-native before migration to maximize long-term benefits  
> D) Retain all applications on-premises and use Cloud Interconnect for hybrid access  
>
> **Answer**: B  
> **Rationale**: Pilot migration builds team experience with low-risk workloads. Phased approach manages complexity and timeline. Parallel run for e-commerce ensures zero downtime for business-critical system. This balances risk mitigation with practical timeline. Option A is high-risk for inexperienced team; C extends timeline beyond 18 months; D avoids migration benefits [[40]].

---

## Scenario-Based Practice Quizzes

### Quiz 1: Global Financial Services Migration

> **Scenario**: A global bank is migrating trading systems to Google Cloud:
> - 200+ applications with complex interdependencies
> - Regulatory requirements: RTO ≤ 15 minutes, RPO = 0 for core trading systems
> - Data residency: EU customer data must remain in EU regions
> - Timeline: Complete migration in 24 months with minimal business disruption
>
> **Question 1**: Which DR strategy should be implemented for core trading systems?
>
> A) Backup & Restore with daily snapshots to EU Coldline storage  
> B) Pilot Light with Cloud SQL cross-region replica in EU  
> C) Warm Standby with Global Load Balancer + async replication every 5 minutes  
> D) Multi-Region Active-Active with Cloud Spanner + synchronous replication within EU regions  
>
> **Answer**: D  
> **Rationale**: RPO = 0 requires synchronous replication; RTO ≤ 15 minutes requires automated failover with pre-provisioned infrastructure. Multi-region active-active with Cloud Spanner provides both. EU region constraint is satisfied by selecting EU regions for both active sites. Other options cannot meet RPO = 0 requirement [[38]][[42]].
>
> **Question 2**: How should Shared VPC be designed to meet data residency and team autonomy requirements?
>
> A) Single global Shared VPC with subnets in all regions; grant all teams networkUser access  
> B) Separate Shared VPC host projects per region (EU, US, APAC) with regional subnet allocation  
> C) Independent VPCs per team with VPC Peering and centralized firewall management  
> D) Single host project with EU-only subnets; restrict non-EU teams to EU resources via IAM conditions  
>
> **Answer**: B  
> **Rationale**: Regional host projects enable data residency enforcement at the network layer. EU teams use EU host project; other regions have separate hosts. This prevents accidental cross-region data flow while maintaining team autonomy within regions. Option A violates data residency; C adds peering complexity; D restricts legitimate non-EU workloads [[44]].
>
> **Question 3**: Which migration wave planning approach BEST manages risk for this scenario?
>
> A) Wave 1: Core trading systems; Wave 2: All other applications  
> B) Wave 0: Foundation (network, IAM); Wave 1: Non-critical internal tools; Wave 2: Customer-facing apps; Wave 3: Core trading systems with parallel run  
> C) Migrate all applications simultaneously using Big Bang cutover during low-traffic period  
> D) Retain core trading systems on-premises; migrate only non-critical applications  
>
> **Answer**: B  
> **Rationale**: Foundation first establishes secure, compliant platform. Low-risk workloads build team experience. Customer-facing apps validate migration process before core systems. Parallel run for core trading systems ensures zero downtime. This phased approach balances risk mitigation with timeline requirements. Option A migrates highest-risk first; C is high-risk for complex environment; D avoids migration benefits for critical systems [[40]].

### Quiz 2: Healthcare Provider DR Design

> **Scenario**: A healthcare network runs patient management systems with:
> - HIPAA compliance requirements for audit trails and encryption
> - Business requirement: RTO ≤ 1 hour, RPO ≤ 10 minutes for patient record systems
> - Budget constraint: DR infrastructure cost ≤ 1.8x production
> - Current architecture: Single-region deployment with daily backups
>
> **Question 1**: Which backup strategy ensures RPO ≤ 10 minutes?
>
> A) Persistent Disk snapshots every hour with cross-region copy  
> B) Cloud SQL backups with point-in-time recovery enabled  
> C) Continuous async replication to secondary region with 5-minute checkpoint intervals  
> D) Daily exports to Cloud Storage with versioning enabled  
>
> **Answer**: C  
> **Rationale**: RPO ≤ 10 minutes requires replication frequency ≤ 10 minutes. Continuous async replication with 5-minute checkpoints satisfies this. Persistent Disk snapshots hourly (A) and daily exports (D) exceed RPO tolerance. Cloud SQL point-in-time recovery (B) can restore to any second but requires manual intervention and may not meet RTO. Continuous replication enables automated failover [[38]].
>
> **Question 2**: How to implement HIPAA-compliant audit logging for DR failover events?
>
> A) Enable Cloud Audit Logs for all projects + export to BigQuery with 7-year retention  
> B) Use application-level logging only; rely on database transaction logs  
> C) Enable VPC Flow Logs + Cloud Monitoring metrics for network events  
> D) Configure Cloud Logging exclusions to reduce costs; retain only error-level logs  
>
> **Answer**: A  
> **Rationale**: Cloud Audit Logs capture administrative and data access events required for HIPAA audits. Export to BigQuery enables long-term retention and SQL-based analysis. Application logs alone (B) miss infrastructure events; VPC Flow Logs (C) don't capture application/data access; exclusions (D) risk losing required audit data [[28]].
>
> **Question 3**: Which DR architecture BEST balances RTO/RPO requirements with budget constraint?
>
> A) Multi-Region Active-Active with synchronous replication (cost multiplier: 4x)  
> B) Warm Standby with Global Load Balancer + async replication every 5 minutes (cost multiplier: 2x)  
> C) Pilot Light with automated scale-up + async replication every 15 minutes (cost multiplier: 1.5x)  
> D) Backup & Restore with hourly snapshots (cost multiplier: 1.2x)  
>
> **Answer**: C  
> **Rationale**: RTO ≤ 1 hour allows for scale-up time; Pilot Light with automation can meet this. RPO ≤ 10 minutes requires replication ≤ 10 minutes; 15-minute interval is slightly over but may be acceptable with business approval (or optimize to 10 min). Cost multiplier 1.5x fits within 1.8x budget. Option A exceeds budget; B is at budget limit with less margin; D cannot meet RPO [[38]].

### Quiz 3: Startup Phased Migration

> **Scenario**: A fast-growing startup with limited DevOps resources:
> - 50 applications: 10 customer-facing, 40 internal tools
> - No formal DR strategy currently; RTO/RPO not defined
> - Goal: Establish cloud foundation while migrating with minimal disruption
> - Constraint: Small team; cannot dedicate full-time to migration
>
> **Question 1**: What should be the FIRST step in establishing DR capability?
>
> A) Implement Multi-Region Active-Active for all applications immediately  
> B) Define RTO/RPO targets through business impact analysis with stakeholders  
> C) Migrate all applications to GCP first; design DR after migration completes  
> D) Purchase third-party DR-as-a-Service solution to outsource complexity  
>
> **Answer**: B  
> **Rationale**: DR design must start with business requirements. Defining RTO/RPO through stakeholder alignment ensures technical solutions match business needs. Implementing DR without requirements (A) risks over/under-engineering. Migrating first (C) delays critical risk mitigation. Outsourcing (D) may not align with cloud strategy and adds vendor dependency.
>
> **Question 2**: Which migration approach BEST fits the team's resource constraints?
>
> A) Refactor all applications to cloud-native microservices before migration  
> B) Rehost customer-facing apps first using Migrate for Compute Engine; replatform internal tools later  
> C) Retain all applications on-premises; use Cloud Interconnect for hybrid access only  
> D) Repurchase all applications with SaaS alternatives from GCP Marketplace  
>
> **Answer**: B  
> **Rationale**: Rehosting customer-facing apps first addresses business-critical needs with minimal changes. Migrate for Compute Engine automates VM migration, reducing team effort. Internal tools can be replatformed later as resources allow. This balances business priority with team capacity. Option A requires significant development effort; C avoids cloud benefits; D may not be feasible for custom applications [[40]].
>
> **Question 3**: How to implement cost-effective backup for internal tools with RTO ≤ 4 hours, RPO ≤ 1 hour?
>
> A) Multi-region active-active deployment for all internal tools  
> B) Persistent Disk snapshots every 30 minutes + automated restore scripts to secondary region  
> C) Daily exports to Coldline storage with manual restore procedures  
> D) No backups; rely on application-level data export features  
>
> **Answer**: B  
> **Rationale**: Snapshots every 30 minutes satisfy RPO ≤ 1 hour. Automated restore scripts enable RTO ≤ 4 hours without manual intervention. This provides balanced protection at moderate cost. Option A is overkill for internal tools; C cannot meet RPO; D provides no recovery capability [[23]].

---

## Quick Reference Tables

### RTO/RPO → DR Strategy Quick Reference

| Business Requirement | RTO Target | RPO Target | Recommended Pattern | Key GCP Services |
|---------------------|------------|------------|-------------------|-----------------|
| Non-critical internal tool | > 24 hours | > 24 hours | Backup & Restore | Persistent Disk snapshots, Cloud Storage |
| Business application | 4-24 hours | 1-24 hours | Pilot Light | Cloud SQL replicas, Instance Templates, Cloud DNS |
| Customer-facing app | 1-4 hours | 15 min-1 hour | Warm Standby | Global Load Balancer, async replication, automated failover |
| Mission-critical system | < 1 hour | < 15 minutes | Multi-Region Active-Active | Cloud Spanner, global LB, synchronous replication |
| Zero data loss required | Any | = 0 | Multi-Region Active-Active + sync replication | Cloud Spanner, application-level conflict resolution |

### Migration Tool Selection Matrix

| Source Environment | Target on GCP | Recommended Tool | Minimal Downtime? | Notes |
|-------------------|---------------|-----------------|-------------------|-------|
| VMware vSphere | Compute Engine | Migrate for Compute Engine | ✅ Yes (agentless replication) | Requires vCenter access |
| Physical servers | Compute Engine | Migrate for Compute Engine (agent) | ✅ Yes | Requires OS agent installation |
| AWS EC2 / Azure VM | Compute Engine | Migrate for Compute Engine + Storage Transfer | ✅ Yes | Requires source cloud API credentials |
| MySQL / PostgreSQL | Cloud SQL | Database Migration Service | ✅ Yes (continuous replication) | Source version must be supported |
| SQL Server | Cloud SQL / SQL on GCE | Database Migration Service | ✅ Yes | License compliance verification required |
| Oracle | Oracle on GCE | Database Migration Service (homogeneous) | ✅ Yes | Heterogeneous migration requires manual effort |
| On-prem Kubernetes | GKE | Migrate for Anthos / manual migration | ⚠️ Limited | Requires containerization assessment |
| File shares / NAS | Filestore / Cloud Storage | Storage Transfer Service / rsync | ❌ No (scheduled transfers) | Plan for transfer duration |

### Shared VPC IAM Role Reference

| Role | Permission Scope | Typical Use Case | Assignment Level |
|------|-----------------|-----------------|-----------------|
| `roles/compute.networkAdmin` | Host project: Full network config | Network team managing VPC, subnets, firewalls | Host project |
| `roles/compute.networkUser` | Service project: Attach to subnets | App teams deploying resources to shared network | Service project or specific subnet |
| `roles/compute.securityAdmin` | Host project: Firewall/SSL policies | Security team enforcing network policies | Host project |
| `roles/compute.loadBalancerAdmin` | Host project: LB configuration | Platform team managing traffic distribution | Host project |
| `roles/compute.instanceAdmin.v1` | Service project: VM management | App teams managing their own compute resources | Service project |

### Phased Migration Wave Planning Template

```
Wave 0: Foundation (Weeks 1-4)
├─ Objectives: Establish secure, compliant cloud foundation
├─ Workloads: Networking (Shared VPC), IAM, logging/monitoring, backup policies
├─ Success Criteria: Security review passed; baseline monitoring operational
└─ Rollback: Not applicable (foundation only)

Wave 1: Low-Risk Validation (Weeks 5-8)
├─ Objectives: Validate migration process with minimal business impact
├─ Workloads: Dev/test environments, internal tools with no external dependencies
├─ Success Criteria: All workloads functional; performance within 10% of baseline
└─ Rollback: Revert to on-premises within 1 hour if critical issues

Wave 2: Business Applications (Weeks 9-16)
├─ Objectives: Migrate customer-facing applications with controlled risk
├─ Workloads: Web applications, APIs with moderate dependencies
├─ Success Criteria: Zero data loss; RTO/RPO targets met during testing
└─ Rollback: DNS-based failback to on-premises within 30 minutes

Wave 3: Critical Systems (Weeks 17-24)
├─ Objectives: Migrate mission-critical systems with maximum safeguards
├─ Workloads: Core databases, transaction processing systems
├─ Success Criteria: Parallel run validation; business sign-off before cutover
└─ Rollback: Pre-tested procedure with < 15 minute execution time

Wave 4: Decommissioning (Weeks 25-28)
├─ Objectives: Retire legacy infrastructure after successful validation
├─ Workloads: On-premises systems with confirmed GCP replacements
├─ Success Criteria: Cost savings realized; no residual dependencies
└─ Rollback: Not applicable (post-validation)
```

### Common Exam Keywords → Service Mapping

| Keyword in Question | Likely Service/Concept | Why |
|--------------------|----------------------|-----|
| "RTO ≤ 30 minutes", "zero downtime" | Multi-region active-active, Global Load Balancer | Requires automated failover with pre-provisioned infrastructure |
| "RPO = 0", "no data loss" | Synchronous replication, Cloud Spanner | Only sync replication guarantees zero data loss |
| "minimize DR costs", "budget constraint" | Pilot Light or Backup/Restore | Lower infrastructure multiplier (1.2-1.5x vs 3-5x) |
| "cross-project internal communication" | Shared VPC + networkUser role | Enables private IP connectivity across projects |
| "centralized firewall management" | Shared VPC host project + networkAdmin role | Single location for policy definition and enforcement |
| "VMware migration", "minimal downtime" | Migrate for Compute Engine | Agentless replication with cutover flexibility |
| "database migration", "continuous replication" | Database Migration Service | Native support for ongoing sync with minimal downtime |
| "phased migration", "risk mitigation" | Wave planning with Pilot → Phased approach | Builds confidence before migrating critical workloads |
| "data residency", "EU-only" | Regional resources + location constraints | Enforces geographic boundaries at service configuration level |
| "HIPAA audit trail", "compliance logging" | Cloud Audit Logs + BigQuery export | Captures required administrative and data access events |

---

## Final Exam Day Checklist: Backup/DR & Migration Section ✅

### 24 Hours Before
- [ ] Memorize RTO/RPO definitions and business impact examples
- [ ] Review DR pattern cost multipliers and use cases
- [ ] Practice mapping business statements to technical RTO/RPO targets
- [ ] Revisit Shared VPC IAM roles and host/service project model
- [ ] Review Migration Center workflow phases and tool selection criteria

### During Backup/DR/Migration Questions
- [ ] Extract RTO/RPO from scenario (explicit or implied from business impact)
- [ ] Match to appropriate DR pattern using cost/benefit framework
- [ ] Verify data residency and compliance constraints before selecting regions
- [ ] For migration questions: Identify workload type → select appropriate tool
- [ ] For Shared VPC: Confirm host project owns network; service projects attach resources

### Red Flags to Double-Check
```
⚠️ "Use Backup & Restore for RTO ≤ 15 minutes" → Wrong (too slow)
⚠️ "Multi-region active-active for non-critical internal tool" → Wrong (over-engineered)
⚠️ "Grant networkAdmin to application teams in Shared VPC" → Wrong (violates least privilege)
⚠️ "Big Bang migration for 500+ complex applications" → Wrong (high risk)
⚠️ "Database Migration Service for Oracle to Cloud SQL heterogeneous migration" → Wrong (limited support)
⚠️ "Skip assessment phase to accelerate migration timeline" → Wrong (increases failure risk)
⚠️ "Use global Shared VPC for EU data residency requirement" → Wrong (data may replicate globally)
```

> 💡 **Pro Tip**: When stuck between two DR/migration answers, choose the one that:  
> (1) Explicitly addresses the RTO/RPO or business impact stated in the scenario, AND  
> (2) Respects compliance/data residency constraints mentioned, AND  
> (3) Balances technical solution with practical constraints (budget, timeline, team skills)

---

## Appendix: Quick Command Reference

### Persistent Disk Snapshot Management
```bash
# Create snapshot with application-consistent flag
gcloud compute disks snapshot DISK_NAME \
  --snapshot-names=backup-$(date +%Y%m%d) \
  --storage-locations=us-central1 \
  --guest-flush  # Quiesce filesystem for consistency

# Copy snapshot to another region for DR
gcloud compute snapshots move SOURCE_SNAPSHOT \
  --target-storage-locations=europe-west1

# Set retention policy using lifecycle management
# (Configure via Cloud Console or gsutil lifecycle commands on snapshot bucket)
```

### Cloud SQL Backup & DR Configuration
```bash
# Enable automated backups with point-in-time recovery
gcloud sql instances patch INSTANCE_NAME \
  --backup-start-time=02:00 \
  --enable-point-in-time-recovery \
  --retained-backups=7

# Create cross-region read replica for DR
gcloud sql instances create-replica REPLICA_NAME \
  --master-instance-name=INSTANCE_NAME \
  --region=europe-west1 \
  --replication-type=ASYNCHRONOUS

# Promote replica to standalone during failover
gcloud sql instances promote-replica REPLICA_NAME
```

### Shared VPC Configuration
```bash
# Enable Shared VPC on host project
gcloud compute shared-vpc enable HOST_PROJECT_ID

# Attach service project to host
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
  --host-project=HOST_PROJECT_ID

# Grant networkUser role to service project for specific subnet
gcloud compute networks subnets add-iam-policy-binding SUBNET_NAME \
  --region=REGION \
  --member=project:SERVICE_PROJECT_ID \
  --role=roles/compute.networkUser \
  --project=HOST_PROJECT_ID
```

### Migration Center CLI Essentials
```bash
# Import inventory from VMware vCenter
gcloud migrationcenter import-jobs create vmware-import \
  --source=vmware \
  --vcenter-host=VCENTER_HOST \
  --vcenter-username=USER \
  --location=us-central1

# Generate assessment report
gcloud migrationcenter assessments create assessment-1 \
  --source-inventory=vmware-import \
  --location=us-central1

# View TCO comparison
gcloud migrationcenter assessments describe assessment-1 \
  --location=us-central1 \
  --format="value(tco_comparison)"
```

---

*This study guide is based on official Google Cloud documentation and PCA exam objectives as of April 2026. Always verify with the [official exam guide](https://cloud.google.com/certification/cloud-architect) for updates.*

> 🔄 **Stay Updated**: DR and migration capabilities evolve rapidly. Subscribe to:
> - [Migration Center documentation](https://cloud.google.com/migration-center)
> - [Disaster Recovery planning guide](https://cloud.google.com/architecture/disaster-recovery)
> - [Shared VPC best practices](https://cloud.google.com/vpc/docs/shared-vpc)

---
*© 2026 PCA Study Materials. For educational purposes only. Not affiliated with Google Cloud.*