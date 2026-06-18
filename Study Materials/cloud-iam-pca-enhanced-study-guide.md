# ☁️ Google Cloud IAM — PCA 2026 Enhanced Study Guide
> **Target:** Google Cloud Professional Cloud Architect (PCA) Exam  
> **Domain:** Security, Compliance & Identity  
> **Enhanced:** April 2026 | Consolidated, enriched with missing topics, new exam traps, and scenario questions

---

## Table of Contents
1. [Cloud IAM — Introduction](#1-cloud-iam--introduction)
2. [Understanding Service Accounts](#2-understanding-service-accounts)
3. [Service Account Key Management](#3-service-account-key-management)
4. [Permissions and Roles](#4-permissions-and-roles)
5. [Role Type Details](#5-role-type-details)
6. [Principle of Least Privilege](#6-principle-of-least-privilege)
7. [Least Privilege via Environment Separation](#7-least-privilege-via-environment-separation)
8. [Groups](#8-groups)
9. [IAM Conditions & ABAC](#9-iam-conditions--abac)
10. [Federated Authentication](#10-federated-authentication)
11. [Privileged Access Manager (PAM)](#11-privileged-access-manager-pam--ga-2024)
12. [IAM Deny Policies](#12-iam-deny-policies--evaluation-order)
13. [OS Login & VM SSH Access Control](#13-os-login--vm-ssh-access-control)
14. [Cloud Audit Logs & IAM Observability](#14-cloud-audit-logs--iam-observability)
15. [BigQuery Data Access Controls](#15-bigquery-data-access-controls)
16. [BeyondCorp & Context-Aware Access](#16-beyondcorp--context-aware-access)
17. [AI/ML Service IAM Patterns](#17-aiml-service-iam-patterns)
18. [Cross-Org & Multi-Tenant IAM](#18-cross-org--multi-tenant-iam)
19. [IAM Limits & Operational Constraints](#19-iam-limits--operational-constraints)
20. [Scenario-Based Exam Questions](#20-scenario-based-exam-questions)
21. [Quick Reference & Decision Frameworks](#21-quick-reference--decision-frameworks)

---

## 1. Cloud IAM — Introduction

### What is Cloud IAM?
Cloud IAM lets you define **who** (identity/principal) can do **what** (role/permission) on **which resource** (resource hierarchy node). It replaces ACL-style access control with a unified, declarative, auditable policy model.

### Core Components

| Component | Description |
|-----------|-------------|
| **Principal (Member)** | Who: Google Account, Service Account, Google Group, Workspace domain, `allUsers`, `allAuthenticatedUsers` |
| **Role** | Named collection of permissions. Roles are assigned to principals — never permissions directly |
| **Permission** | Atomic capability on a resource: `SERVICE.RESOURCE.VERB` (e.g., `storage.objects.get`) |
| **Policy** | JSON document binding principals → roles on a resource. Max 1 policy per resource |
| **Resource** | GCP entity being accessed: Project, Bucket, GKE Cluster, BigQuery Dataset, Secret, etc. |

### IAM Policy Structure (Version 3 — Required for Conditions)

```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:nilanjan@example.com",
        "serviceAccount:app-sa@project.iam.gserviceaccount.com",
        "group:data-readers@scotiabank.com"
      ],
      "condition": {
        "title": "Prod-only access",
        "expression": "resource.matchTag(\"ORG_ID/env\", \"prod\")"
      }
    }
  ],
  "etag": "BwXXXXXX",
  "version": 3
}
```

> 🎯 **PCA Exam Catch:** Policy `"version": 3` is **required** whenever IAM Conditions are present. With version 1 or 2, conditions are **silently ignored** — not rejected. This is a dangerous misconfiguration the exam actively tests.

### IAM Policy Evaluation Order (Complete)

```
Incoming request
    ↓
1. Explicit DENY policy? → DENY (cannot be overridden by any allow)
    ↓ (no deny match)
2. Allow policy binding matches (resource + ancestor union)?
    → YES → Allow
    → NO  → Implicit DENY
    ↓
3. Org Policy constraint violated? → Block (independent of IAM)
```

> 🎯 **PCA Exam Catch:** The evaluation order is **Deny Policies → Allow Policies → Implicit Deny**. A principal with `roles/owner` can still be blocked by an IAM Deny Policy. This is the key feature of Deny Policies — they are absolute.

### Resource Hierarchy & Policy Inheritance

```
Organization  ←── Policies here: all folders, projects, resources inherit
    └── Folder ←── Policies here: all projects and resources within inherit
            └── Project ←── Most common assignment scope
                    └── Resource (GCS Bucket, VM, BigQuery Dataset...)
                              ←── Most granular: only affects this resource
```

**Critical Inheritance Rules:**
- Effective policy = **union** of all policies from resource up to org
- Child nodes **cannot restrict** what a parent has granted (for allow policies)
- **Deny policies** propagate downward and cannot be overridden at child nodes
- Org Policies (separate from IAM) propagate downward and can be restricted further but not relaxed

### When to Use IAM vs. Alternatives

| Scenario | Use IAM | Use Alternative |
|----------|---------|-----------------|
| Control who calls GCP APIs | ✅ Yes | — |
| Row-level data access in BigQuery | Partial | Authorized views + row-level security policies |
| Column-level data restriction in BigQuery | Partial | Column policy tags (Data Catalog) |
| Application-level authorization | ❌ No | Firebase Auth, Identity Platform |
| Network-level access control | ❌ No | VPC Firewall Rules |
| Data exfiltration prevention across API boundary | ❌ No | VPC Service Controls |
| Secret access with audit | ✅ Yes (Secret Manager IAM) | — |
| VM SSH access control | ✅ Yes (OS Login) | — |

---

## 2. Understanding Service Accounts

### What is a Service Account?
A service account is both a **principal** (can be granted roles on resources) and a **resource** (roles can be granted *on* it to control who can use it). It belongs to an application or compute resource, not a human user.

### Types of Service Accounts

| Type | Managed By | Key Use Case | Risk Level |
|------|-----------|--------------|------------|
| **User-managed** | You | Custom apps, GKE workloads, Terraform | You control |
| **Google-managed** | Google (auto-created) | Internal GCP service operations (Cloud Build, Pub/Sub) | Google controls |
| **Default Compute SA** | Google (auto-created per project) | Auto-attached to GCE, App Engine | ⚠️ **High — avoid in prod** |
| **Default Cloud Build SA** | Google (per project) | Cloud Build jobs | ⚠️ Over-privileged by default |

> ⚠️ **Critical Pitfall:** Default Compute Engine SA (`PROJECT_NUMBER-compute@developer.gserviceaccount.com`) is auto-granted `roles/editor` on the project. Always create **custom SAs** with least privilege instead.

### Service Account as a Principal vs. Resource

```
SA as PRINCIPAL (granting a role TO the SA):
    → The SA can perform actions on GCP resources
    gcloud projects add-iam-policy-binding PROJECT_ID \
      --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
      --role="roles/storage.objectViewer"

SA as RESOURCE (granting a role ON the SA):
    → Controls who can USE or IMPERSONATE the SA
    gcloud iam service-accounts add-iam-policy-binding \
      app-sa@PROJECT_ID.iam.gserviceaccount.com \
      --member="user:developer@example.com" \
      --role="roles/iam.serviceAccountUser"
```

### SA Roles — Impersonation vs. Attachment

| Role | Permission Granted | Use Case |
|------|-------------------|---------|
| `roles/iam.serviceAccountUser` | `iam.serviceAccounts.actAs` | Attach SA to a resource (VM, Cloud Run) |
| `roles/iam.serviceAccountTokenCreator` | `iam.serviceAccounts.getAccessToken` | Generate short-lived tokens to impersonate SA |
| `roles/iam.serviceAccountAdmin` | Full management of the SA | Create, delete, modify SAs |
| `roles/iam.workloadIdentityUser` | Used by KSA→GSA Workload Identity binding | GKE pod → GCP SA |

> 🎯 **PCA Exam Catch:** `serviceAccountUser` ≠ `serviceAccountTokenCreator`. The exam distinguishes them:
> - `serviceAccountUser` → attach SA to a resource (set VM's SA at creation time)
> - `serviceAccountTokenCreator` → impersonate/act as SA (generate token, use SA credentials programmatically)
> Granting both to a developer effectively gives them full access to everything the SA can do.

### Workload Identity (GKE — Keyless)

```
Kubernetes SA (k8s: NAMESPACE/KSA_NAME)
        ↕ Workload Identity binding (annotated KSA → GSA)
GCP Service Account (GSA_EMAIL)
        ↓
GCP APIs (GCS, BigQuery, Spanner, etc.)
```

```bash
# Step 1: Annotate Kubernetes SA
kubectl annotate serviceaccount KSA_NAME \
  --namespace=NAMESPACE \
  iam.gke.io/gcp-service-account=GSA_EMAIL

# Step 2: Allow KSA to impersonate GSA
gcloud iam service-accounts add-iam-policy-binding GSA_EMAIL \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"
```

### Cross-Project SA Access

A SA from Project A can be granted roles in Project B:

```bash
# SA in project-a accesses GCS bucket in project-b
gcloud storage buckets add-iam-policy-binding gs://project-b-bucket \
  --member="serviceAccount:app-sa@project-a.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

> 🎯 **PCA Exam Catch:** Cross-project SA access is **legitimate and common** for shared services. However, it means compromise of Project A's SA can impact Project B's resources. Minimize cross-project SA grants and prefer **Shared VPC + scoped subnet IAM** for network-level sharing.

### Architecture Tradeoffs

| Approach | Pros | Cons | When |
|----------|------|------|------|
| Per-workload SA | Least privilege, auditable, blast radius contained | More SA management overhead | Always in production |
| Shared SA | Simpler management | Violates least privilege, wide blast radius | Never in production |
| SA impersonation | No key files, auditable | Requires IAM setup | Developers + CI/CD |
| Workload Identity | Keyless, auto-rotated, no credential management | GKE only | GKE pods in production |
| Workload Identity Federation | Keyless for external systems | Pool + provider setup | GitHub, AWS, Azure |

---

## 3. Service Account Key Management

### Key Types

| Type | Rotation | Use Case | Risk |
|------|----------|---------|------|
| **Google-managed keys** | Automatic (daily) | Workload Identity, attached SA | None — fully managed |
| **User-managed keys (JSON)** | Manual — your responsibility | External systems, legacy on-prem | High — if leaked, permanent until deleted |

### Application Default Credentials (ADC) — Search Order

```
1. GOOGLE_APPLICATION_CREDENTIALS env var → points to key file (avoid in prod)
2. gcloud auth application-default login  → local dev only
3. Attached service account              → GCE, GKE, Cloud Run, Cloud Functions
4. Error

Production Best Practice: Always rely on Step 3 (attached SA).
Local Dev Best Practice: Use Step 2 + SA impersonation (not key files).
```

### Org Policies for Key Governance

```bash
# Block SA key creation org-wide (most important)
constraints/iam.disableServiceAccountKeyCreation

# Block SA key upload (prevent importing external keys)
constraints/iam.disableServiceAccountKeyUpload

# Block SA creation entirely in specific projects (high security)
constraints/iam.disableServiceAccountCreation

# Set maximum key age (enforces rotation)
constraints/iam.serviceAccountKeyExpiryHours
    → Set to 2160 (90 days) for PCI-DSS compliance
```

> 🎯 **PCA Exam Catch (NEW for 2026):** `constraints/iam.disableServiceAccountCreation` is separate from `constraints/iam.disableServiceAccountKeyCreation`. The first blocks creating SAs entirely; the second only blocks key downloads. For maximum lockdown in production, apply **both** — along with `disableServiceAccountKeyUpload`. These three org policies together eliminate key-based SA risks.

### Key Exposure Detection & Response

```
Detection:
    → Cloud Audit Logs: iam.googleapis.com/serviceAccounts.keys.create (Data Access)
    → SCC Finding: "Service Account Key Created"
    → Cloud Asset Inventory: enumerate all active key counts per SA
    → Policy Intelligence: flag SAs with > 1 active key

Response Playbook (SA key compromised):
    1. Immediately delete the compromised key:
       gcloud iam service-accounts keys delete KEY_ID --iam-account=SA_EMAIL
    2. Audit Cloud Audit Logs for unauthorized usage of the key
    3. Rotate all credentials that SA had access to
    4. Review and tighten SA permissions if over-privileged
    5. Apply org policy to prevent future key creation
```

### Architecture Decision: Keys vs. Keyless

```
Workload Location              → Authentication Method
─────────────────────────────────────────────────────
GKE pod                        → Workload Identity (keyless ✅)
GCE / Cloud Run / Functions    → Attached SA (keyless ✅)
Cloud Build                    → Attached SA or WIF (keyless ✅)
GitHub Actions                 → Workload Identity Federation (keyless ✅)
AWS Lambda                     → Workload Identity Federation (keyless ✅)
Developer local machine        → gcloud ADC + SA impersonation (keyless ✅)
Legacy on-prem system          → SA key in Secret Manager/Vault (key ⚠️)
Third-party SaaS               → SA key (unavoidable) → rotate 90 days (key ⚠️)
```

---

## 4. Permissions and Roles

### Permission Format
```
SERVICE.RESOURCE.VERB

Examples:
storage.objects.get          → Read a GCS object
container.clusters.create    → Create a GKE cluster
bigquery.tables.getData      → Query a BigQuery table
iam.serviceAccounts.actAs    → Act as (use) a service account
resourcemanager.projects.setIamPolicy → Modify project IAM policies
```

### IAM Policy Evaluation — Full Flow

```
Request arrives
    ↓
[1] IAM Deny Policies checked (org → folder → project → resource)
    Any deny match? → HARD DENY (no bypass)
    ↓ (no deny)
[2] Allow policies collected (resource + all ancestors)
    Effective = UNION of all allow bindings
    Does effective policy include permission?
    → YES → Allow
    → NO  → Implicit Deny
    ↓ (allow granted)
[3] Org Policy constraints checked
    Constraint violated? → Block regardless of IAM
```

### Deny Policies (IAM v2)

```bash
# Deny policy JSON structure
{
  "rules": [
    {
      "denyRule": {
        "deniedPrincipals": ["principalSet://goog/public:all"],
        "deniedPermissions": ["storage.objects.get"],
        "exceptionPrincipals": [
          "principal://iam.googleapis.com/projects/PROJECT/serviceAccounts/trusted-sa@..."
        ]
      }
    }
  ]
}

# Create deny policy at org level
gcloud iam policies create block-public-storage \
  --attachment-point=cloudresourcemanager.googleapis.com/organizations/ORG_ID \
  --policy-file=deny-policy.json
```

**Deny Policy Exemptions** — A critical feature often missed:
- You can add `exceptionPrincipals` to exclude specific SAs or users from a deny rule
- Example: deny public access to all except a specific CDN SA

> 🎯 **PCA Exam Catch:** Deny policies support **exception principals**. If the question says "block all public access to GCS except for the CDN service account," the answer uses a Deny Policy with an `exceptionPrincipals` carve-out — not a complex allow policy construction.

### Exam-Critical Permission Groups

| Category | Key Permissions | Role That Includes It |
|----------|----------------|----------------------|
| Modify project IAM | `resourcemanager.projects.setIamPolicy` | `roles/resourcemanager.projectIamAdmin`, `roles/owner` |
| Use a service account | `iam.serviceAccounts.actAs` | `roles/iam.serviceAccountUser` |
| Generate SA token | `iam.serviceAccounts.getAccessToken` | `roles/iam.serviceAccountTokenCreator` |
| Create deny policies | `iam.denypolicies.create` | `roles/iam.denyAdmin` |
| Set org policies | `orgpolicy.policy.set` | `roles/orgpolicy.policyAdmin` |
| Create custom roles | `iam.roles.create` | `roles/iam.roleAdmin` |

---

## 5. Role Type Details

### Basic (Primitive) Roles — Avoid in Production

| Role | What It Includes | Exam Rule |
|------|-----------------|-----------|
| `roles/viewer` | Read-only, all services | Acceptable only for audit access |
| `roles/editor` | Read+write, no IAM/billing | ❌ Never in production |
| `roles/owner` | Everything including IAM | ❌ Only for project bootstrap; remove after |

> ❌ **Absolute Exam Rule:** Basic roles are **always a wrong answer** in security-focused PCA questions. The exam uses them as distractors. If you see `roles/editor` or `roles/owner` in an option for a production workload, eliminate it immediately.

### Predefined Roles — Must-Know for Exam

**GKE / Compute:**
| Role | Key Capabilities |
|------|-----------------|
| `roles/container.admin` | Full GKE control including cluster lifecycle |
| `roles/container.developer` | Deploy workloads, no cluster management |
| `roles/container.clusterViewer` | Read cluster config only |
| `roles/container.nodeServiceAccount` | Required for GKE node pool SA |
| `roles/compute.instanceAdmin` | Full VM management |
| `roles/compute.osLogin` | SSH to VMs via OS Login |
| `roles/compute.osAdminLogin` | SSH to VMs with sudo via OS Login |

**Storage / Data:**
| Role | Key Capabilities |
|------|-----------------|
| `roles/storage.objectViewer` | Read GCS objects only |
| `roles/storage.objectCreator` | Create (not read/delete) objects |
| `roles/storage.objectAdmin` | Full object CRUD, no bucket-level |
| `roles/storage.admin` | Full control including bucket creation/deletion |
| `roles/bigquery.dataViewer` | Read/query tables, not create |
| `roles/bigquery.dataEditor` | CRUD tables and data |
| `roles/bigquery.jobUser` | Run jobs — needed WITH dataViewer to query |
| `roles/bigquery.user` | `jobUser` + `dataViewer` on datasets user creates |

**IAM / Security:**
| Role | Key Capabilities |
|------|-----------------|
| `roles/iam.serviceAccountUser` | Attach SA to resources |
| `roles/iam.serviceAccountTokenCreator` | Impersonate SA (generate tokens) |
| `roles/iam.securityReviewer` | View IAM policies, no modification |
| `roles/iam.securityAdmin` | Manage IAM policies across org |
| `roles/iam.denyAdmin` | Create/manage Deny Policies |
| `roles/iam.roleAdmin` | Create/manage custom roles |
| `roles/secretmanager.secretAccessor` | Read secret versions |
| `roles/secretmanager.secretVersionManager` | Create/destroy secret versions |

> 🎯 **PCA Exam Catch (BigQuery):** `roles/bigquery.dataViewer` alone is **NOT sufficient** to run queries. You also need `roles/bigquery.jobUser` to submit query jobs. The exam frequently tests this two-role requirement. Alternatively, `roles/bigquery.user` bundles both.

### Custom Roles

```bash
# custom-role.yaml
title: "GCS Report Reader"
description: "Read-only access to reports/ prefix only — used by report SA"
stage: GA
includedPermissions:
  - storage.objects.get
  - storage.objects.list
  - storage.buckets.list

# Create at project scope
gcloud iam roles create gcsReportReader \
  --project=PROJECT_ID \
  --file=custom-role.yaml

# Create at org scope (available to all projects)
gcloud iam roles create gcsReportReader \
  --organization=ORG_ID \
  --file=custom-role.yaml
```

**Custom Role Constraints:**

| Constraint | Detail |
|-----------|--------|
| Scope | Project-level OR Org-level (not folder-level) |
| Max permissions | 3,000 per custom role |
| Stages | `ALPHA` → `BETA` → `GA` → `DISABLED` |
| Inheritance | Custom roles are NOT inherited — must be assigned explicitly at each resource |
| Maintenance | Google evolving APIs may deprecate included permissions — you must update |

> 🎯 **PCA Exam Catch:** Custom roles are **not available at folder level** — only project or organization. If you need a custom role available across multiple projects in a folder, create it at the **organization level**.

---

## 6. Principle of Least Privilege

### PoLP Implementation Framework

```
1. IDENTIFY: What exact actions does this principal perform?
2. ENUMERATE: What minimal permissions cover exactly those actions?
3. SELECT: Find predefined role → if too broad, create custom role
4. SCOPE: Assign at the lowest resource level (resource > project > folder > org)
5. BOUND: Add IAM Conditions for time or resource constraints if applicable
6. AUDIT: Schedule IAM Recommender review quarterly; act on suggestions
```

### Policy Intelligence Tools — Deeper Coverage

| Tool | What It Does | When to Use |
|------|-------------|------------|
| **IAM Recommender** | ML-based excess permission detection; suggests role downgrade or removal | Quarterly review; automate via Pub/Sub + Cloud Function |
| **Policy Analyzer** | Answers "who has access to resource X?" across the org | Before decommissioning a resource; security audits |
| **Policy Troubleshooter** | Explains exactly why principal Y can/cannot do action Z | Debugging access denials in production |
| **Activity Analyzer** | Shows last-used timestamp for each role binding | Identifying dormant/stale permissions |

```bash
# IAM Recommender — automate remediation
gcloud recommender recommendations list \
  --recommender=google.iam.policy.Recommender \
  --location=global \
  --project=PROJECT_ID \
  --format=json | \
  jq '.[] | select(.stateInfo.state == "ACTIVE")' | \
  # Pipe to Cloud Function for auto-apply after review

# Policy Analyzer — who has storage.admin on the org?
gcloud asset search-all-iam-policies \
  --scope=organizations/ORG_ID \
  --query="roles:roles/storage.admin" \
  --format="table(resource,policy.bindings)"

# Activity Analyzer — find bindings unused for 90+ days
gcloud policy-intelligence query-activity \
  --project=PROJECT_ID \
  --activity-type=serviceAccountKeyLastAuthentication
```

### Privileged Access Management (PAM) — GA 2024

PAM provides **Just-in-Time (JIT)** privileged access with approval workflows and automatic expiry:

```
PAM JIT Access Flow:
    SRE requests elevated access (e.g., roles/container.admin for 2 hours)
        → PAM creates grant request
        → Manager/approver receives notification (email, Slack via webhook)
        → Approver approves in PAM console
        → Role binding created with IAM Condition: expires in 2 hours
        → SRE performs task
        → Binding auto-expires → access removed
        → Full audit trail in Cloud Audit Logs

PAM vs. IAM Conditions:
    IAM Conditions alone: Cannot enforce approval workflow
    PAM: Includes approval, auto-expiry, AND audit trail in one service
```

> 🎯 **PCA Exam Catch:** For any scenario involving "break-glass," "emergency access," "time-bounded admin access with approval," or "SOC2 compliance for privileged operations" → **PAM** is the correct answer, NOT IAM Conditions alone. IAM Conditions can set time bounds but have no built-in approval workflow.

### PoLP Anti-Patterns — Exam Recognition

| Anti-Pattern in Question | What to Choose |
|--------------------------|---------------|
| `roles/editor` on a service account | Narrow predefined or custom role |
| `roles/owner` for a developer "for simplicity" | `roles/container.developer` + specific scoped roles |
| SA with `roles/storage.admin` to read one bucket | `roles/storage.objectViewer` on that specific bucket |
| Shared SA across 10 microservices | Dedicated SA per microservice |
| Permanent admin role for break-glass scenarios | PAM JIT access with approval + auto-expiry |
| Org-level role for project-scoped need | Scope to project or resource level |
| `roles/bigquery.admin` for a report runner | `roles/bigquery.dataViewer` + `roles/bigquery.jobUser` |

---

## 7. Least Privilege via Environment Separation

### GCP Recommended Pattern: Separate Projects per Environment

```
Organization: scotiabank.com
    ├── Folder: Production
    │       ├── Project: identity-platform-prod
    │       ├── Project: payments-prod
    │       └── Project: shared-vpc-prod (host project for Shared VPC)
    ├── Folder: Staging
    │       ├── Project: identity-platform-staging
    │       └── Project: payments-staging
    ├── Folder: Development
    │       ├── Project: identity-platform-dev
    │       └── Project: payments-dev
    └── Folder: Security & Governance
            ├── Project: audit-logging-central
            ├── Project: scc-management
            └── Project: cicd-platform
```

### Folder-Level Org Policy Differentiation

| Policy | Dev Folder | Prod Folder | Why |
|--------|-----------|-------------|-----|
| `constraints/compute.requireShieldedVm` | Not enforced | Enforced | Compliance required in prod |
| `constraints/gcp.resourceLocations` | Multi-region OK | `northamerica-northeast1` only | Data residency |
| `constraints/iam.disableServiceAccountKeyCreation` | Allowed | Enforced | Dev may need legacy access |
| `constraints/iam.disableServiceAccountKeyUpload` | Allowed | Enforced | Prevent external key import in prod |
| `constraints/compute.vmExternalIpAccess` | Allowed | Denied | Prod VMs must use Private Google Access |
| `constraints/compute.skipDefaultNetworkCreation` | Enforced | Enforced | Never use default VPC |

### CI/CD with Environment Separation — Separate SA per Stage

```
Developer commits → PR created
    ↓
Cloud Build (CI project): ci-build-sa@cicd-project.iam.gserviceaccount.com
    → roles/container.developer on DEV project only
    → Runs tests, builds container image
    ↓
PR approved + merged to main
    ↓
Cloud Deploy / Cloud Build (CD): cd-staging-sa@cicd-project.iam.gserviceaccount.com
    → roles/container.developer on STAGING project only
    → Deploys to staging, runs integration tests
    ↓
Manual approval gate (release manager)
    ↓
Prod deploy: prod-deploy-sa@cicd-project.iam.gserviceaccount.com
    → roles/container.developer on PROD project only
    → Verified: SA cannot access dev/staging resources
    → All actions logged in prod project's Audit Logs
```

**Key Rule:** Each SA is scoped to exactly one environment. No SA crosses environment boundaries.

### VPC Service Controls — Complement to Project Isolation

```
Even with IAM correctly configured:
    → An authenticated user could call BigQuery API from outside the org
    → An IAM-privileged user could exfiltrate data to a personal GCS bucket

VPC-SC adds an API-level perimeter:
    → Blocks all BigQuery/GCS/Spanner API calls from outside the perimeter
    → Even requests WITH valid credentials are rejected if outside the perimeter
    → Prevents data exfiltration by blocking copy-to-external-bucket operations

IAM + VPC-SC = defense in depth:
    IAM controls who → VPC-SC controls from where
```

---

## 8. Groups

### Why Groups Over Individuals

| Benefit | Without Groups | With Groups |
|---------|---------------|------------|
| Onboarding | Add user to N IAM policies | Add user to 1 group |
| Offboarding | Remove from N IAM policies | Remove from 1 group |
| Auditability | Policy shows individual names | Policy shows intent (`prod-sre-oncall`) |
| Policy churn | High | Zero for personnel changes |

### Group Types in GCP Context

| Type | Managed In | Key Feature | IAM Use |
|------|-----------|-------------|---------|
| **Google Group** | Google Groups / Workspace | Standard collection | Standard IAM bindings |
| **Cloud Identity Group** | Cloud Identity | Dynamic membership rules | Automated membership |
| **Security Group** | Cloud Identity | Enforced in VPC-SC perimeters | VPC-SC access levels + IAM |
| **Posix Group** | Cloud Identity | Linux GID/UID mapping | OS Login SSH access control |

### Recommended Group Naming

```
Pattern: <environment>-<service>-<role>@corp.com

prod-gke-admins@scotiabank.com
prod-data-engineers@scotiabank.com
staging-all-developers@scotiabank.com
platform-sre-oncall@scotiabank.com
security-auditors@scotiabank.com
cicd-deployment-approvers@scotiabank.com
```

### allUsers vs. allAuthenticatedUsers — Dangerous Member Types

| Member | Meaning | Safe Use | Exam Answer |
|--------|---------|---------|------------|
| `allUsers` | Everyone, no auth | Static public website assets | Signed URLs instead |
| `allAuthenticatedUsers` | Any Google account globally | Almost never | Signed URLs or explicit group |

> 🎯 **PCA Exam Catch:** "Share a GCS object temporarily with an external partner" → **Signed URL** (not `allUsers`). Signed URLs are time-bound, traceable (include SA identity), and require no IAM changes. `allUsers` is untraceable, permanent until removed, and fails compliance audits.

### Org Policy for Domain Restriction

```bash
# Restrict IAM bindings to your domain only
constraints/iam.allowedPolicyMemberDomains
    → Value: your Cloud Identity customer ID (C01234567)
    → Effect: Blocks adding @gmail.com, @hotmail.com, external accounts to IAM
    → Works at Org/Folder/Project level

# This prevents:
    gcloud projects add-iam-policy-binding PROJECT_ID \
      --member="user:external@gmail.com" \
      --role="roles/viewer"
    → ERROR: Policy violates constraint iam.allowedPolicyMemberDomains
```

---

## 9. IAM Conditions & ABAC

### What are IAM Conditions?
Conditions implement **Attribute-Based Access Control (ABAC)** — a role binding only takes effect when a CEL expression evaluates to `true`. Requires **policy version 3**.

### Condition Attributes

| Category | Attributes | CEL Example |
|----------|-----------|------------|
| **Time** | `request.time` | `request.time < timestamp("2026-12-31T00:00:00Z")` |
| **Resource name** | `resource.name` | `resource.name.startsWith("projects/_/buckets/prod-bucket/")` |
| **Resource type** | `resource.type` | `resource.type == "storage.googleapis.com/Bucket"` |
| **Resource service** | `resource.service` | `resource.service == "storage.googleapis.com"` |
| **Tags** | `resource.matchTag()` | `resource.matchTag("ORG_ID/env", "prod")` |
| **Access Level** | `request.auth.access_levels` | BeyondCorp integration (see Section 16) |

### IAM Conditions Cannot Apply To

```
❌ Basic roles (owner, editor, viewer) — conditions not supported
❌ Google-managed service accounts — conditions not supported
❌ allUsers / allAuthenticatedUsers — conditions not supported
```

> 🎯 **PCA Exam Catch:** You **cannot** add a time-bound condition to `roles/owner`. The exam may present this as an option for "make the owner access temporary" — it is invalid. Use **PAM** for time-bound privileged access instead.

### CEL Expression Examples

```cel
# 1. Business hours only (weekdays, 9 AM–5 PM UTC)
request.time.getHours("UTC") >= 9 &&
request.time.getHours("UTC") < 17 &&
request.time.getDayOfWeek("UTC") >= 1 &&
request.time.getDayOfWeek("UTC") <= 5

# 2. Specific GCS bucket prefix only
resource.name.startsWith(
  "projects/_/buckets/reports-bucket/objects/2026/")

# 3. Temporary access expiry
request.time < timestamp("2026-06-30T23:59:59Z")

# 4. Tag-based: prod environment only
resource.matchTag("123456789/env", "prod")

# 5. Specific resource type (GCS buckets only, not objects)
resource.type == "storage.googleapis.com/Bucket"

# 6. Combined: access prod-tagged GCS objects until end of quarter
resource.matchTag("123456789/env", "prod") &&
resource.type == "storage.googleapis.com/Object" &&
request.time < timestamp("2026-06-30T23:59:59Z")
```

### Resource Tags vs. Labels — Critical Distinction

| | Resource Tags | Resource Labels |
|--|--------------|----------------|
| **Used in IAM Conditions** | ✅ Yes (`resource.matchTag()`) | ❌ No |
| **Used in Billing** | ❌ No | ✅ Yes |
| **Scope** | Org-level (hierarchical) | Per-resource (flat) |
| **Inheritance** | Flows down hierarchy | Not inherited |
| **Security-relevant** | ✅ Critical for ABAC | ❌ Not security control |

> 🎯 **PCA Exam Catch:** **Labels cannot be used in IAM Conditions.** Only **Resource Tags** (created via Resource Manager Tags API) can be referenced in `resource.matchTag()`. If a question asks "grant access to all GCS buckets labeled `env=prod`", the answer is NOT using labels in conditions — you must use Tags instead.

---

## 10. Federated Authentication

### Two Federation Models

| Model | For | GCP Service | Protocol |
|-------|-----|------------|---------|
| **Workforce Identity Federation** | Human users from corporate IdP | Workforce Identity Pools | OIDC or SAML 2.0 |
| **Workload Identity Federation** | Non-GCP compute workloads | Workload Identity Pools | OIDC, SAML, or AWS SigV4 |

### Workforce Identity Federation — Human Users

```
Okta / Azure AD / ADFS / PingFederate
    ↓ User authenticates to IdP
    ↓ OIDC ID token (JWT) or SAML assertion issued
    ↓ Token sent to Google STS (Security Token Service)
    ↓ STS validates token against pool provider config
    ↓ Short-lived Google token issued
    ↓ User accesses GCP Console or CLI or APIs

Principal format in IAM:
    principal://iam.googleapis.com/locations/global/workforcePools/POOL/subject/USER_ID
```

```bash
# Create workforce pool at org level
gcloud iam workforce-pools create corp-employees \
  --organization=ORG_ID \
  --location=global \
  --session-duration=28800s  # Max 8-hour sessions

# Create OIDC provider (Okta example)
gcloud iam workforce-pools providers create-oidc okta-provider \
  --workforce-pool=corp-employees \
  --location=global \
  --issuer-uri="https://scotiabank.okta.com" \
  --client-id="CLIENT_ID" \
  --attribute-mapping="google.subject=assertion.sub,\
    google.email=assertion.email,\
    google.groups=assertion.groups,\
    attribute.department=assertion.department"
```

**Session Duration Control** — NEW detail for 2026:
- Default session: 1 hour
- Maximum session: 12 hours
- Set via `--session-duration` on the pool
- Shorter sessions = better security posture for compliance

### Workload Identity Federation — External Workloads

```
GitHub Actions / AWS EC2 / Azure VM / On-prem
    ↓ Native OIDC token issued (e.g., GitHub OIDC token for repo/branch)
    ↓ Token sent to Google STS with pool provider info
    ↓ STS validates: issuer, audience, attribute conditions
    ↓ Short-lived Google credential issued
    ↓ [Option A] Direct IAM binding on pool identity
    ↓ [Option B] Impersonate a GCP Service Account → get SA access token

Direct binding (Option A — preferred when possible):
    Grant IAM role directly to the federated identity
    No intermediate GCP SA needed

SA impersonation (Option B — when cross-project or SA features needed):
    Grant roles/iam.workloadIdentityUser on a GCP SA to the pool identity
    Federated identity → impersonates SA → SA's permissions apply
```

```bash
# GitHub Actions — bind to specific repo + branch
gcloud iam service-accounts add-iam-policy-binding \
  deploy-sa@project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/\
    locations/global/workloadIdentityPools/POOL/\
    attribute.repository/my-org/my-repo"

# Further restrict to specific branch
--member="principalSet://...attribute.ref/refs/heads/main"
```

### Attribute Mapping — Deep Dive

```
Attribute mapping translates IdP token claims to Google attributes:

GitHub OIDC claims:
    sub:        "repo:my-org/my-repo:ref:refs/heads/main"
    repository: "my-org/my-repo"
    ref:        "refs/heads/main"

Attribute mapping (pool provider config):
    google.subject   = assertion.sub
    attribute.repository = assertion.repository
    attribute.ref        = assertion.ref

IAM binding uses custom attributes:
    principalSet://...attribute.repository/my-org/my-repo
    → Only repos named "my-org/my-repo" can use this binding
    → Attribute conditions enforce repo + branch restrictions
```

### SAML 2.0 vs. OIDC for Federation

| | SAML 2.0 | OIDC |
|--|---------|------|
| Format | XML assertions | JWT tokens |
| Common IdPs | ADFS, Okta (SAML mode), Ping Identity | Okta (OIDC mode), Azure AD, GitHub, AWS |
| GCP Support | **Workforce Identity only** | Both Workforce + Workload |
| Complexity | Higher — XML parsing, metadata | Lower — standard JWT |
| For CI/CD | ❌ Not suitable | ✅ Standard for GitHub/AWS/Azure |

> 🎯 **PCA Exam Catch:** **SAML is only supported for Workforce Identity Federation** (human users), NOT for Workload Identity Federation. If a question describes a machine workload needing federation and mentions SAML — the answer must be OIDC or the AWS SigV4 path, not SAML.

---

## 11. Privileged Access Manager (PAM) — GA 2024

### What is PAM?
PAM is Google Cloud's managed service for **Just-in-Time (JIT)** privileged access. It eliminates long-lived admin role bindings by replacing them with time-bounded, approval-gated, auditable access grants.

### PAM Core Concepts

```yaml
Entitlement:
  Definition: Reusable template defining what role can be requested and by whom
  Contents:
    - Eligible requestors (group or individual)
    - Role(s) to be granted
    - Max duration (e.g., 2 hours)
    - Required approvers (1 or more)
    - Notification channels

Grant Request:
  Process: Requestor selects entitlement → provides justification → submits
  Approval: Approver reviews context + justification → approve/deny
  Activation: On approval, IAM Condition with expiry auto-created
  Expiry: Binding auto-deleted after duration → access removed

Audit:
  All requests, approvals, and activations → Cloud Audit Logs
  Full justification text preserved in audit record
  Integrates with SIEM systems via log sinks
```

### PAM vs. IAM Conditions — Know the Difference

| Feature | IAM Conditions (time-bound) | PAM |
|---------|---------------------------|-----|
| Approval workflow | ❌ No | ✅ Yes |
| Auto-expiry | ✅ Yes (manual setup) | ✅ Yes (built-in) |
| Audit trail | Partial (IAM change logged) | ✅ Full (request, justification, approval) |
| Self-service request | ❌ No | ✅ Yes |
| Break-glass support | Limited | ✅ Designed for it |
| Compliance (SOC2, HIPAA) | Partial | ✅ Full audit trail meets requirements |

```bash
# Create PAM entitlement for SRE break-glass access
gcloud pam entitlements create sre-breakglass \
  --location=global \
  --project=PROJECT_ID \
  --eligible-users="group:prod-sre@scotiabank.com" \
  --roles="roles/container.admin,roles/compute.instanceAdmin" \
  --max-request-duration=7200s \
  --require-approvers="group:sre-managers@scotiabank.com" \
  --approval-count=1

# SRE requests access
gcloud pam grants create \
  --entitlement=sre-breakglass \
  --location=global \
  --project=PROJECT_ID \
  --requested-duration=3600s \
  --justification="P0 incident: GKE cluster OOMKilling, need to scale node pool"
```

---

## 12. IAM Deny Policies — Evaluation Order

### How Deny Policies Work

```
Evaluation sequence per request:
[1] Org-level deny policies
[2] Folder-level deny policies
[3] Project-level deny policies
[4] Resource-level deny policies
    → Any match = HARD DENY (overrides all allows)
[5] Allow policies (from resource up to org)
    → Match = Allow
    → No match = Implicit deny
```

### Deny Policy with Exception Principals

```json
{
  "rules": [
    {
      "denyRule": {
        "deniedPrincipals": ["principalSet://goog/public:all"],
        "deniedPermissions": ["storage.objects.get", "storage.objects.list"],
        "exceptionPrincipals": [
          "serviceAccount:cdn-sa@project.iam.gserviceaccount.com",
          "serviceAccount:backup-sa@project.iam.gserviceaccount.com"
        ]
      }
    }
  ]
}
```

### Deny Policies vs. Org Policies — Critical Distinction

| | IAM Deny Policy | Org Policy Constraint |
|--|-----------------|----------------------|
| **Controls** | Who can do what (principal + permission) | What configurations are allowed (resource properties) |
| **Evaluated** | Per API call (real-time) | At resource creation/modification |
| **Bypassed by** | Exception principals only | `restoreDefault` on child nodes (if allowed) |
| **Example** | Deny `storage.objects.delete` to all except backup-sa | Deny creation of GCS buckets outside Canada |
| **Scope** | Org, Folder, Project, Resource | Org, Folder, Project |

> 🎯 **PCA Exam Catch:** Org Policy prevents configurations from being **created**. IAM Deny Policy prevents **API calls** from being **executed**. They are complementary:
> - "Prevent any team from creating external IPs on VMs" → **Org Policy** (`compute.vmExternalIpAccess`)
> - "Prevent any user from making GCS objects public even with storage.admin" → **IAM Deny Policy**

---

## 13. OS Login & VM SSH Access Control

### What is OS Login?
OS Login is Google Cloud's **managed SSH key management** service for Compute Engine VMs. It ties SSH access to a principal's **GCP IAM identity**, replacing manually managed SSH keys in project metadata.

### OS Login vs. Project Metadata SSH Keys

| | OS Login (Recommended) | Project Metadata SSH Keys (Legacy) |
|--|------------------------|-----------------------------------|
| **Key management** | Automatic — tied to user's GCP account | Manual — added to project/instance metadata |
| **Access control** | IAM roles | Anyone with metadata write access |
| **Multi-factor auth** | ✅ Supports 2FA via BeyondCorp | ❌ Not supported |
| **Audit trail** | ✅ Cloud Audit Logs | ❌ No audit per key |
| **Revocation** | Instant — revoke IAM role | Manual — remove key from metadata |
| **Org-wide enforcement** | ✅ `constraints/compute.requireOsLogin` | ❌ No enforcement mechanism |

### OS Login IAM Roles

| Role | Access Level | Use Case |
|------|-------------|---------|
| `roles/compute.osLogin` | SSH as regular user (no sudo) | Application developers |
| `roles/compute.osAdminLogin` | SSH with sudo / root access | SRE, system administrators |
| `roles/iam.serviceAccountUser` | Required to use SA-attached instances | When SSH-ing into a VM with an SA |

```bash
# Enable OS Login at org level (apply to all VMs)
gcloud resource-manager org-policies set-policy \
  --organization=ORG_ID \
  constraints/compute.requireOsLogin.yaml

# Grant SSH access (no sudo) to a developer group
gcloud compute instances add-iam-policy-binding VM_NAME \
  --zone=northamerica-northeast1-a \
  --member="group:dev-team@scotiabank.com" \
  --role="roles/compute.osLogin"

# Grant admin SSH access to SRE (sudo)
gcloud compute instances add-iam-policy-binding VM_NAME \
  --zone=northamerica-northeast1-a \
  --member="group:prod-sre@scotiabank.com" \
  --role="roles/compute.osAdminLogin"
```

### OS Login + POSIX Groups

Cloud Identity POSIX groups allow mapping GCP groups to Linux groups for fine-grained file permission control:

```
GCP Group: prod-sre@scotiabank.com → POSIX GID: 1001 → Linux group: sre-team
GCP Group: dev-team@scotiabank.com → POSIX GID: 1002 → Linux group: developers
```

> 🎯 **PCA Exam Catch:** OS Login is enforced via org policy `constraints/compute.requireOsLogin`. Without this org policy, teams can still use project metadata SSH keys (bypassing IAM control). The exam tests whether you know that OS Login alone, without the org policy, is not a complete control.

---

## 14. Cloud Audit Logs & IAM Observability

### Audit Log Types — Must Know

| Log Type | What It Captures | Default Status | IAM-Relevant Events |
|----------|-----------------|---------------|---------------------|
| **Admin Activity** | Resource configuration changes | Always on (cannot disable) | IAM policy changes, role grants/revokes, SA creation |
| **Data Access** | API calls reading/writing data | **Disabled by default** | `GetIamPolicy`, SA key creation, Secret Manager reads |
| **System Events** | Google system operations | Always on (Google-generated) | Auto-scaling, maintenance events |
| **Policy Denied** | Access denied by IAM or VPC-SC | Always on | Every access denied — valuable for troubleshooting |

> 🎯 **PCA Exam Catch:** **Data Access audit logs are off by default** and have cost implications when enabled. For compliance (SOC2, HIPAA, PCI-DSS), you MUST explicitly enable them. A question about "audit trail for all IAM reads (GetIamPolicy)" → requires enabling **Data Access audit logs for the IAM API**.

### Enabling Data Access Audit Logs

```bash
# Enable Data Access audit logs for IAM API (catches GetIamPolicy calls)
gcloud projects get-iam-policy PROJECT_ID > policy.yaml

# Add to policy.yaml:
auditConfigs:
- auditLogConfigs:
  - logType: DATA_READ
  - logType: DATA_WRITE
  service: iam.googleapis.com

gcloud projects set-iam-policy PROJECT_ID policy.yaml
```

### Centralized Audit Log Architecture

```
All Projects (org-wide)
    ↓ Log Sink (org-level aggregated sink)
    ↓ Filter: logName:"cloudaudit.googleapis.com"
Central Logging Project: audit-logs-prod
    └── BigQuery dataset: audit_logs_bq (90-day retention + analytics)
    └── Cloud Storage bucket: audit-logs-archive (7-year archive, Coldline)
    └── Pub/Sub topic: audit-alerts → Cloud Function → SIEM (Splunk/Chronicle)

Access control on central logging project:
    Security team → roles/logging.viewer (read-only)
    Audit SA → roles/logging.logWriter (write only)
    No one has roles/logging.admin on the logging project
```

### IAM Events in Audit Logs — Key Log Entries

```json
// Example: Role granted to a user
{
  "protoPayload": {
    "methodName": "SetIamPolicy",
    "authenticationInfo": {"principalEmail": "admin@scotiabank.com"},
    "request": {
      "policy": {
        "bindings": [{"role": "roles/storage.admin", "members": ["user:new@scotiabank.com"]}]
      }
    }
  },
  "resource": {"type": "project", "labels": {"project_id": "my-prod-project"}},
  "severity": "NOTICE"
}
```

### Policy Intelligence for Ongoing Compliance

```
Monthly IAM hygiene workflow:
    1. Policy Analyzer → export all IAM bindings org-wide to BigQuery
    2. Activity Analyzer → identify bindings with no usage in 90+ days
    3. IAM Recommender → auto-flag over-privileged roles
    4. Cross-reference: if binding unused for 90 days → generate ticket for review
    5. After human review → remove or downgrade stale bindings
    6. Document removals in change management system
```

---

## 15. BigQuery Data Access Controls

### Layered BigQuery Access Model

```
Layer 1: Project IAM
    → roles/bigquery.user / dataViewer / dataEditor / admin
    → Coarse — applies to all datasets in the project

Layer 2: Dataset IAM
    → Granted per dataset
    → finer — user can access only specific datasets

Layer 3: Table/View IAM
    → roles/bigquery.dataViewer on specific table
    → Finest native granularity in IAM

Layer 4: Column-level Security
    → Data Catalog Policy Tags (taxonomy)
    → roles/datacatalog.categoryFineGrainedReader on specific tag
    → Restricts which columns users can query

Layer 5: Row-level Security
    → BigQuery row-level access policies (row filter expressions)
    → User sees only rows matching their identity/group filter
```

### BigQuery Role Combination — Exam Critical

```
To QUERY tables in Project A's dataset:
    → roles/bigquery.jobUser (or bigquery.user) on Project A
    → roles/bigquery.dataViewer on the specific dataset or table
    Both are required — neither alone is sufficient

To QUERY + CREATE tables:
    → roles/bigquery.jobUser on project
    → roles/bigquery.dataEditor on dataset

To run DML (INSERT/UPDATE/DELETE):
    → roles/bigquery.jobUser on project
    → roles/bigquery.dataEditor on dataset
```

### Authorized Views — Cross-Project Data Sharing

An authorized view allows queries from Project B to access data in Project A's dataset **without granting Project B users direct table access**:

```sql
-- In Project A (source project)
-- Create a view that filters sensitive columns
CREATE VIEW project_a.public_dataset.employee_view AS
SELECT
  employee_id,
  department,
  hire_date
  -- Excludes: salary, SSN, home_address
FROM project_a.hr_dataset.employees;
```

```bash
# Authorize the view to access the source dataset
bq update --source_dataset_id=hr_dataset \
  --authorized_view=project_a:public_dataset.employee_view \
  project_a:hr_dataset

# Grant Project B's analysts access only to the VIEW (not the source table)
gcloud projects add-iam-policy-binding project_a \
  --member="group:analytics-team@project-b.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataViewer"
# Scope to public_dataset only — not hr_dataset
```

> 🎯 **PCA Exam Catch:** Authorized views are how you **share BigQuery data across projects without exposing source tables**. The view runs with the source dataset's permissions. Users of the view see only what the view exposes. This is the exam-correct pattern for "share BigQuery data with partner team in another project without giving them source table access."

### Column-Level Security with Policy Tags

```
Data Catalog Taxonomy:
    Financial Data (tag)
        └── PII (child tag)
                ├── SSN
                └── Credit Card Number

Grant fine-grained reader role on tag:
    Group "finance-analysts" → roles/datacatalog.categoryFineGrainedReader
        on SSN tag → Can query SSN column
    Group "marketing-analysts" → NOT granted on SSN tag → SSN column returns NULL
```

---

## 16. BeyondCorp & Context-Aware Access

### Zero-Trust Architecture Components

```
Traditional (perimeter-based):
    Corporate network → Trust everything inside → VPN clients
    Problem: Insider threats, compromised VPN endpoints

Zero-trust (BeyondCorp):
    No implicit network trust → Verify every request:
        1. User identity (IAP + Workforce Federation)
        2. Device posture (BeyondCorp — MDM, OS version, encryption)
        3. Context (IP, location, time)
        4. Resource access policy (IAM + Conditions)
```

### Cloud IAP (Identity-Aware Proxy)

IAP sits in front of HTTP(S) apps (GCE, GKE, Cloud Run, App Engine) and enforces identity verification:

```
User browser
    → IAP (identity verification: Google account or federated identity)
    → BeyondCorp Access Level check (device posture)
    → IAM policy check (does user have roles/iap.httpsResourceAccessor?)
    → Application (only sees authenticated request with user identity header)
```

```bash
# Grant access to IAP-protected app
gcloud iap web add-iam-policy-binding \
  --member="group:employees@scotiabank.com" \
  --role="roles/iap.httpsResourceAccessor" \
  --resource-type=backend-services \
  --service=my-backend-service
```

### Context-Aware Access Levels (BeyondCorp)

Access Levels define device and context requirements:

```yaml
# Access Level: Corporate Managed Device
name: "accessPolicies/POLICY_ID/accessLevels/corp_managed_device"
conditions:
  - devicePolicy:
      requireAdminApproval: true
      requireCorpOwned: true
      osConstraints:
        - osType: DESKTOP_WINDOWS
          minimumVersion: "10.0"
        - osType: DESKTOP_MAC_OS
          minimumVersion: "12.0"
      requireScreenlock: true
      allowedEncryptionStatuses: [ENCRYPTED]
  - regions: ["CA", "US"]  # Only from Canada or US
```

```bash
# Apply access level to IAM binding via condition
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="group:employees@scotiabank.com" \
  --role="roles/run.invoker" \
  --condition="title=corp-device-only,\
    expression=request.auth.access_levels.exists(x, x=='accessPolicies/POLICY_ID/accessLevels/corp_managed_device')"
```

> 🎯 **PCA Exam Catch:** BeyondCorp Access Levels are referenced in IAM Conditions via `request.auth.access_levels`. The condition checks whether the requester's device has been evaluated and assigned the named access level. If the question says "block access from unmanaged devices without VPN" → the answer is **IAP + BeyondCorp Access Level in IAM Condition**, NOT VPN or firewall rules.

---

## 17. AI/ML Service IAM Patterns

### Vertex AI — Least Privilege by Pipeline Stage

| Stage | SA | Required Roles | NOT Needed |
|-------|-----|---------------|-----------|
| Training | `training-sa@` | `roles/aiplatform.user` + `roles/bigquery.dataViewer` + `roles/storage.objectCreator` | `aiplatform.admin` |
| Model Registry | `registry-sa@` | `roles/aiplatform.user` (model create) + `roles/storage.objectAdmin` (artifacts) | Full `storage.admin` |
| Inference endpoint | `inference-sa@` | `roles/aiplatform.predictor` (endpoint-only) | `aiplatform.user` (includes training ops) |
| Monitoring | `monitoring-sa@` | `roles/aiplatform.viewer` + `roles/monitoring.metricWriter` | — |

```
🎯 Exam Catch: aiplatform.predictor is scoped to prediction calls on a specific
endpoint. aiplatform.user grants training and deployment operations — too broad for
inference SAs. Never grant aiplatform.admin to production inference services.
```

### BigQuery ML IAM Chain

```
Training a BQML model requires:
    → roles/bigquery.dataEditor (write model to dataset)
    → roles/bigquery.jobUser (submit training job)
    → roles/storage.objectCreator (if using external data from GCS)

Querying/predicting from a BQML model requires:
    → roles/bigquery.dataViewer (read model)
    → roles/bigquery.jobUser (submit prediction job)

NOT required for prediction: dataEditor — prediction is read-only
```

### Sensitive Data Protection (DLP) IAM

```
Pipeline role needed                → IAM role
───────────────────────────────────────────────
Inspect content for PII             → roles/dlp.user
Create/manage inspection templates  → roles/dlp.admin
Write findings to BigQuery          → roles/dlp.user + roles/bigquery.dataEditor on findings dataset
View findings in SCC                → roles/securitycenter.findingsViewer
```

### Model Armor + IAM Integration

```
Model Armor template management:
    → roles/modelarmor.admin (create/modify templates)
    → roles/modelarmor.user (apply templates to endpoints)

Vertex AI endpoint with Model Armor:
    1. Model Armor SA must be able to inspect traffic
    2. Grant Vertex AI SA roles/modelarmor.templateViewer
    3. Attach template to endpoint via Vertex AI Endpoint update
```

---

## 18. Cross-Org & Multi-Tenant IAM

### VPC Peering Does NOT Share IAM

```
VPC Peering:           Shares network routes only
                       IAM stays 100% project-scoped
                       Resource in Project B is NOT accessible just because
                       VPCs are peered — IAM binding still required

Shared VPC:            Host project owns network + firewall
                       Service projects get roles/compute.networkUser on subnets
                       IAM for workloads still in service projects

PSC (Private Service Connect): Cross-org API access
    Service producer: publishes a service (requires Service Attachment)
    Service consumer: creates endpoint in their VPC
    IAM: consumer needs roles/servicedirectory.viewer + roles/compute.networkUser
         on consumer side to create the endpoint
```

### Multi-Tenant Isolation Pattern

```
Tenant A                    Tenant B
  Project: tenant-a-prod      Project: tenant-b-prod
  SA: app-sa@tenant-a         SA: app-sa@tenant-b
  VPC: tenant-a-vpc           VPC: tenant-b-vpc
       ↓                           ↓
       PSC endpoint                PSC endpoint
       ↓                           ↓
  Shared Service Project (producer)
  VPC: shared-service-vpc
  SA: service-sa@shared-svc
  → Serves both tenants via PSC
  → Tenant A and B cannot see each other's data
  → IAM binding: service-sa has viewer on each tenant's dataset
```

### Workload Identity Federation — Direct vs. SA Impersonation

```
Direct binding (preferred for simple cases):
    Federated identity → IAM role on resource directly
    No intermediate GCP SA
    Works for single-project access

SA impersonation (required for):
    → Cross-project access (federated identity needs to access multiple projects)
    → When SA-specific features needed (SA-signed URLs, etc.)
    → When the role needs to be auditable as a named GCP SA

Direct binding IAM format:
    principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/
      workloadIdentityPools/POOL/subject/SUBJECT_ID
```

---

## 19. IAM Limits & Operational Constraints

### Important Quota Limits

| Resource | Limit | Exam Implication |
|----------|-------|-----------------|
| Service accounts per project | 100 (default) | Request increase for large microservice deployments |
| IAM bindings per policy | 1,500 members per policy | Use groups to avoid hitting this limit |
| Custom roles per project | 300 | Organize at org level for reuse |
| Custom roles per org | 1,000 | Plan custom role taxonomy carefully |
| IAM Conditions per binding | 1 | Combine conditions with `&&` in CEL |
| Service account key size | 4 KB | Standard — not an architectural constraint |
| Workforce pool providers per pool | 5 | Plan multiple IdP integration carefully |

> 🎯 **PCA Exam Catch:** The 1,500 members-per-policy limit is why **groups** are architecturally required for large organizations — not just a best practice. At scale, individual user bindings exhaust the policy limit. The exam may present a scenario where an org with 2,000 developers is having IAM policy errors — the fix is **Google Groups**, not increasing the limit.

### IAM Propagation Timing

```
IAM policy change → Takes effect:
    GCP API calls:     Within seconds globally
    Full propagation:  Up to 60 seconds in edge cases
    Audit log record:  Near-real-time (< 1 minute)

Org Policy change → Takes effect:
    Resource creation: Immediately for new resources
    Existing resources: Not retroactive (no remediation of existing violations)

🎯 PCA Catch: Org Policies are not retroactive. If you set
constraints/compute.vmExternalIpAccess to DENY, existing VMs with external IPs
keep them. New VMs cannot be created with external IPs. You must separately remove
external IPs from existing VMs.
```

---

## 20. Scenario-Based Exam Questions

> **Instructions:** These questions match the style, difficulty, and deliberate ambiguity of real PCA 2026 exam questions. All options are plausible. Allow 60–90 seconds per question. Read every word of the scenario for hidden constraints.

---

### Question 1 — Multi-Team BigQuery Access with Data Segregation

**Scenario:** Scotiabank's data platform team manages a BigQuery project (`bq-central-prod`) containing two datasets:
- `hr_dataset`: Contains employee salary, performance scores, and SSNs
- `analytics_dataset`: Contains aggregated business metrics (no PII)

Requirements:
- Finance analysts (Finance group) need to query salary data in `hr_dataset` but must NOT see SSNs
- Business analysts (BizAnalytics group) need full access to `analytics_dataset` only
- External auditors from a partner firm need read access to `analytics_dataset` for Q4 only — access must auto-expire
- Auditors use non-Google corporate email and cannot be added as Google accounts

**Question**: Which combination of controls **best** satisfies all requirements?

A) Grant `roles/bigquery.dataViewer` to Finance group on `hr_dataset`, to BizAnalytics on `analytics_dataset`. Create a Google account for auditors and grant time-bound `roles/bigquery.dataViewer` with IAM Condition.

B) Use Data Catalog policy tags on SSN column → grant `roles/datacatalog.categoryFineGrainedReader` to Finance group (excluding SSN tag). Grant BizAnalytics `roles/bigquery.dataViewer` on `analytics_dataset` only. Configure Workforce Identity Federation for auditor IdP → grant `roles/bigquery.dataViewer` on `analytics_dataset` with IAM Condition expiry for Q4.

C) Create authorized view in `hr_dataset` excluding SSN column → grant Finance group access to view only. Grant BizAnalytics `roles/bigquery.dataViewer` on `analytics_dataset`. Provision temporary Google accounts for auditors with `roles/bigquery.dataViewer` and IAM Condition expiry.

D) Grant Finance group `roles/bigquery.dataViewer` on `hr_dataset` and configure row-level security to exclude SSNs. Grant BizAnalytics `roles/bigquery.user` on the full project. Create a signed URL for auditors to access `analytics_dataset` for Q4.

---

**Answer: B**

**Rationale — map every requirement:**

| Requirement | Option B Solution | Why Others Fail |
|-------------|-----------------|----------------|
| Finance sees salary but NOT SSN | Policy tag on SSN column; Finance lacks `categoryFineGrainedReader` on SSN tag → column returns masked | A: `dataViewer` on dataset exposes all columns including SSN; C: Authorized view works but requires view maintenance; Row-level security (D) controls rows, not columns |
| BizAnalytics: analytics_dataset only | `dataViewer` scoped to `analytics_dataset` dataset only | D: `bigquery.user` on the project gives access to all datasets |
| Auditor auto-expiry Q4 | IAM Condition `request.time < timestamp("2026-12-31T00:00:00Z")` | — |
| Auditors can't be Google accounts | Workforce Identity Federation enables external IdP identities in GCP IAM | A, C: Creating Google accounts violates the constraint; D: Signed URLs cannot be used for interactive BigQuery console queries |

**🎯 Trap:** Option C (authorized view) is almost correct for the Finance/SSN requirement but fails for the auditor requirement — creating temporary Google accounts violates the stated constraint. Option B handles all four requirements in one consistent architecture.

---

### Question 2 — Multi-Environment CI/CD with Least Privilege

**Scenario:** A financial services company uses Cloud Build for CI/CD across three environments (dev, staging, prod). The security team has identified the following risks:
- The current Cloud Build SA has `roles/editor` on all three projects
- A compromised build script could deploy malicious images to production
- Developers can trigger prod deployments directly from feature branches
- No approval gate exists before production deployment

**Question**: Which architecture **best** remediates all identified risks with minimum operational overhead?

A) Create separate Cloud Build triggers per environment. Grant the Cloud Build default SA `roles/container.developer` on each project. Restrict prod triggers to the `main` branch only.

B) Create separate SAs per environment (`ci-dev-sa`, `ci-staging-sa`, `ci-prod-sa`). Grant each SA `roles/container.developer` on its respective project only. Configure Cloud Build triggers: dev/feature branches → `ci-dev-sa`; `main` branch → `ci-staging-sa`; prod deployment requires Cloud Deploy approval gate → `ci-prod-sa`. Apply Org Policy `constraints/iam.disableServiceAccountKeyCreation` on all projects.

C) Use a single CI/CD SA with `roles/container.developer` on all three projects and add IAM Conditions restricting the SA's access to each project based on the pipeline stage environment tag.

D) Replace Cloud Build with Tekton on GKE. Use Workload Identity for the pipeline SA. Grant `roles/container.developer` on each project and require manual kubectl apply for prod deployments.

---

**Answer: B**

**Rationale — each risk addressed:**

| Identified Risk | How Option B Remediates |
|----------------|------------------------|
| Cloud Build SA has `roles/editor` on all projects | Separate SAs with `roles/container.developer` only — no IAM, no billing access |
| Compromised script deploys to prod | Each SA has access to ONE environment only — `ci-dev-sa` has zero prod access |
| Developers trigger prod from feature branches | Prod SA only activated via Cloud Deploy approval gate, not directly from triggers |
| No prod approval gate | Cloud Deploy requires explicit approver action before prod SA executes |

**Why NOT others:**
- A: Using the **default Cloud Build SA** is the core problem — Option A fixes branch restrictions but keeps the shared SA with broad access. One compromise still affects all environments.
- C: IAM Conditions based on tags cannot prevent a script from using the SA's token at an arbitrary time. The tag condition on the SA binding doesn't scope WHERE the SA can be used — only when. A malicious script running in a dev pipeline could still call prod APIs using the SA's token if the condition logic has any gap.
- D: Tekton + GKE adds significant operational overhead (cluster management, Tekton maintenance) and doesn't improve the security posture beyond what Cloud Build + separate SAs achieves more simply.

**🎯 Trap:** Option C sounds sophisticated (IAM Conditions on SA) but IAM Conditions on a **role binding** don't prevent the SA token from being exfiltrated and used elsewhere. The only true isolation is separate SAs with separate project-scoped roles. The exam tests whether candidates understand that conditions restrict when the role applies, not what you can do with a stolen token.

---

### Question 3 — Federated Access with Branch-Level Restriction

**Scenario:** A company's DevOps team uses GitHub Actions to deploy to Google Cloud. Current state:
- SA JSON key stored as GitHub secret `GCP_SA_KEY`
- Key has `roles/run.admin` on the production project
- Security audit found the key was accidentally committed to a feature branch 6 months ago
- Requirement: Different branches should have different deployment permissions:
  - `feature/*` branches → deploy to dev project only
  - `main` branch → deploy to staging project only
  - Tagged releases (`v*`) → deploy to prod project with additional approval

**Question**: Which configuration **best** implements the requirements securely?

A) Rotate the SA key and store it more securely in Secret Manager. Add branch checks in the GitHub Actions workflow YAML to select the right project. Use the same SA key but with a different `--project` flag per branch.

B) Configure Workload Identity Federation with one pool/provider. Create three SAs (`deploy-dev-sa`, `deploy-staging-sa`, `deploy-prod-sa`) each with `roles/run.developer` on their respective projects. Use attribute conditions in IAM bindings: `feature/*` → `deploy-dev-sa`, `main` → `deploy-staging-sa`, `v*` tags → `deploy-prod-sa`. For prod, add a manual approval step in GitHub Actions before Workload Identity token is requested.

C) Configure Workload Identity Federation. Use one SA (`deploy-sa`) with `roles/run.developer` on all projects. Add IAM Conditions using `attribute.ref` to restrict access to specific branches. For prod, require a separate GitHub environment with protection rules.

D) Use GitHub Actions OIDC with separate Workload Identity pools per environment (dev-pool, staging-pool, prod-pool). Each pool maps to a dedicated SA with `roles/run.developer` on its project. GitHub environments enforce branch protection (feature/* → dev-pool, main → staging-pool, tag → prod-pool).

---

**Answer: B**

**Rationale:**

| Requirement | Option B |
|-------------|---------|
| No SA keys | WIF — keyless, OIDC token exchange |
| `feature/*` → dev only | IAM binding: `deploy-dev-sa` with `attribute.ref` condition matching `refs/heads/feature/*` |
| `main` → staging only | IAM binding: `deploy-staging-sa` with `attribute.ref=refs/heads/main` |
| Tagged releases → prod | IAM binding: `deploy-prod-sa` with `attribute.ref=refs/tags/v*` |
| Prod approval | GitHub Actions manual approval step before WIF token request |
| Separate per-environment permissions | Three separate SAs with one-project scope each |

**Why NOT others:**
- A: Still uses SA keys — the root security problem. Key rotation doesn't fix the fundamental risk of key-based authentication. Branch logic in workflow YAML is easily bypassed by modifying the YAML in a PR.
- C: One SA with `roles/run.developer` on all projects — if the SA token is obtained (even via WIF), it has access to all three projects. `attribute.ref` conditions on IAM bindings restrict when the token can be exchanged, but the SA itself is over-privileged. A compromised WIF provider config could bypass the condition.
- D: Separate pools per environment adds unnecessary complexity — one pool can support multiple providers and attribute conditions. The added value of separate pools doesn't justify the management overhead. Option B achieves the same isolation with one pool and attribute-based binding conditions.

**🎯 Trap:** Option D looks like "gold-plated security" with separate pools but is architecturally equivalent to Option B with more moving parts. Option C's single-SA approach is the most common real-world anti-pattern — it feels right but the SA is still over-privileged.

---

### Question 4 — Org-Wide Security Guardrails for Regulated Environment

**Scenario:** A healthcare organization (subject to HIPAA) has 50+ GCP projects managed by multiple teams. A security audit found:
1. Three projects have GCS buckets with `allUsers` on some objects
2. Two teams created SA keys that were committed to internal Git repos
3. One developer granted `roles/owner` to an external contractor's Gmail account on a project containing PHI
4. Some VMs run without Shielded VM enabled in production projects

**Question**: Which combination of controls **prevents all four issues at the org level** going forward?

A) IAM Deny Policy blocking `storage.objects.setIamPolicy` for `allUsers` at org level + Org Policy `constraints/iam.disableServiceAccountKeyCreation` + Org Policy `constraints/iam.allowedPolicyMemberDomains` + Org Policy `constraints/compute.requireShieldedVm` on prod folder

B) Cloud SCC alerts for public buckets + Cloud Monitoring for SA key creation + IAM Conditions blocking external accounts + VM compliance policy in SCC

C) Org Policy `constraints/storage.publicAccessPrevention` at org level + `constraints/iam.disableServiceAccountKeyCreation` at org level + `constraints/iam.allowedPolicyMemberDomains` at org level + `constraints/compute.requireShieldedVm` on prod folder only

D) VPC Service Controls perimeter for all PHI projects + Org Policy `constraints/iam.disableServiceAccountKeyCreation` + SCC CSCC standard tier + Manual IAM reviews quarterly

---

**Answer: C**

**Rationale — map each issue to a control:**

| Audit Finding | Control in Option C | Why Specifically This |
|--------------|--------------------|-----------------------|
| GCS buckets with `allUsers` | `constraints/storage.publicAccessPrevention` at org level | Prevents any bucket or object from being made public; retroactively revokes existing public access on existing buckets (unlike most org policies) |
| SA keys committed to Git | `constraints/iam.disableServiceAccountKeyCreation` org-wide | Blocks creation of downloadable keys entirely; eliminates the attack vector |
| External Gmail `roles/owner` | `constraints/iam.allowedPolicyMemberDomains` org-wide | Blocks any IAM binding to non-organizational accounts |
| VMs without Shielded VM | `constraints/compute.requireShieldedVm` on prod folder | Applied only to prod (dev teams need flexibility; prod requires compliance) |

**Why NOT others:**
- A: Uses IAM Deny Policy for the GCS issue, but `storage.publicAccessPrevention` org constraint is more appropriate — it's specifically designed for this use case and is simpler to configure. Deny Policy requires correct permission enumeration and exemption management. Option A would work but is more complex than necessary.
- B: All detections, no prevention. SCC alerts and Cloud Monitoring tell you AFTER a violation occurs — they don't prevent it. HIPAA requires prevention controls, not just detection.
- D: VPC-SC helps with data exfiltration but doesn't prevent public GCS ACLs or external IAM bindings. The combination doesn't address all four issues comprehensively.

**🎯 Trap:** `constraints/storage.publicAccessPrevention` is a key org policy that many candidates overlook — they default to IAM Deny Policy for this. The key differentiator: `publicAccessPrevention` is **retroactive** on existing buckets (unlike most org policies that only apply going forward). This makes it the correct choice for immediate remediation of existing public buckets.

---

### Question 5 — Zero-Trust Application Access for Hybrid Workforce

**Scenario:** A professional services firm is migrating its internal tools to GCP. Employees work from both corporate offices and home (personal laptops). Requirements:
- Employees on **corporate-managed laptops** with MDM enrollment should have full access to all internal apps
- Employees on **personal laptops** should only access low-sensitivity apps (project tracking, calendar)
- **No VPN client** should be required for either scenario
- Access must be revoked immediately when an employee's device fails compliance checks
- All access attempts must be auditable

**Question**: Which architecture **best** meets these requirements?

A) Deploy all internal apps on GCE behind a Cloud VPN gateway. Grant `roles/compute.networkUser` to all employees. Enforce MDM compliance via mobile device management software.

B) Deploy apps on GKE behind a GKE Ingress. Use Cloud Armor IP allowlist for corporate office IPs. Employees at home use VPN client to connect to corporate network, then access apps.

C) Deploy apps behind Cloud IAP. Create two BeyondCorp Access Levels: `corp_managed` (requires MDM enrollment + corporate device) and `low_sensitivity` (any authenticated Google account). Assign `roles/iap.httpsResourceAccessor` to all employees on low-sensitivity apps with `corp_managed` Access Level condition on high-sensitivity apps. Configure Context-Aware Access via BeyondCorp Enterprise.

D) Deploy apps on Cloud Run with IAM Conditions checking `request.auth.access_levels`. Create one BeyondCorp Access Level for all employees. Use Cloud Armor for DDoS protection. Revoke access by removing the employee's Google account.

---

**Answer: C**

**Rationale:**

| Requirement | Option C |
|-------------|---------|
| Corporate laptop → full access | IAP + `corp_managed` Access Level (MDM + corporate device check) on high-sensitivity apps |
| Personal laptop → low-sensitivity only | IAP + `low_sensitivity` Access Level (just authentication) on low-sensitivity apps; high-sensitivity blocked |
| No VPN required | IAP handles auth and access control — no VPN needed for any scenario |
| Immediate revocation on compliance fail | BeyondCorp re-evaluates device posture on every request; MDM compliance flag removed → `corp_managed` level no longer met → access blocked immediately |
| Audit trail | IAP access logs in Cloud Logging record every access attempt with user + device context |

**Why NOT others:**
- A: Requires VPN — violates the explicit "no VPN client" requirement. Also, `roles/compute.networkUser` is a network role, not an application access role.
- B: Requires VPN for home workers — violates the requirement. IP allowlist doesn't differentiate managed vs. personal devices at home (same home IP for both).
- D: Cloud Run IAM Conditions with Access Levels can work for Cloud Run specifically but the question describes "internal apps" (plural, mixed types). More critically, D creates only ONE Access Level for all employees — it doesn't implement the two-tier access model (corporate = full, personal = low-sensitivity only). Also, revoking by removing the Google account is drastic — device compliance revocation should revoke the access level, not the account.

**🎯 Trap:** Option D mentions IAM Conditions with `request.auth.access_levels` on Cloud Run, which is technically valid — but it only creates ONE access level without differentiation. The exam tests whether you read the full scenario: two device tiers require TWO access levels with different permissions. Candidates who focus on the technology (Access Levels in conditions) without matching the architecture to ALL requirements will choose D.

---

### Question 6 — Service Account Impersonation Chain for Auditable CI/CD

**Scenario:** A platform engineering team builds a self-service infrastructure provisioning tool where:
- Developers submit Terraform plans via a web portal (Cloud Run service)
- Plans are reviewed by a platform engineer in the portal
- After approval, Terraform apply runs via Cloud Build
- Terraform must create GKE clusters, GCS buckets, and IAM bindings in target projects
- A complete audit trail is required: who submitted, who approved, what was applied
- Developers must never have direct GCP access to create infrastructure themselves

**Question**: Which IAM architecture **best** implements this workflow with complete auditability?

A) Grant developers `roles/owner` on target projects so they can run Terraform locally. Platform engineers review plans in Git PRs. Cloud Build applies changes with a shared admin SA.

B) Cloud Run portal SA: `roles/viewer` on target projects (read-only for plan display). Platform engineer group: `roles/iam.serviceAccountTokenCreator` on a Terraform execution SA. Cloud Build: uses Terraform SA (impersonated after approval) with `roles/resourcemanager.projectIamAdmin` + `roles/container.admin` + `roles/storage.admin` on target projects. All SA impersonations logged in Cloud Audit Logs.

C) Developers submit Terraform plans via Cloud Run portal (portal SA: read-only). Cloud Build trigger requires approval from platform engineers. Cloud Build uses a dedicated Terraform SA with exactly `roles/container.admin` + `roles/storage.admin` + `roles/resourcemanager.projectIamAdmin` on target projects. Terraform SA is not directly accessible to developers or platform engineers — only Cloud Build can assume it.

D) Use Terraform Cloud for plan and apply. Grant Terraform Cloud's SA `roles/owner` on target projects. Platform engineers approve in Terraform Cloud UI. All actions logged in Terraform Cloud.

---

**Answer: C**

**Rationale:**

| Requirement | Option C |
|-------------|---------|
| Developers submit plans | Cloud Run portal with read-only SA — developers can use the portal, not direct GCP access |
| Platform engineer approval | Cloud Build trigger requires approval step — only after approval does apply run |
| Terraform creates GKE, GCS, IAM | Terraform SA has specific roles for these resources only — no `roles/owner` |
| Complete audit trail | Cloud Build approval logged; SA usage logged in Cloud Audit Logs; who triggered and who approved visible |
| Developers never have direct GCP access | Developers only interact with the portal; Terraform SA is only accessible by Cloud Build (not developers or platform engineers directly) |

**Why NOT others:**
- A: Grants developers `roles/owner` on target projects — completely violates the requirement "developers must never have direct GCP access to create infrastructure."
- B: Grants platform engineers `roles/iam.serviceAccountTokenCreator` on the Terraform SA — this means platform engineers can impersonate the Terraform SA and bypass the approval workflow entirely. A platform engineer could generate a token and run Terraform directly outside the approved flow. The audit trail is incomplete.
- D: Granting `roles/owner` to Terraform Cloud's SA violates least privilege. `roles/owner` includes billing and IAM management beyond what Terraform needs. Also, "Terraform Cloud" is a third-party service — for regulated environments, cloud-native tools (Cloud Build) are preferred over sending credentials to external SaaS.

**🎯 Trap:** Option B looks like a sophisticated impersonation chain and is almost correct — but the critical flaw is `roles/iam.serviceAccountTokenCreator` granted to platform engineers. This is an intentional exam trap: it sounds like it adds an approval step, but it actually means platform engineers have unilateral access to the powerful Terraform SA outside of any workflow. Option C keeps the Terraform SA accessible ONLY to Cloud Build's own SA — not to any human — which is the architecturally correct separation.

---

## 21. Quick Reference & Decision Frameworks

### IAM Exam Decision Framework

```
Q: Which role should I assign?
    → Avoid Editor/Owner → Predefined or Custom
    → BigQuery query: dataViewer + jobUser (both required)
    → GKE deploy: container.developer (not container.admin)
    → GCS single bucket read: storage.objectViewer on THAT bucket

Q: Where should I assign the role?
    → Lowest possible scope: resource > project > folder > org

Q: How should I authenticate?
    → GKE pod:           Workload Identity (KSA → GSA)
    → GCE/Cloud Run:     Attached SA (keyless)
    → GitHub Actions:    Workload Identity Federation (keyless)
    → AWS workload:      Workload Identity Federation (AWS SigV4)
    → Corporate users:   Workforce Identity Federation (Okta/ADFS/Azure AD)
    → Local dev:         gcloud ADC + SA impersonation
    → Legacy on-prem:    SA key in Secret Manager (last resort)

Q: How do I grant temporary/privileged access?
    → With approval workflow: PAM JIT
    → Without approval (ABAC): IAM Conditions (CEL) + version 3 policy
    → Note: Cannot add conditions to basic roles (owner/editor/viewer)

Q: How do I enforce org-wide guardrails?
    → Config prevention: Org Policy constraints
    → Permission enforcement: IAM Deny Policies (evaluated before allows)
    → Deny + Org Policy = defense in depth for compliance

Q: How do I enforce zero-trust app access?
    → IAP + BeyondCorp Access Levels + IAM Conditions
    → No VPN needed

Q: How do I share BigQuery data without exposing source tables?
    → Authorized views + scoped IAM on the view dataset

Q: How do I control SSH to VMs?
    → OS Login + roles/compute.osLogin (no sudo)
    → OS Login + roles/compute.osAdminLogin (sudo)
    → Enforce with Org Policy: constraints/compute.requireOsLogin
```

### IAM Security Controls Evaluation Matrix

| Requirement | Primary Control | Secondary | Exam Answer Pattern |
|-------------|----------------|-----------|---------------------|
| Prevent public GCS buckets | Org Policy: `publicAccessPrevention` | IAM Deny Policy | Org Policy is retroactive — preferred |
| Temporary contractor access | IAM Conditions (time-bound, v3 policy) | PAM JIT grant | Conditions if simple, PAM if approval needed |
| Emergency break-glass access | PAM with approval + auto-expiry | IAM Conditions | PAM — approval workflow is key differentiator |
| Block external Gmail in IAM | Org Policy: `allowedPolicyMemberDomains` | IAM Deny Policy | Org Policy — domain restriction |
| Secure GKE pod → BigQuery | Workload Identity → GSA → `bigquery.dataViewer` + `jobUser` | — | Never use SA keys in pods |
| Zero-trust device-aware app access | IAP + BeyondCorp Access Level + IAM Condition | — | No VPN, device-aware |
| GitHub CI/CD to GCP (no keys) | Workload Identity Federation + attribute mapping | — | Branch + repo restricted |
| SA key exposure prevention | Org Policy: `disableServiceAccountKeyCreation` + `disableServiceAccountKeyUpload` | IAM Deny Policy | Both org policies together |
| Cross-project data sharing BigQuery | Authorized views | Column policy tags | Authorized view = no source table access |
| VM SSH with audit trail | OS Login + `roles/compute.osLogin` | Org Policy: `requireOsLogin` | OS Login not optional for compliance |

### Complete PCA IAM Exam Trap Table

| Trap in Exam Question | Correct Approach |
|----------------------|-----------------|
| `roles/editor` or `roles/owner` for any production SA | Custom or predefined role scoped to resource |
| SA keys for GitHub Actions / CI/CD | Workload Identity Federation (keyless) |
| Block public GCS with Deny Policy when `publicAccessPrevention` exists | Use `publicAccessPrevention` Org Policy — it's retroactive |
| Add IAM Condition to `roles/owner` | Cannot — basic roles don't support conditions. Use PAM or custom role |
| Use labels in `resource.matchTag()` condition | Labels ≠ Tags. Only Resource Manager Tags work in IAM conditions |
| BigQuery query access with only `dataViewer` | Also need `jobUser` — both required to run queries |
| VPC Peering shares IAM across projects | False — peering shares routes only. IAM remains project-scoped |
| SAML for Workload Identity Federation | SAML is only for Workforce (human users). WIF uses OIDC or AWS SigV4 |
| `allUsers` on GCS for temporary external sharing | Use Signed URLs — time-bound, traceable, no IAM change |
| IAM Recommender suggestions as optional housekeeping | Treat as mandatory — stale bindings = compliance risk |
| PAM vs. IAM Conditions for break-glass with approval | PAM = approval + auto-expiry. Conditions alone = no approval workflow |
| Org Policy prevents existing violations | Org Policies are NOT retroactive (except `publicAccessPrevention`) |
| Grant `aiplatform.admin` to inference SA | Use `aiplatform.predictor` on endpoint — admin includes training/deployment ops |
| `serviceAccountTokenCreator` to platform engineers "for impersonation" | This grants unilateral SA access outside any workflow — use Cloud Build SA instead |
| Single SA for all CI/CD environments | Separate SA per environment — isolation is the control |
| Data Access audit logs are automatically enabled | Off by default — must explicitly enable per service for compliance |

---

*Enhanced Study Guide v2.0 | Aligned to PCA Exam 2026 | IAM Domain*  
*Added sections: PAM, Deny Policies, OS Login, Audit Logs, BigQuery Access Controls, BeyondCorp, AI/ML IAM, Cross-Org IAM, IAM Limits. Added 6 hard scenario questions with full trade-off analysis.*

> 🔄 Validate against: [Cloud IAM docs](https://cloud.google.com/iam/docs) | [PAM docs](https://cloud.google.com/privileged-access-manager/docs) | [BeyondCorp docs](https://cloud.google.com/beyondcorp-enterprise/docs)
