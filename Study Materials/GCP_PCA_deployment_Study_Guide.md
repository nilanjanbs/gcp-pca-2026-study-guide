# Google Cloud Professional Cloud Architect (PCA) Exam Study Guide
## Deployment, Artifact Management & Infrastructure as Code Domain

> **Target Certification**: Google Cloud Professional Cloud Architect (PCA)  
> **Focus Area**: Cloud Build, Artifact Registry, Container Registry, Deployment Manager, Infrastructure as Code  
> **Last Updated**: April 2026  
> **Format**: Markdown (.md)

---

## 📋 Table of Contents

1. [Exam Overview & Domain Weighting](#exam-overview--domain-weighting)
2. [PCA Exam Tips for Deployment & IaC Questions](#pca-exam-tips-for-deployment--iac-questions)
3. [Cloud Deployment Manager (Deprecation Notice)](#cloud-deployment-manager-deprecation-notice)
4. [Infrastructure Manager: The Replacement](#infrastructure-manager-the-replacement)
5. [Google Artifact Registry (GAR)](#google-artifact-registry-gar)
6. [Google Container Registry (GCR) - Legacy](#google-container-registry-gcr---legacy)
7. [Cloud Build: CI/CD Platform](#cloud-build-cicd-platform)
8. [Infrastructure as Code Tool Comparison](#infrastructure-as-code-tool-comparison)
9. [Scenario-Based Practice Quizzes](#scenario-based-practice-quizzes)
10. [Quick Reference Tables](#quick-reference-tables)

---

## Exam Overview & Domain Weighting

### PCA Exam Domain Structure (2026)

| Domain | Approximate Weight | Key Topics for This Guide |
|--------|-------------------|---------------------------|
| Designing & Planning Cloud Solution Architecture | ~24-30% | IaC strategy, artifact storage design, CI/CD architecture |
| **Managing & Provisioning a Solution Infrastructure** | **~18-25%** | **Cloud Build, Artifact Registry, Deployment automation, Infrastructure Manager** [[46]][[48]] |
| Designing for Security & Compliance | ~20% | Supply chain security, build provenance, repository access controls |
| Analyzing & Optimizing Technical/Business Processes | ~15% | Build cost optimization, deployment velocity, artifact lifecycle management |
| Managing Implementation | ~10% | CI/CD pipeline design, migration strategies, rollback procedures |

> ⚠️ **Critical Note**: The "Managing & Provisioning" domain carries significant weight (~18-25%) and is where most deployment/IaC questions appear [[63]]. However, deployment concepts also appear in security (supply chain), design (architecture patterns), and optimization (cost/velocity) domains.

### Expected Question Distribution for This Domain

```
📊 Approximate Question Count (of 50-60 total):
• Cloud Build configuration & triggers: 3-5 questions
• Artifact Registry vs Container Registry: 2-4 questions  
• Infrastructure as Code tool selection: 2-3 questions
• Deployment Manager migration/alternatives: 1-2 questions
• Supply chain security (SLSA, provenance): 2-3 questions
• CI/CD pipeline design scenarios: 3-4 questions

Total: ~13-21 questions (22-35% of exam) may touch on these topics
```

### Exam Format Reminder
- **Duration**: 2 hours (120 minutes) [[57]]
- **Questions**: 50-60 multiple-choice/multiple-select
- **Case Studies**: 20-30% of questions reference fictional scenarios [[5]]
- **Passing Score**: Not publicly disclosed (scaled scoring, estimated ~70%) [[67]]

---

## PCA Exam Tips for Deployment & IaC Questions 🎯

### 🧠 Strategic Approach

1. **Know the Deprecation Timeline (Critical!)**
   ```
   Cloud Deployment Manager → End of Support: March 31, 2026 [[web_extractor]]
   
   Exam Impact: Questions may ask:
   • "What should you migrate to?" → Infrastructure Manager or Terraform
   • "Why is Deployment Manager not recommended?" → Deprecation, limited multi-cloud
   • "Legacy system uses DM; what's the migration path?" → Infrastructure Manager
   ```

2. **Apply the "Build → Store → Deploy" Mental Model**
   ```
   Source Code → Cloud Build (build/test) → Artifact Registry (store) → Target Runtime (deploy)
   
   When designing solutions, verify each stage:
   • Build: Triggers, pools, security, provenance
   • Store: Repository type, location, access control, vulnerability scanning
   • Deploy: Target platform compatibility, rollback strategy, promotion workflow
   ```

3. **Master the Artifact Registry vs Container Registry Decision Tree**
   ```
   New project? → Use Artifact Registry (recommended) [[23]]
   Existing GCR? → Plan migration to Artifact Registry [[29]]
   Need multi-format support? → Artifact Registry (Docker, Maven, npm, etc.)
   Need fine-grained repo control? → Artifact Registry (repo-level IAM)
   Simple container-only, legacy? → GCR still works but not recommended
   ```

4. **Cloud Build: Know the Pool Types**
   | Pool Type | Network Access | Customization | Best For |
   |-----------|---------------|---------------|----------|
   | **Default Pool** | Public internet only | Limited (machine type, disk) | Public repos, simple builds |
   | **Private Pool** | VPC/private network | Full (custom images, network config) | Internal repos, secure builds, on-prem integration |

5. **SLSA Compliance is Explicitly Tested** [[30]]
   ```
   SLSA Level Requirements for PCA:
   • Level 1: Scripted/automated builds (Cloud Build qualifies)
   • Level 2: Build service (not local dev env) + provenance generation
   • Level 3: Hardened build service + isolated builds + verified provenance
   
   Cloud Build Features:
   ✅ Ephemeral build environments (per-build VMs)
   ✅ Build provenance generation (SLSA L3)
   ✅ Binary Authorization integration
   ✅ Customer-managed encryption keys (CMEK)
   ```

### 🔍 Question Analysis Framework

```
When you see a deployment/IaC question:
1. IDENTIFY the workflow stage (build? store? deploy? migrate?)
2. LOCATE constraints (security? cost? compliance? legacy compatibility?)
3. MATCH to appropriate GCP service(s) with trade-off analysis
4. VERIFY supply chain security requirements (SLSA, provenance, signing)
5. CHECK for deprecation/migration implications (Deployment Manager → Infrastructure Manager)
```

### 🚫 Common Pitfalls to Avoid

| Mistake | Correct Approach |
|---------|-----------------|
| Recommending Deployment Manager for new projects | Recommend Infrastructure Manager or Terraform [[11]] |
| Using GCR for new container storage projects | Use Artifact Registry (GAR) as the recommended service [[23]] |
| Ignoring build pool network requirements | Choose Private Pool for VPC-SC or internal resource access |
| Assuming Cloud Build can directly deploy to any target | Verify target compatibility (GKE, Cloud Run, Compute Engine supported) |
| Overlooking artifact promotion workflows | Design multi-repo strategy (dev/stage/prod) with promotion gates |
| Forgetting build provenance for compliance | Enable SLSA L3 provenance for regulated workloads |

---

## Cloud Deployment Manager (Deprecation Notice) ⚠️

### Critical Exam Fact: End of Support

```
🚨 Cloud Deployment Manager reaches end of support on March 31, 2026 [[web_extractor]]

If you currently use Deployment Manager, migrate to:
• Infrastructure Manager (Google's Terraform-based service)
• Terraform with Google Cloud provider
• Other IaC tools (Pulumi, Ansible, Crossplane)
```

### What Was Cloud Deployment Manager?

Cloud Deployment Manager was Google Cloud's native infrastructure deployment service that automated creation and management of GCP resources using YAML/Python/Jinja templates [[11]].

### Key Concepts (For Legacy/Scenario Questions)

```yaml
Core Components:
  - Templates: Reusable configuration files (YAML, Python, Jinja2)
  - Configurations: Instance of a template with specific parameters
  - Deployments: Running instances of configurations
  - Runtime Configurator: (Deprecated) Store deployment-time variables

Template Structure Example:
resources:
- name: my-vm
  type: compute.v1.instance
  properties:
    zone: us-central1-a
    machineType: zones/us-central1-a/machineTypes/n1-standard-1
    disks:
    - deviceName: boot
      boot: true
      autoDelete: true
      initializeParams:
        sourceImage: projects/debian-cloud/global/images/family/debian-11
    networkInterfaces:
    - network: global/networks/default
```

### Why Was It Deprecated? (Exam Rationale Questions)

✅ **Limitations That Led to Deprecation**:
- Google Cloud-native only (no multi-cloud support)
- Limited community/ecosystem vs. Terraform
- Less mature state management and drift detection
- Google shifting to Terraform-based Infrastructure Manager

✅ **Migration Path Questions**:
```
Scenario: "Legacy system uses Deployment Manager templates. 
What is the recommended migration strategy?"

Answer Framework:
1. Assess template complexity and dependencies
2. Choose target: Infrastructure Manager (native) or Terraform (multi-cloud)
3. Use conversion tools or manual rewrite
4. Test in staging with parallel deployments
5. Cutover with rollback plan
```

### Sample Exam Question

> **Scenario**: A company has 50+ Deployment Manager templates managing their GCP infrastructure. Deployment Manager support ends March 2026. They need a solution that:
> - Supports existing YAML templates with minimal changes
> - Provides Google Cloud-native integration
> - Enables future multi-cloud expansion
>
> **Question**: Which migration strategy BEST balances short-term compatibility and long-term flexibility?
>
> A) Rewrite all templates in Terraform immediately  
> B) Migrate to Infrastructure Manager first, then evaluate Terraform for multi-cloud  
> C) Continue using Deployment Manager with extended support contract  
> D) Switch to Pulumi for Python-based template compatibility  
>
> **Answer**: B  
> **Rationale**: Infrastructure Manager is Google's Terraform-based replacement for Deployment Manager, offering template compatibility and native integration. Starting there minimizes immediate disruption while allowing future evaluation of Terraform for multi-cloud needs. Option A is high-risk for large template libraries; C is impossible (no extended support); D adds new tooling complexity without addressing the deprecation timeline [[11]][[39]].

---

## Infrastructure Manager: The Replacement

### What is Infrastructure Manager?

Infrastructure Manager is Google Cloud's fully managed service for deploying and managing infrastructure using Terraform configurations [[37]].

### Key Exam Concepts

```yaml
Core Features:
  - Terraform-Native: Uses standard HCL syntax and Terraform workflows
  - Managed Service: Google handles backend state storage, locking, execution
  - Integration: Works with Cloud Build, Cloud Source Repositories, GitHub
  - Security: IAM controls, CMEK support, VPC Service Controls compatible
  - State Management: Automatic remote state with encryption and versioning

Architecture Flow:
Source Repo (GitHub/CSR) → Cloud Build Trigger → Infrastructure Manager → GCP Resources
                              ↓
                    Terraform plan/apply executed in managed environment
```

### PCA Exam Focus Areas

✅ **When to Recommend Infrastructure Manager**:
- Organizations standardizing on Terraform for IaC
- Teams wanting managed state backend without operational overhead
- Google Cloud-focused projects (with potential for future multi-cloud)
- Compliance requirements needing audit trails for infrastructure changes

✅ **Configuration Best Practices**:
```hcl
# Example: Infrastructure Manager compatible Terraform config
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  # Infrastructure Manager manages backend automatically
  # Do NOT configure backend block manually
}

resource "google_compute_instance" "app_server" {
  name         = "app-server-${var.environment}"
  machine_type = var.machine_type
  zone         = var.zone
  
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  
  network_interface {
    network = "default"
    access_config {}
  }
}
```

✅ **Integration with Cloud Build** (Common Exam Scenario):
```yaml
# cloudbuild.yaml for Infrastructure Manager deployment
steps:
  - name: 'gcr.io/cloud-builders/gcloud'
    args: ['infra-manager', 'deployments', 'create', 'my-deployment',
           '--location=us-central1',
           '--source-uri=gs://my-bucket/terraform-config',
           '--service-account=terraform-sa@project.iam.gserviceaccount.com']
```

### Sample Exam Question

> **Scenario**: A financial services company wants to:
> - Use Terraform for infrastructure as code
> - Avoid managing Terraform state backend operations
> - Integrate deployments with existing Cloud Build CI/CD pipelines
> - Meet compliance requirements for change audit trails
>
> **Question**: Which solution BEST meets these requirements?
>
> A) Self-managed Terraform with Cloud Storage backend + manual state locking  
> B) Infrastructure Manager with Cloud Build triggers + Audit Logs enabled  
> C) Cloud Deployment Manager with Python templates + Cloud Logging  
> D) Pulumi with Kubernetes backend + custom audit scripting  
>
> **Answer**: B  
> **Rationale**: Infrastructure Manager provides managed Terraform execution with automatic state management, native Cloud Build integration, and comprehensive audit logging via Cloud Audit Logs. Option A adds operational overhead; C uses deprecated service; D introduces non-standard tooling without managed state benefits [[37]][[39]].

---

## Google Artifact Registry (GAR)

### What is Artifact Registry?

Artifact Registry is Google Cloud's unified repository management service for storing and managing software artifacts including container images, language packages (Maven, npm, PyPI), and other build dependencies [[23]].

### Key Exam Concepts

```yaml
Repository Types:
  - Docker: Container images (OCI format)
  - Maven: Java/JVM dependencies
  - npm: Node.js packages
  - PyPI: Python packages
  - Apt/Yum: Linux package repositories
  - Generic: Arbitrary file storage with metadata

Repository Configuration:
  location: regional (us-central1) or multi-regional (us)
  format: DOCKER | MAVEN | NPM | etc.
  description: "Production container images"
  docker_config:
    immutable_tags: true  # Prevent tag overwrites (security best practice)
  cleanup_policies:  # Automated lifecycle management
    - action: DELETE
      condition:
        older_than: "90d"
        tag_prefixes: ["dev-", "test-"]
```

### Critical PCA Exam Features

✅ **Software Supply Chain Security**:
```
• Vulnerability Scanning: Integration with Artifact Analysis for container CVE detection
• Build Provenance: Store SLSA-compliant provenance metadata with artifacts
• Binary Authorization: Enforce deployment policies based on artifact attestation
• Repository Access Control: IAM roles at project, repository, or package level
• VPC Service Controls: Protect repositories within security perimeters
```

✅ **Repository Design Patterns**:
```
Pattern 1: Environment-Based Repositories
  • dev-registry: Mutable tags, short retention, relaxed policies
  • stage-registry: Immutable tags, vulnerability scanning required
  • prod-registry: Immutable tags, Binary Authorization enforced, audit logging

Pattern 2: Team/Project-Based Repositories  
  • team-a-registry, team-b-registry with separate IAM bindings
  • Virtual repository aggregating multiple repos for unified access

Pattern 3: Upstream Caching (Remote Repositories)
  • Cache public dependencies (docker.io, npmjs.org) to reduce external calls
  • Scan cached dependencies for vulnerabilities before use
```

✅ **Location Strategy for Compliance**:
```
Data Residency Requirements:
• Regional repository (us-central1): Data stays in specific region
• Multi-regional (us): Data replicated within US geography
• Cross-region replication: Not natively supported; use export/import workflows

Exam Tip: Questions about "data never leaves EU" → Choose eu-west1 regional repository
```

### Artifact Registry vs Container Registry Comparison

| Feature | Artifact Registry (Recommended) | Container Registry (Legacy) |
|---------|--------------------------------|----------------------------|
| **Formats Supported** | Docker, Maven, npm, PyPI, Apt, Yum, Generic | Docker containers only |
| **Repository Granularity** | Multiple repos per project, repo-level IAM | Single global/project-level registry |
| **Vulnerability Scanning** | Integrated Artifact Analysis | Limited, via separate setup |
| **Build Provenance** | Native SLSA L3 support | Manual integration required |
| **Location Options** | Regional or multi-regional per repo | Global (gcr.io) or regional (LOCATION.gcr.io) |
| **Pricing Model** | Per-repository + storage + egress | Per-project + storage + egress |
| **Future Support** | Actively developed, recommended [[23]] | Deprecated, migrate to GAR [[29]] |

### Sample Exam Question

> **Scenario**: A healthcare application stores container images with patient data processing logic. Requirements:
> - Images must never leave the europe-west1 region (GDPR compliance)
> - All images must be scanned for vulnerabilities before deployment
> - Production images must have immutable tags to prevent accidental overwrites
> - Development images can be cleaned up after 30 days
>
> **Question**: Which Artifact Registry configuration BEST meets these requirements?
>
> A) Multi-regional (eu) repository with vulnerability scanning enabled  
> B) Regional (europe-west1) repository with immutable_tags=true + cleanup policy for dev-* tags  
> C) Global gcr.io project registry with bucket-level lifecycle rules  
> D) Regional repository with manual vulnerability scanning via third-party tool  
>
> **Answer**: B  
> **Rationale**: Regional europe-west1 repository satisfies GDPR data residency. Immutable tags prevent production image tampering. Cleanup policies automate dev image lifecycle. Artifact Analysis provides integrated vulnerability scanning. Option A violates regional constraint; C uses deprecated GCR with global scope; D adds unnecessary operational complexity [[23]][[29]].

---

## Google Container Registry (GCR) - Legacy

### What Was Container Registry?

Google Container Registry (GCR) was Google Cloud's Docker container image storage service, now superseded by Artifact Registry [[20]].

### Critical Exam Facts

```
⚠️ Status: Deprecated / Legacy
• No new features being added
• Existing repositories continue to work
• Migration to Artifact Registry recommended [[29]]
• New projects should use Artifact Registry [[23]]
```

### When Might GCR Appear on the Exam?

✅ **Legacy Migration Scenarios**:
```
Question Pattern: "Existing system uses gcr.io/project-id/image. What is the migration path?"

Answer Framework:
1. Create equivalent Artifact Registry Docker repository
2. Use gcloud artifacts docker images import or skopeo for image copy
3. Update CI/CD pipelines to push/pull from new registry
4. Update deployment manifests (K8s, Cloud Run) with new image paths
5. Decommission GCR repository after validation
```

✅ **Cost/Architecture Comparison Questions**:
```
Why migrate from GCR to Artifact Registry?
• Fine-grained access control (repo-level vs project-level IAM)
• Integrated vulnerability scanning (Artifact Analysis)
• Support for non-container artifacts (Maven, npm, etc.)
• Better cleanup/lifecycle management policies
• Regional repository options for compliance
```

### Sample Exam Question

> **Scenario**: A company has 200+ container images in gcr.io/legacy-project. They are starting a new microservice project and want to:
> - Use modern security features (vulnerability scanning, immutable tags)
> - Enable repo-level access controls for different teams
> - Prepare for potential multi-format artifact storage (containers + npm packages)
>
> **Question**: What is the MOST appropriate registry strategy?
>
> A) Continue using gcr.io for consistency across projects  
> B) Create new Artifact Registry repositories for the new project; plan phased migration of legacy images  
> C) Use Cloud Storage buckets with custom metadata for artifact management  
> D) Deploy a self-managed Harbor registry on GKE for full control  
>
> **Answer**: B  
> **Rationale**: Artifact Registry provides the required security features, access controls, and multi-format support. Creating new repos for the new project avoids disrupting legacy systems while enabling modern practices. Phased migration balances risk and benefit. Option A misses security improvements; C lacks registry-specific features; D adds operational overhead without managed service benefits [[20]][[23]].

---

## Cloud Build: CI/CD Platform

### What is Cloud Build?

Cloud Build is Google Cloud's serverless CI/CD platform that executes builds, runs tests, and produces artifacts according to user-defined configurations [[30]].

### Key Exam Concepts

```yaml
Build Configuration (cloudbuild.yaml):
steps:
  # Build step examples
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '$_AR_HOSTNAME/$PROJECT_ID/my-app:$COMMIT_SHA', '.']
  
  # Test step
  - name: 'gcr.io/cloud-builders/gcloud'
    args: ['test', 'run', 'tests/unit']
    
  # Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '$_AR_HOSTNAME/$PROJECT_ID/my-app:$COMMIT_SHA']
    
  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'my-service'
      - '--image=$_AR_HOSTNAME/$PROJECT_ID/my-app:$COMMIT_SHA'
      - '--region=us-central1'
      - '--platform=managed'

substitutions:
  _AR_HOSTNAME: 'us-central1-docker.pkg.dev'  # Artifact Registry hostname
  _DEPLOY_REGION: 'us-central1'

options:
  logging: CLOUD_LOGGING_ONLY  # Reduce storage costs
  pool:
    name: 'projects/my-project/locations/us-central1/workerPools/private-pool-1'  # Private pool
```

### Build Triggers: Automation Patterns

✅ **Trigger Types**:
```
• GitHub/Bitbucket/Cloud Source Repositories: Push, pull request, tag events
• Cloud Storage: New file in bucket triggers build
• Schedule: Cron-based periodic builds (e.g., nightly security scans)
• Manual: On-demand builds via console/gcloud/API
```

✅ **Trigger Configuration Best Practices**:
```yaml
# Example: PR validation trigger
name: 'pr-validation'
description: 'Run tests on pull requests'
trigger_template:
  branch_name: ''  # All branches
  invert_regex: false
  repo_name: 'my-repo'
  project_id: 'my-project'
  tag_name: ''
  commit_sha: ''
  dir: ''
included_files: ['src/**', 'cloudbuild.yaml']
excluded_files: ['docs/**', '*.md']
filename: 'cloudbuild-pr.yaml'  # Different config for PR builds

# PR builds: faster, skip deployment steps
# Main branch builds: full pipeline with deployment
```

### Pool Types: Default vs Private (Critical Exam Topic)

| Feature | Default Pool | Private Pool |
|---------|-------------|--------------|
| **Network Access** | Public internet only | VPC, private Google access, on-prem via Cloud Interconnect/VPN |
| **Customization** | Machine type, disk size, timeout | Custom container images, network config, secrets access |
| **Isolation** | Shared Google infrastructure | Dedicated worker pool per project/region |
| **Setup Complexity** | None (automatic) | Requires VPC configuration, firewall rules |
| **Cost** | Included in Cloud Build pricing | Additional Compute Engine costs for worker VMs |
| **Use Case** | Public repos, simple builds | Internal repos, secure builds, hybrid cloud |

✅ **When to Choose Private Pool**:
```
• Building from private GitHub/Bitbucket with VPC-restricted access
• Pulling dependencies from internal Artifact Registry within VPC-SC perimeter
• Deploying to private GKE clusters or internal endpoints
• Compliance requirements prohibiting public internet access during builds
• Need for custom build environment (pre-installed tools, proxies)
```

### Supply Chain Security Features (SLSA Compliance)

✅ **Cloud Build Security Capabilities**:
```
• Ephemeral Build Environments: New VM per build, destroyed after completion
• Build Provenance: SLSA Level 3 attestation generation (verifiable build metadata)
• Binary Authorization Integration: Enforce deployment policies based on build attestations
• Customer-Managed Encryption Keys (CMEK): Encrypt build-time persistent disks
• Secret Manager Integration: Securely inject secrets into build steps
• Audit Logging: Cloud Audit Logs for build creation, execution, and modification
```

✅ **Provenance Generation Example**:
```yaml
# Enable SLSA L3 provenance in build config
options:
  dynamic_substitutions: true
  logging: CLOUD_LOGGING_ONLY
  # Generate build provenance
  build_provenance: true  # Or use --provenance flag in gcloud

# Verify provenance after build
gcloud builds describe BUILD_ID --format="value(provenance)"
```

### Cost Optimization Strategies (Exam Favorite!)

✅ **Reduce Cloud Build Costs**:
```
• Use log aggregation: logging: CLOUD_LOGGING_ONLY to avoid duplicate storage
• Optimize build steps: Cache dependencies, parallelize independent steps
• Choose appropriate machine type: n1-standard-1 for simple builds, highmem for memory-intensive
• Use build timeouts: Prevent runaway builds consuming resources
• Leverage free tier: 120 build-minutes/day free per project
• Clean up old builds: Set retention policy on build history
```

### Sample Exam Question

> **Scenario**: A fintech company builds containerized applications with these requirements:
> - Source code in private GitHub repo accessible only via corporate VPC
> - Build process must pull dependencies from internal Artifact Registry (VPC-SC protected)
> - Generated images must include SLSA Level 3 provenance for compliance
> - Build logs must be retained for 2 years for audit purposes
>
> **Question**: Which Cloud Build configuration BEST meets these requirements?
>
> A) Default pool build trigger with GitHub app integration + Cloud Logging export to BigQuery  
> B) Private pool build trigger with VPC connector + build_provenance: true + log retention policy  
> C) Self-managed Jenkins on GKE with GitHub webhooks + custom provenance scripting  
> D) Default pool with Cloud NAT for GitHub access + manual provenance injection  
>
> **Answer**: B  
> **Rationale**: Private pool enables VPC access to GitHub and internal Artifact Registry. Native build_provenance option provides SLSA L3 compliance without custom development. Cloud Logging with retention policy satisfies audit requirements. Option A cannot access VPC-restricted resources; C adds operational overhead; D lacks reliable VPC access and native provenance support [[30]].

---

## Infrastructure as Code Tool Comparison

### Tool Selection Framework for PCA Exam

```
When asked "Which IaC tool should we use?", evaluate:

1. Multi-cloud requirement?
   • Yes → Terraform, Pulumi, Crossplane
   • No (GCP-only) → Infrastructure Manager, Terraform, or native APIs

2. Team expertise?
   • Terraform experience → Infrastructure Manager or self-managed Terraform
   • Python/TypeScript preference → Pulumi or Deployment Manager (legacy)
   • Kubernetes-native → Config Connector, Crossplane

3. Operational model preference?
   • Managed service (no state backend ops) → Infrastructure Manager
   • Full control + customization → Self-managed Terraform
   • GitOps workflow → Config Connector + Anthos Config Management

4. Compliance/audit requirements?
   • Need change approval workflows → Infrastructure Manager + Cloud Build approvals
   • Need detailed drift detection → Terraform with periodic plan/apply cycles
```

### Tool Comparison Matrix

| Feature | Infrastructure Manager | Terraform (Self-Managed) | Pulumi | Config Connector |
|---------|----------------------|-------------------------|--------|-----------------|
| **Language** | HCL (Terraform) | HCL (Terraform) | TypeScript, Python, Go, C#, YAML | YAML (Kubernetes CRDs) |
| **State Management** | Managed by Google | User-managed (GCS, etc.) | User-managed or Pulumi Cloud | Kubernetes etcd |
| **Multi-Cloud** | Google Cloud focused | ✅ Full multi-cloud | ✅ Full multi-cloud | Google Cloud only |
| **Learning Curve** | Medium (Terraform knowledge) | Medium-High | Medium (programming language) | High (Kubernetes expertise) |
| **Integration with GCP** | Native (Cloud Build, IAM, Audit Logs) | Via provider plugin | Via provider plugin | Native (Kubernetes API) |
| **Drift Detection** | Via periodic deployments | terraform plan | pulumi preview | Kubernetes reconciliation |
| **Best For** | Teams wanting managed Terraform on GCP | Multi-cloud, advanced Terraform users | Developers preferring general-purpose languages | Kubernetes-centric teams, GitOps workflows |

### Migration Strategy Questions (Common Exam Pattern)

✅ **Deployment Manager → Infrastructure Manager**:
```
Steps:
1. Inventory existing DM templates and configurations
2. Convert YAML templates to Terraform HCL (manual or tool-assisted)
3. Set up Infrastructure Manager with appropriate service account permissions
4. Test deployments in non-production environment
5. Implement parallel run: DM and IM managing separate resources
6. Cutover with rollback plan
7. Decommission DM configurations after validation

Exam Tip: Questions may ask about "minimal disruption" → emphasize testing, parallel run, rollback
```

✅ **GCR → Artifact Registry Migration**:
```
Steps:
1. Create Artifact Registry Docker repository in target location
2. Copy images using: 
   gcloud artifacts docker images import SOURCE_IMAGE --source-uri=gcr.io/... --destination-uri=REGION-docker.pkg.dev/...
3. Update CI/CD pipelines to push to new registry
4. Update deployment manifests (K8s, Cloud Run, etc.) with new image paths
5. Validate deployments with new images
6. Update IAM policies for new repository
7. Decommission GCR repository after successful migration

Exam Tip: Emphasize immutable tags during migration to prevent accidental overwrites
```

### Sample Exam Question

> **Scenario**: A company currently uses:
> - Cloud Deployment Manager for infrastructure provisioning
> - gcr.io for container storage
> - Manual deployment scripts for application releases
>
> They want to modernize with:
> - Infrastructure as code with drift detection
> - Unified artifact management with vulnerability scanning
> - Automated CI/CD with approval gates
> - Minimal operational overhead for tooling maintenance
>
> **Question**: Which technology stack BEST meets these goals?
>
> A) Infrastructure Manager + Artifact Registry + Cloud Build with approval steps  
> B) Self-managed Terraform + GCR + Jenkins on GKE  
> C) Pulumi + Cloud Storage buckets + Cloud Deploy  
> D) Config Connector + Artifact Registry + Cloud Build  
>
> **Answer**: A  
> **Rationale**: Infrastructure Manager provides managed Terraform with drift detection and low ops overhead. Artifact Registry offers unified artifact management with integrated vulnerability scanning. Cloud Build provides serverless CI/CD with native approval workflow support. This stack directly addresses all requirements with minimal operational burden. Option B retains legacy GCR and adds Jenkins ops; C uses non-standard artifact storage; D requires Kubernetes expertise for Config Connector [[37]][[23]][[30]].

---

## Scenario-Based Practice Quizzes

### Quiz 1: Greenfield Microservice Platform

> **Scenario**: A startup is building a new microservice platform on Google Cloud:
> - 15+ microservices, each in separate Git repositories
> - Requirements:
>   • Automated builds on every pull request and merge to main
>   • Container images stored with vulnerability scanning before promotion to production
>   • Production deployments require manual approval gate
>   • All infrastructure defined as code with change audit trails
>   • Data residency: All artifacts must remain in us-central1 region
>
> **Question 1**: Which combination of services BEST implements the build and artifact storage requirements?
>
> A) Cloud Build (default pool) + Container Registry (gcr.io) + manual vulnerability scanning  
> B) Cloud Build (private pool) + Artifact Registry (regional us-central1) + Artifact Analysis + Binary Authorization  
> C) Self-managed Jenkins on GKE + Artifact Registry + third-party scanning tool  
> D) Cloud Build (default pool) + Artifact Registry (multi-regional us) + manual approval via email  
>
> **Answer**: B  
> **Rationale**: Private pool enables future VPC integration if needed. Regional Artifact Registry satisfies data residency. Artifact Analysis provides integrated vulnerability scanning. Binary Authorization enforces deployment policies. Cloud Build native approval steps support manual gates. Option A uses deprecated GCR and lacks integrated scanning; C adds unnecessary ops overhead; D violates regional requirement [[23]][[30]].
>
> **Question 2**: How should infrastructure as code be implemented to meet audit and drift detection requirements?
>
> A) Cloud Deployment Manager templates with Cloud Logging for audit  
> B) Infrastructure Manager with Terraform configurations + Cloud Audit Logs  
> C) Manual gcloud scripts stored in Cloud Source Repositories  
> D) Pulumi with custom audit logging to BigQuery  
>
> **Answer**: B  
> **Rationale**: Infrastructure Manager provides managed Terraform execution with native Cloud Audit Logs integration for change tracking. Terraform's plan/apply workflow enables drift detection. Option A uses deprecated service; C lacks IaC benefits and drift detection; D adds custom development without managed service advantages [[37]].
>
> **Question 3**: Which Cloud Build trigger configuration implements the PR and merge workflow?
>
> A) Single trigger on main branch only  
> B) Two triggers: (1) PR events run tests only, (2) main branch pushes run full pipeline with approval  
> C) Scheduled trigger running hourly regardless of code changes  
> D) Manual trigger only for all builds  
>
> **Answer**: B  
> **Rationale**: Separate triggers allow optimized workflows: fast feedback for PRs (tests only), full pipeline with approval for production merges. This balances developer velocity with production safety. Option A misses PR validation; C wastes resources; D eliminates automation benefits [[30]].

### Quiz 2: Regulated Industry Migration

> **Scenario**: A healthcare organization must migrate from legacy systems:
> - Current: Deployment Manager templates + gcr.io + manual deployments
> - Compliance: HIPAA, data residency (US-only), 7-year audit log retention
> - Goals: Modernize while maintaining compliance, minimize downtime
>
> **Question 1**: What is the FIRST step in the migration planning process?
>
> A) Immediately rewrite all Deployment Manager templates in Terraform  
> B) Inventory existing resources, templates, and dependencies; document compliance controls  
> C) Deploy Infrastructure Manager in production and migrate all resources at once  
> D) Decommission gcr.io and force all teams to use Artifact Registry immediately  
>
> **Answer**: B  
> **Rationale**: Migration success depends on understanding the current state: resource inventory, template dependencies, existing compliance controls. This informs risk assessment and phased migration planning. Options A, C, D skip critical discovery phase, risking compliance gaps and service disruption.
>
> **Question 2**: Which Artifact Registry configuration ensures HIPAA-compliant image storage?
>
> A) Multi-regional (us) repository with default IAM settings  
> B) Regional (us-central1) repository with CMEK, VPC-SC perimeter, and Data Access audit logs enabled  
> C) Global gcr.io project registry with bucket-level encryption  
> D) Regional repository with public read access for scanning tools  
>
> **Answer**: B  
> **Rationale**: Regional repository satisfies data residency. CMEK provides encryption key control. VPC-SC prevents data exfiltration. Data Access audit logs capture image access events required for HIPAA audits. Option A lacks fine-grained controls; C uses deprecated service with global scope; D violates least privilege principle [[23]].
>
> **Question 3**: How should build provenance be handled for compliance auditing?
>
> A) Disable provenance to reduce build time and storage costs  
> B) Enable SLSA Level 3 provenance generation and export to BigQuery with 7-year retention  
> C) Store provenance in Cloud Storage with lifecycle policy deleting after 30 days  
> D) Generate provenance only for production builds, skip for development  
>
> **Answer**: B  
> **Rationale**: SLSA L3 provenance provides verifiable build metadata for compliance audits. Exporting to BigQuery with long-term retention satisfies 7-year audit requirements. Option A removes audit capability; C violates retention requirement; D creates inconsistent audit trail [[30]].

### Quiz 3: Cost-Optimized Startup CI/CD

> **Scenario**: A bootstrapped startup with limited budget:
> - 3 microservices, small team (2 engineers)
> - Requirements:
>   • Automated testing on code changes
>   • Container builds with basic security scanning
>   • Simple deployment to Cloud Run
>   • Keep monthly Cloud Build costs under $50
>   • Minimal operational overhead
>
> **Question 1**: Which Cloud Build configuration provides best value?
>
> A) Private pool with custom build images + Artifact Registry + Binary Authorization  
> B) Default pool + Artifact Registry (regional) + basic vulnerability scanning + free tier optimization  
> C) Self-managed Jenkins on smallest GKE node + GCR + manual scanning  
> D) Cloud Build with aggressive log retention (1 day) + no vulnerability scanning  
>
> **Answer**: B  
> **Rationale**: Default pool eliminates private pool compute costs. Artifact Registry regional repository meets basic needs without multi-regional premium. Basic vulnerability scanning via Artifact Analysis provides security without third-party costs. Leveraging 120 free build-minutes/day and optimizing logs controls costs. Option A over-engineers for startup scale; C adds GKE ops overhead; D sacrifices security for cost [[30]][[23]].
>
> **Question 2**: How to implement simple approval workflow without complex tooling?
>
> A) Build approval logic into application code with feature flags  
> B) Use Cloud Build manual approval step + Slack notification for production deployments  
> C) Require all production changes via pull request with team lead review  
> D) Deploy to production automatically; rollback if monitoring alerts trigger  
>
> **Answer**: B  
> **Rationale**: Cloud Build native manual approval provides simple gate with minimal setup. Slack integration enables team notification without custom development. This balances safety with startup velocity. Option A shifts deployment control to app layer (anti-pattern); C relies on process not tooling; D eliminates proactive approval [[30]].
>
> **Question 3**: Which artifact lifecycle strategy minimizes storage costs?
>
> A) Retain all images indefinitely for debugging  
> B) Keep production images forever; delete dev images after 7 days via cleanup policy  
> C) Export all images to Coldline storage after build  
> D) Use only latest tag, overwrite on every build  
>
> **Answer**: B  
> **Rationale**: Cleanup policies in Artifact Registry automate lifecycle management. Retaining production images supports rollback and auditing. Short retention for dev images reduces storage costs. Option A incurs unnecessary costs; C adds transfer/operation costs; D eliminates version history needed for debugging/rollback [[23]].

---

## Quick Reference Tables

### Service Selection Decision Matrix

| Requirement | Recommended Service | Why |
|------------|-------------------|-----|
| New IaC project on GCP | Infrastructure Manager | Managed Terraform, native GCP integration, low ops |
| Multi-cloud IaC | Terraform (self-managed) or Pulumi | Provider ecosystem, language flexibility |
| Container image storage (new project) | Artifact Registry (regional) | Recommended service, integrated security, compliance options |
| Legacy GCR migration | Artifact Registry + import tools | Future-proofing, enhanced features |
| Simple public repo builds | Cloud Build default pool | Zero setup, cost-effective for basic needs |
| VPC-restricted builds | Cloud Build private pool | Network access to internal resources |
| Supply chain compliance (SLSA) | Cloud Build + provenance + Binary Authorization | Native SLSA L3 support, policy enforcement |
| Multi-format artifacts | Artifact Registry | Unified storage for containers, packages, dependencies |
| Kubernetes-native IaC | Config Connector | Manage GCP resources via K8s API, GitOps workflows |

### Cloud Build Cost Optimization Checklist

```
✅ Use default pool unless VPC access required
✅ Set appropriate machine type (start with e2-standard-2)
✅ Configure build timeouts (prevent runaway costs)
✅ Use logging: CLOUD_LOGGING_ONLY to avoid duplicate storage
✅ Cache dependencies between builds (docker layer caching, npm/maven caches)
✅ Parallelize independent build steps
✅ Leverage 120 free build-minutes/day per project
✅ Clean up old build histories (set retention policy)
✅ Use substitutions for reusable values (avoid hardcoding)
✅ Test build configs in staging before production deployment
```

### Artifact Repository Design Patterns

| Pattern | Repository Structure | IAM Strategy | Cleanup Policy |
|---------|---------------------|--------------|---------------|
| **Environment Promotion** | dev-registry, stage-registry, prod-registry | Dev team: dev only; Release managers: stage/prod | Dev: 7 days; Stage: 30 days; Prod: immutable |
| **Team Isolation** | team-a-registry, team-b-registry | Team-specific IAM bindings at repo level | Team-defined retention based on project needs |
| **Upstream Caching** | remote-docker.io (cache), internal-registry (private) | Read-only for remote; standard IAM for internal | Remote: 30 days; Internal: per artifact type |
| **Compliance Isolation** | prod-secure-registry (CMEK, VPC-SC, audit) | Strict IAM + approval workflows | Immutable tags; no automatic deletion |

### Common Exam Keywords → Service Mapping

| Keyword in Question | Likely Service | Why |
|--------------------|---------------|-----|
| "infrastructure as code", "drift detection" | Infrastructure Manager or Terraform | IaC core capabilities |
| "container image storage", "vulnerability scanning" | Artifact Registry + Artifact Analysis | Integrated registry with security features |
| "build on code change", "CI/CD pipeline" | Cloud Build + Build Triggers | Native CI/CD automation |
| "SLSA", "build provenance", "supply chain" | Cloud Build with provenance enabled | Native SLSA L3 support |
| "VPC access during build", "private dependencies" | Cloud Build private pool | Network configuration for restricted access |
| "data residency for artifacts", "regional compliance" | Artifact Registry regional repository | Location control at repository level |
| "Deployment Manager migration" | Infrastructure Manager | Official replacement service |
| "approval gate before production deploy" | Cloud Build manual approval step | Native workflow control |

### Deprecation/Migration Quick Reference

| Legacy Service | End of Support | Recommended Replacement | Migration Tool/Approach |
|---------------|---------------|------------------------|------------------------|
| Cloud Deployment Manager | March 31, 2026 [[web_extractor]] | Infrastructure Manager or Terraform | Manual template conversion; test in staging |
| Container Registry (gcr.io) | No hard date, but deprecated | Artifact Registry | `gcloud artifacts docker images import` |
| Runtime Configurator | Deprecated | Secret Manager or Deployment Manager variables | Migrate secrets to Secret Manager |

---

## Final Exam Day Checklist: Deployment & IaC Section ✅

### 24 Hours Before
- [ ] Memorize Deployment Manager end-of-support date: **March 31, 2026** [[web_extractor]]
- [ ] Review Artifact Registry repository types and location options
- [ ] Practice reading cloudbuild.yaml configurations (identify steps, substitutions, options)
- [ ] Revisit SLSA levels and Cloud Build provenance features

### During Deployment/IaC Questions
- [ ] Check for deprecation traps: Is the question about a legacy service?
- [ ] Verify network requirements: Does the build need VPC access? → Private pool
- [ ] Confirm compliance needs: Data residency? → Regional repository; Audit? → Enable logs
- [ ] Evaluate cost constraints: Default pool vs private pool; cleanup policies for artifacts

### Red Flags to Double-Check
```
⚠️ "Use Deployment Manager for new project" → Wrong (deprecated)
⚠️ "Store containers in gcr.io for new application" → Wrong (use Artifact Registry)
⚠️ "Default pool for builds accessing internal VPC resources" → Wrong (needs private pool)
⚠️ "Skip build provenance to save costs" → Wrong for compliance scenarios
⚠️ "Multi-regional repository when requirement says 'stay in us-east1'" → Wrong (use regional)
⚠️ "Self-managed state backend when question asks for 'minimal operational overhead'" → Wrong (use Infrastructure Manager)
```

> 💡 **Pro Tip**: When stuck between two deployment answers, choose the one that:  
> (1) Uses actively supported services (not deprecated), AND  
> (2) Explicitly addresses security/compliance constraints mentioned in the scenario, AND  
> (3) Balances automation with appropriate approval controls for the risk level

---

## Appendix: Quick Command Reference

### Cloud Build CLI Essentials
```bash
# Submit build from local directory
gcloud builds submit --config=cloudbuild.yaml .

# List builds with filtering
gcloud builds list --filter="status=SUCCESS AND create_time>2026-01-01"

# View build details and provenance
gcloud builds describe BUILD_ID --format="json"

# Create build trigger from GitHub
gcloud builds triggers create github \
  --name="main-branch-build" \
  --repo="my-repo" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml"
```

### Artifact Registry CLI Essentials
```bash
# Create regional Docker repository
gcloud artifacts repositories create prod-containers \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production container images"

# Configure immutable tags
gcloud artifacts repositories update prod-containers \
  --location=us-central1 \
  --docker-immutable-tags

# Import image from GCR to Artifact Registry
gcloud artifacts docker images import \
  gcr.io/my-project/my-app:latest \
  --destination-uri=us-central1-docker.pkg.dev/my-project/prod-containers/my-app:latest

# List images with vulnerability status
gcloud artifacts docker images list us-central1-docker.pkg.dev/my-project/prod-containers \
  --include-vulnerabilities
```

### Infrastructure Manager CLI Essentials
```bash
# Create deployment from Terraform config in Cloud Storage
gcloud infra-manager deployments create my-deployment \
  --location=us-central1 \
  --source-uri=gs://my-bucket/terraform-config \
  --service-account=terraform-sa@project.iam.gserviceaccount.com

# View deployment status and drift
gcloud infra-manager deployments describe my-deployment --location=us-central1

# Update deployment with new configuration
gcloud infra-manager deployments update my-deployment \
  --location=us-central1 \
  --source-uri=gs://my-bucket/terraform-config-v2
```

---

*This study guide is based on official Google Cloud documentation and PCA exam objectives as of April 2026. Always verify with the [official exam guide](https://cloud.google.com/certification/cloud-architect) for updates.*

> 🔄 **Stay Updated**: Google Cloud services evolve rapidly. Subscribe to the [Cloud Deployment Manager deprecation notice](https://cloud.google.com/deployment-manager/docs/deprecation) and [Artifact Registry migration guide](https://cloud.google.com/artifact-registry/docs/transition/transition-from-gcr) for critical timeline updates.

---
*© 2026 PCA Study Materials. For educational purposes only. Not affiliated with Google Cloud.*