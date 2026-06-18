# ☁️ Google Cloud IAM — PCA Exam Study Guide
> **Target:** Google Cloud Professional Cloud Architect (PCA) Exam  
> **Domain:** Security, Compliance & Identity  
> **Topics Covered:** 10 Core Cloud IAM Areas

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
9. [IAM Conditions](#9-iam-conditions)
10. [Federated Authentication](#10-federated-authentication)

---

## 1. Cloud IAM — Introduction

### What is Cloud IAM?
Cloud Identity and Access Management (IAM) lets you define **who** (identity) can do **what** (role/permission) on **which resource** (resource hierarchy). It replaces ACL-style access control with a unified, declarative policy model.

### Core Components

| Component | Description |
|-----------|-------------|
| **Principal (Member)** | Who is making the request: Google Account, Service Account, Google Group, Workspace domain, allUsers, allAuthenticatedUsers |
| **Role** | A collection of permissions. Roles are assigned to principals, not permissions directly |
| **Permission** | Defines what operation is allowed on a resource (e.g., `storage.objects.get`) |
| **Policy** | A binding of principals to roles on a resource. Stored as a JSON document |
| **Resource** | The GCP entity being accessed (Project, Bucket, Dataset, VM, etc.) |

### IAM Policy Structure
```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:nilanjan@example.com",
        "serviceAccount:app-sa@project.iam.gserviceaccount.com"
      ]
    }
  ],
  "etag": "BwXXXXXX",
  "version": 3
}
```

### Resource Hierarchy & Policy Inheritance

```
Organization
    └── Folder (optional, for grouping)
            └── Project
                    └── Resource (GCS Bucket, VM, BigQuery Dataset...)
```

- **Policies are inherited downward.** A role granted at the Organization level flows to all Folders, Projects, and Resources beneath it.
- **Child policies cannot restrict parent-granted access.** You cannot revoke a permission granted at a higher level by setting a policy at a lower level.
- **The effective policy = union of all policies** at the resource level and all ancestor levels.

### Primary Use Cases
- **Fine-grained access control** — Grant minimal permissions per team/service
- **Separation of duties** — Enforce that no single identity has end-to-end critical path access
- **Auditing** — Every IAM change is logged in Cloud Audit Logs
- **Cross-project access** — Service accounts in one project accessing resources in another
- **Federated identity** — External workforce identities mapped to GCP roles

### When to Choose Cloud IAM (vs. Alternatives)

| Scenario | Use IAM | Use Alternative |
|----------|---------|-----------------|
| Control who calls GCP APIs | ✅ Yes | — |
| Row-level data access in BigQuery | Partial | Use column/row policies + IAM |
| Application-level authz | ❌ No | Firebase Auth, Identity Platform |
| Network-level access control | ❌ No | VPC Firewall Rules, VPC-SC |
| Secret access control | ✅ Yes (Secret Manager IAM) | — |

### Key Exam Points
- IAM policies use **allow model** (deny is the default; exceptions: Deny policies GA since 2023)
- Policy version must be **3** to use IAM Conditions
- `setIamPolicy` requires **owner** role or `resourcemanager.projects.setIamPolicy`
- Policy changes take effect **within seconds** globally but may take up to 60 seconds for all systems

---

## 2. Understanding Service Accounts

### What is a Service Account?
A service account is a **special Google account** that belongs to an application or VM, not a human user. It is both a **principal** (can be granted roles on resources) and a **resource** (roles can be granted on it, e.g., `roles/iam.serviceAccountUser`).

### Types of Service Accounts

| Type | Managed By | Key Use Case |
|------|-----------|--------------|
| **User-managed** | You | Custom apps, GKE workloads, Terraform |
| **Google-managed** | Google (auto-created) | Internal GCP service operations (e.g., Cloud Build SA) |
| **Default** | Google (auto-created per project) | Auto-attached to Compute Engine, App Engine — **avoid using in prod** |

> ⚠️ **Pitfall:** Default Compute Engine SA is auto-granted `roles/editor` on the project. Always create custom SAs with least privilege instead.

### Service Account Naming
```
SA_NAME@PROJECT_ID.iam.gserviceaccount.com
```
Example: `gke-node-sa@my-fintech-project.iam.gserviceaccount.com`

### Service Account as a Principal (Granting Roles TO a SA)
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

### Service Account as a Resource (Granting Roles ON a SA)
Used to allow users/other SAs to **impersonate** or **use** a service account.
```bash
# Allow a user to attach this SA to a VM
gcloud iam service-accounts add-iam-policy-binding \
  app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --member="user:developer@example.com" \
  --role="roles/iam.serviceAccountUser"
```

### Service Account Impersonation
Allows a principal to **act as** a service account, acquiring its permissions temporarily. Preferred over key files.

```bash
gcloud storage ls --impersonate-service-account=app-sa@project.iam.gserviceaccount.com
```

### Workload Identity (GKE Context)
The modern, keyless way to give Kubernetes workloads GCP IAM identities.

```
Kubernetes SA (k8s namespace/ksa-name)
        ↕ (Workload Identity binding)
GCP Service Account (project.iam.gserviceaccount.com)
```

```bash
# Bind KSA to GSA
gcloud iam service-accounts add-iam-policy-binding GSA_EMAIL \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"
```

### Common Pitfalls

| Pitfall | Risk | Fix |
|---------|------|-----|
| Using default Compute SA | Over-privileged (`Editor`) | Create custom SA per workload |
| Sharing SA across multiple apps | Blast radius if compromised | One SA per application/service |
| Granting `roles/owner` to a SA | Full project control | Use granular roles |
| Exporting SA keys | Key leak = identity theft | Use Workload Identity / impersonation |

### Architecture Tradeoffs

| Approach | Pros | Cons |
|----------|------|------|
| Per-workload SA | Least privilege, auditable | More SA management overhead |
| Shared SA | Simpler management | Violates least privilege, blast radius |
| SA impersonation | No key files | Requires IAM setup, slightly complex |
| Workload Identity | Keyless, auto-rotated | GKE only, setup complexity |

---

## 3. Service Account Key Management

### Types of Service Account Keys

| Type | Description | Rotation | Use Case |
|------|-------------|----------|----------|
| **Google-managed** | Auto-created/rotated by Google | Automatic | Default for GCP services, Workload Identity |
| **User-managed** | You create and download as JSON | Manual | External systems that can't use ADC or Workload Identity |

> ⚠️ **Exam Rule:** User-managed keys are a security risk. Always prefer Workload Identity, ADC, or impersonation over key files.

### Application Default Credentials (ADC)
ADC is the recommended authentication mechanism. The client library searches for credentials in this order:

```
1. GOOGLE_APPLICATION_CREDENTIALS env var (points to key file)
2. gcloud auth application-default credentials (local dev)
3. Attached service account (Compute Engine, GKE, Cloud Run, etc.)
4. Error
```

In production on GCP, step 3 (attached SA) is the secure, keyless approach.

### Creating and Managing Keys
```bash
# Create a key
gcloud iam service-accounts keys create key.json \
  --iam-account=app-sa@PROJECT_ID.iam.gserviceaccount.com

# List keys
gcloud iam service-accounts keys list \
  --iam-account=app-sa@PROJECT_ID.iam.gserviceaccount.com

# Delete a key
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=app-sa@PROJECT_ID.iam.gserviceaccount.com
```

### Enforcing Key Restrictions via Org Policy
```
# Disable SA key creation org-wide
constraints/iam.disableServiceAccountKeyCreation

# Disable SA key upload
constraints/iam.disableServiceAccountKeyUpload
```

### Key Rotation Best Practices
- Rotate keys every **90 days maximum** (align with PCI-DSS / SOC2)
- Use **Secret Manager** to store and version key files if they must exist
- Set up **Cloud Monitoring alerts** on key age using `iam.googleapis.com/service_account_key` audit log events
- Automate rotation with **Cloud Scheduler + Cloud Functions**

### Detecting Key Misuse
- Enable **Cloud Audit Logs** → Data Access logs for IAM
- Set up **Security Command Center** findings for exposed keys
- Use **Cloud Asset Inventory** to enumerate all active keys across the org

### Common Pitfalls

| Pitfall | Risk | Fix |
|---------|------|-----|
| Committing key.json to Git | Immediate credential exposure | Use `.gitignore`, Secret Manager, detect with SCC |
| Long-lived keys with no rotation | Stale credential risk | Automate rotation; set org policy max age |
| Multiple active keys per SA | Complicates audit | Keep 1 active key; delete old keys immediately after rotation |
| SA key for GKE pods | Keyfile in container image or env var | Use Workload Identity instead |

### Architecture Decision: Keys vs. Keyless

```
External system (on-prem, 3rd party)
    → SA Key (unavoidable, secure in Vault/Secret Manager)

GKE workload → Workload Identity Federation (keyless)

GCE / Cloud Run / Cloud Functions → Attached SA (keyless)

Developer local → gcloud ADC + impersonation (keyless)

Cross-org GCP → Workload Identity Federation (keyless)
```

---

## 4. Permissions and Roles

### Permissions
- Format: `SERVICE.RESOURCE.VERB` — e.g., `compute.instances.start`, `storage.buckets.create`
- Permissions are **never granted directly** to principals — they are bundled into Roles
- A permission **must be included** in a role assigned to the principal for the action to succeed

### Roles
A **role** is a named collection of permissions. There are three categories:

| Role Type | Prefix | Managed By | Granularity |
|-----------|--------|-----------|-------------|
| Basic (Primitive) | `roles/owner`, `roles/editor`, `roles/viewer` | Google | Very coarse |
| Predefined | `roles/storage.objectViewer` | Google | Service-level |
| Custom | `roles/[org or project-scoped]` | You | Exact fit |

### How IAM Policy Evaluation Works
```
Request → Resource → Collect all policies (resource + ancestors)
        → Does any binding match (principal + role contains permission)?
        → YES → Allow
        → NO  → Deny (implicit)
        → Deny Policy exists? → Explicit Deny overrides Allow
```

### Deny Policies (IAM v2 — GA)
- Separate from allow policies, evaluated **before** allow policies
- Used to enforce guardrails regardless of allow bindings
- Example: deny `roles/storage.admin` to all except a specific group

```bash
gcloud iam policies create deny-policy \
  --attachment-point=cloudresourcemanager.googleapis.com/projects/PROJECT_ID \
  --policy-file=deny-policy.json
```

### Viewing Effective Permissions
```bash
# What can this SA do on this project?
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com"

# Test permissions
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/PROJECT_ID
```

### Exam-Critical Permission Groups

| Category | Key Permissions |
|----------|----------------|
| IAM Admin | `resourcemanager.projects.setIamPolicy`, `iam.roles.create` |
| Service Account | `iam.serviceAccounts.actAs`, `iam.serviceAccounts.getAccessToken` |
| Org Admin | `resourcemanager.organizations.setIamPolicy` |
| Security | `iam.denypolicies.create`, `orgpolicy.policy.set` |

### Common Pitfalls

| Pitfall | Issue | Fix |
|---------|-------|-----|
| Granting Editor/Owner broadly | Violates least privilege | Use predefined or custom roles |
| Forgetting `actAs` permission | SA attachment to VM fails | Grant `roles/iam.serviceAccountUser` |
| Policy version 1 with conditions | Conditions silently dropped | Always use `"version": 3` |
| Confusing role on SA vs. role for SA | Access vs. identity | Understand dual nature of SA |

---

## 5. Role Type Details

### Basic (Primitive) Roles

| Role | Permissions | Use Case |
|------|-------------|----------|
| `roles/viewer` | Read-only across all services | Audit/read-only access |
| `roles/editor` | Viewer + modify (no IAM, billing) | **Avoid in prod** |
| `roles/owner` | Editor + IAM + billing | **Only for project owners** |

> ❌ **Exam Rule:** Never use basic roles in production. They are too broad and violate least privilege. The examiner will use this as a distractor in correct-answer options.

### Predefined Roles
Google maintains 1000+ predefined roles. Key patterns:

```
roles/SERVICE.admin           → Full control of SERVICE
roles/SERVICE.editor          → Read + Write (no IAM)
roles/SERVICE.viewer          → Read-only
roles/SERVICE.RESOURCE.VERB   → Narrow capability
```

**Commonly tested predefined roles:**

| Role | Key Capabilities |
|------|-----------------|
| `roles/container.admin` | Full GKE cluster control |
| `roles/container.developer` | Deploy to GKE, no cluster mgmt |
| `roles/container.clusterViewer` | Read cluster configs only |
| `roles/storage.admin` | Full GCS control incl. ACLs |
| `roles/storage.objectViewer` | Read objects only |
| `roles/bigquery.dataViewer` | Query tables, no table creation |
| `roles/bigquery.jobUser` | Run jobs (needed alongside dataViewer) |
| `roles/iam.serviceAccountUser` | Attach SA to resources / impersonate |
| `roles/iam.serviceAccountTokenCreator` | Generate tokens for SA |
| `roles/logging.logWriter` | Write logs (for app SAs) |
| `roles/monitoring.metricWriter` | Write custom metrics |
| `roles/secretmanager.secretAccessor` | Read secret versions |

### Custom Roles
Use when predefined roles are too broad.

```bash
# Create from a YAML definition
gcloud iam roles create customStorageReader \
  --project=PROJECT_ID \
  --file=custom-role.yaml
```

**custom-role.yaml:**
```yaml
title: "Custom Storage Reader"
description: "Read objects and list buckets only"
stage: GA
includedPermissions:
  - storage.objects.get
  - storage.objects.list
  - storage.buckets.list
```

### Custom Role Constraints

| Constraint | Detail |
|-----------|--------|
| Scope | Project-level or Org-level |
| Cannot include | Permissions from `iam`, `resourcemanager` admin perms, or deprecated perms |
| Stages | `ALPHA`, `BETA`, `GA`, `DISABLED` |
| Max permissions | 3000 per custom role |
| Inheritance | Custom roles do NOT inherit — must be explicitly assigned |

### Architecture Tradeoffs: Custom vs. Predefined

| Factor | Predefined | Custom |
|--------|-----------|--------|
| Maintenance | Google manages | You manage as APIs evolve |
| Precision | May be too broad | Exact least privilege |
| Auditability | Well-known names | Requires documentation |
| Org Policy | `constraints/iam.allowedPolicyMemberDomains` | Not impacted |

---

## 6. Principle of Least Privilege

### Definition
Grant principals **only the minimum permissions** required to perform their specific tasks — nothing more, nothing less.

### PoLP Framework for GCP

```
1. Identify the task the principal must perform
2. Find the minimal set of permissions for that task
3. Find or create the role that matches exactly
4. Assign at the lowest resource scope possible
5. Set time-bound conditions where applicable
6. Audit and remove excess permissions regularly
```

### Scope: Assign at the Lowest Level Possible

```
Organization   → Only for org-wide administrators
    └── Folder → For team-level or BU-level access
         └── Project → Standard scope for most workloads
              └── Resource → Granular (GCS bucket, BQ dataset, Secret)
```

> **Exam Pattern:** A question will show a role granted at Org level when it only needs to apply to one project. The answer is to scope it to the project or resource.

### PoLP Anti-Patterns to Recognize in Exam

| Anti-Pattern | What to Choose Instead |
|--------------|----------------------|
| `roles/editor` on a service account | Narrow predefined or custom role |
| `roles/owner` for a developer | `roles/container.developer` + specific roles |
| SA with `roles/storage.admin` to read one bucket | `roles/storage.objectViewer` on that bucket only |
| Shared SA across 10 microservices | Dedicated SA per microservice |
| Permanent admin access for break-glass | IAM Conditions with time bounds + PAM |

### Privileged Access Management (PAM)
For human admins who need occasional elevated access:
- Use **IAM Conditions** with time-bound expressions
- Use **Privileged Access Manager (PAM)** (GA 2024) for just-in-time access grants
- All access requests and grants logged in Cloud Audit Logs

### Policy Intelligence Tools

| Tool | Purpose |
|------|---------|
| **IAM Recommender** | ML-based suggestions to remove excess permissions |
| **Policy Analyzer** | Answer "who has access to X resource?" |
| **Policy Troubleshooter** | Debug "why can't user Y do action Z?" |
| **Activity Analyzer** | Show last-used date for each granted role |

```bash
# Check for excess permissions (IAM Recommender)
gcloud recommender recommendations list \
  --recommender=google.iam.policy.Recommender \
  --location=global \
  --project=PROJECT_ID
```

### Common Pitfalls

| Pitfall | Resolution |
|---------|-----------|
| Granting org-level roles for project-scoped needs | Scope to project or resource |
| Not reviewing permissions after role changes | Schedule quarterly IAM reviews |
| Ignoring IAM Recommender suggestions | Automate remediation via SCC + Findings |
| Granting allUsers/allAuthenticatedUsers | Audit with Policy Analyzer; use VPC-SC for data |

---

## 7. Least Privilege via Environment Separation

### Why Separate Environments?
Environment separation enforces least privilege at the **infrastructure boundary** level — not just the role level. It prevents dev credentials from accessing prod data, and ensures blast radius is limited.

### GCP Recommended Pattern: One Project Per Environment

```
Organization: bank.com
    ├── Folder: Production
    │       ├── Project: identity-platform-prod
    │       ├── Project: payments-prod
    │       └── Project: shared-vpc-prod
    ├── Folder: Staging
    │       ├── Project: identity-platform-staging
    │       └── Project: payments-staging
    └── Folder: Development
            ├── Project: identity-platform-dev
            └── Project: payments-dev
```

### Folder-Level IAM to Enforce Separation

```bash
# Grant dev team access ONLY to the dev folder
gcloud resource-manager folders add-iam-policy-binding FOLDER_ID \
  --member="group:dev-team@bank.com" \
  --role="roles/editor"

# Prod folder: only CI/CD SA + SRE team
gcloud resource-manager folders add-iam-policy-binding PROD_FOLDER_ID \
  --member="serviceAccount:ci-cd-sa@devops-project.iam.gserviceaccount.com" \
  --role="roles/container.developer"
```

### Org Policies per Environment Folder

| Policy | Dev Folder | Prod Folder |
|--------|-----------|-------------|
| `constraints/compute.requireShieldedVm` | Not enforced | Enforced |
| `constraints/gcp.resourceLocations` | Multi-region allowed | `northamerica-northeast1` only |
| `constraints/iam.disableServiceAccountKeyCreation` | Allowed | Enforced |
| `constraints/compute.vmExternalIpAccess` | Allowed | Denied |

### Service Account Isolation

```
Dev SA:  app-sa@identity-platform-dev.iam.gserviceaccount.com
           → Can only access dev resources
           
Prod SA: app-sa@identity-platform-prod.iam.gserviceaccount.com
           → Isolated to prod project
           → No cross-project access unless explicitly granted
```

**Never share service accounts across environments.**

### CI/CD with Environment Separation

```
Developer commits code
    → Cloud Build triggers in CI project
    → SA: ci-build-sa@ci-project.iam.gserviceaccount.com
    → Has roles/container.developer on DEV project only
    
Approved PR merged to main
    → CD pipeline SA: cd-deploy-sa@ci-project.iam.gserviceaccount.com
    → Has roles/container.developer on STAGING project only
    
Manual approval gate (or automated after tests)
    → Prod deploy SA: prod-deploy-sa@ci-project.iam.gserviceaccount.com
    → Has roles/container.developer on PROD project only
    → All prod changes logged and require PR approval
```

### VPC Service Controls (complement to IAM)
Even with IAM, APIs can be called from outside your org. **VPC-SC** creates a security perimeter:
- Blocks data exfiltration from BigQuery, GCS, etc.
- Prevents API calls from outside the perimeter
- Works alongside IAM — both must allow the action

### Architecture Tradeoffs

| Approach | Pros | Cons |
|----------|------|------|
| Single project, multiple namespaces | Simpler mgmt | Blast radius spans envs, hard to IAM-separate |
| Separate projects per env | True isolation, separate billing, org policy control | More overhead, cross-project networking needed |
| Separate orgs per env | Maximum isolation | Extremely high overhead, rarely necessary |

> **Exam Answer:** For enterprises and regulated industries (banks, healthcare), the answer is **separate projects per environment**, organized under folders.

---

## 8. Groups

### What are Groups?
Google Groups (Workspace/Cloud Identity groups) are collections of user accounts and service accounts. Assigning IAM roles to **groups** instead of individual users is the Google-recommended best practice.

### Why Use Groups for IAM?

| Benefit | Explanation |
|---------|-------------|
| **Scalability** | Add/remove users from a group — IAM policy unchanged |
| **Auditability** | IAM policy shows group intent (e.g., `data-engineers`) |
| **Reduced policy churn** | Onboarding/offboarding doesn't touch IAM bindings |
| **Role clarity** | Group names encode intent (`prod-gke-admins@corp.com`) |

### Recommended Group Naming Convention

```
<environment>-<service>-<role>@corp.com

Examples:
prod-gke-admins@scotiabank.com
staging-data-engineers@scotiabank.com
dev-all-developers@scotiabank.com
platform-sre-oncall@scotiabank.com
security-auditors@scotiabank.com
```

### Group Types in GCP Context

| Group Type | Best For |
|-----------|---------|
| **Google Group** | Standard IAM role assignment |
| **Cloud Identity Group** | Includes dynamic membership rules |
| **Security Group** | Enforces group-based VPC-SC, stricter controls |
| **Posix Group** | Linux OS Login group for VM SSH access |

### Granting Roles to a Group
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="group:prod-gke-admins@scotiabank.com" \
  --role="roles/container.admin"
```

### allUsers vs allAuthenticatedUsers

| Member | Meaning | Risk |
|--------|---------|------|
| `allUsers` | **Any** internet user, no auth required | Public access — never on sensitive resources |
| `allAuthenticatedUsers` | Any Google account (not just your org) | Still very broad — avoid in regulated environments |

> ⚠️ **Exam Trap:** A question may ask the "least risky way to share a GCS object publicly." The answer is NOT `allUsers` on the bucket — use **signed URLs** for temporary, traceable public access.

### Groups with Org Policies
```
# Restrict IAM bindings to only your domain
constraints/iam.allowedPolicyMemberDomains
    → Allowed value: your Cloud Identity customer ID
    → Blocks adding external Gmail accounts to IAM
```

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Assigning roles to individual users | Use groups; individuals change teams |
| Not managing group membership lifecycle | Integrate with HR system / IdP for auto-provisioning |
| Overloaded groups (e.g., `all-engineers`) | Create purpose-scoped groups matching least privilege |
| `allAuthenticatedUsers` on sensitive data | Use explicit group membership + VPC-SC |

---

## 9. IAM Conditions

### What are IAM Conditions?
IAM Conditions allow you to define **attribute-based access control (ABAC)** — granting roles only when specific conditions are met, rather than unconditionally.

> Conditions are part of **Policy version 3**. Always set `"version": 3` in your policy document when using conditions.

### Condition Attributes Available

| Category | Attributes | Example |
|----------|-----------|---------|
| **Time** | `request.time` | Grant access only during business hours |
| **Resource** | `resource.name`, `resource.type`, `resource.service` | Limit to specific GCS bucket or prefix |
| **Request** | `request.host`, `request.path`, `request.method` | IAP-protected app conditions |
| **Tag** | `resource.matchTag()` | Access based on resource tags |
| **IP/Network** | Via Access Context Manager | Not directly in conditions — use Access Levels |

### Condition Expression Language
Conditions use **Common Expression Language (CEL)**.

```cel
# Time-based: only allow access during weekdays 9am-5pm UTC
request.time.getHours("UTC") >= 9 &&
request.time.getHours("UTC") < 17 &&
request.time.getDayOfWeek("UTC") >= 1 &&
request.time.getDayOfWeek("UTC") <= 5

# Resource-based: only allow access to a specific GCS bucket prefix
resource.name.startsWith(
  "projects/_/buckets/my-prod-bucket/objects/reports/")

# Temporary access: expires after a specific date
request.time < timestamp("2025-12-31T00:00:00Z")
```

### Policy Binding with Condition (JSON)
```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": ["user:contractor@external.com"],
      "condition": {
        "title": "Temporary contractor access",
        "description": "Expires end of Q4 2025",
        "expression": "request.time < timestamp(\"2025-12-31T00:00:00Z\")"
      }
    }
  ],
  "version": 3
}
```

### Use Cases

| Use Case | Condition Type |
|----------|---------------|
| Contractor temporary access | Time-bound expiry |
| Access only to resources tagged `env:prod` | Resource tag match |
| Break-glass access window | Time window |
| Isolate access to a specific bucket path | `resource.name.startsWith()` |
| Enforce access from specific network (with BeyondCorp) | Access Level condition |

### Resource Tags with Conditions
Tags are key-value labels attached to resources and used in IAM condition expressions.

```bash
# Create a tag key/value at org level
gcloud resource-manager tags keys create env \
  --parent=organizations/ORG_ID

gcloud resource-manager tags values create prod \
  --parent=tagKeys/TAG_KEY_ID

# Attach tag to a project
gcloud resource-manager tags bindings create \
  --tag-value=tagValues/TAG_VALUE_ID \
  --parent=//cloudresourcemanager.googleapis.com/projects/PROJECT_ID
```

```cel
# IAM Condition using the tag
resource.matchTag("ORG_ID/env", "prod")
```

### IAM Conditions vs. VPC Service Controls vs. Org Policy

| Mechanism | Controls | Granularity |
|-----------|---------|-------------|
| IAM Conditions | Who + when + which resource | Per binding |
| VPC Service Controls | API-level perimeter | Per service/project |
| Org Policy | What configurations are allowed | Per constraint |

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Policy version 1 or 2 with conditions | Always use version 3 |
| Condition on `roles/owner` | Not allowed — basic roles cannot have conditions |
| Using conditions without testing | Use Policy Troubleshooter to validate |
| Time conditions without UTC clarity | Always specify timezone in CEL expressions |

---

## 10. Federated Authentication

### What is Federated Authentication?
Federated authentication lets **external identities** (from your corporate IdP, GitHub, AWS, Azure, etc.) access Google Cloud resources **without creating Google accounts or SA keys**.

### Two Key Federation Models

| Model | Use Case | GCP Product |
|-------|---------|-------------|
| **Workforce Identity Federation** | Human users from external IdP (Okta, ADFS, Azure AD) | Workforce Identity Pools |
| **Workload Identity Federation** | External workloads (GitHub Actions, AWS, Azure, on-prem) | Workload Identity Pools |

### Workforce Identity Federation

```
Corporate User (Okta / ADFS / Azure AD)
    → OIDC/SAML token issued by IdP
    → Exchanged via STS for short-lived Google credential
    → Mapped to IAM principal: principal://iam.googleapis.com/...
    → Access GCP APIs / Console
```

**Setup (OIDC example):**
```bash
# Create workforce pool
gcloud iam workforce-pools create corp-pool \
  --organization=ORG_ID \
  --location=global

# Create OIDC provider
gcloud iam workforce-pools providers create-oidc okta-provider \
  --workforce-pool=corp-pool \
  --location=global \
  --issuer-uri="https://corp.okta.com" \
  --client-id="CLIENT_ID" \
  --attribute-mapping="google.subject=assertion.sub,google.groups=assertion.groups"
```

### Workload Identity Federation

```
External Workload (GitHub Actions / AWS EC2 / Azure VM / On-prem)
    → Issues OIDC/SAML/AWS token
    → Google STS validates token against pool provider config
    → Exchanges for short-lived access token
    → Impersonates a GCP Service Account (or direct access)
    → Accesses GCP APIs
```

**GitHub Actions example:**
```yaml
# .github/workflows/deploy.yaml
- id: auth
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL/providers/PROVIDER'
    service_account: 'github-actions-sa@PROJECT_ID.iam.gserviceaccount.com'
```

```bash
# Allow GitHub repo to impersonate GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  github-actions-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL/attribute.repository/my-org/my-repo"
```

### Attribute Mapping
Maps claims from the external token to Google attributes used in IAM bindings.

| External Claim | Google Attribute |
|---------------|-----------------|
| `assertion.sub` | `google.subject` |
| `assertion.email` | `google.email` |
| `assertion.groups` | `google.groups` |
| `assertion.repository` | `attribute.repository` (custom) |

### Principal Formats for Federated Identities

| Scope | Format |
|-------|--------|
| Single identity | `principal://iam.googleapis.com/locations/global/workforcePools/POOL/subject/USER` |
| All pool members | `principalSet://iam.googleapis.com/locations/global/workforcePools/POOL/*` |
| By attribute | `principalSet://...workforcePools/POOL/attribute.department/engineering` |
| By group | `principalSet://...workforcePools/POOL/group/sre-team` |

### GKE Workload Identity vs. Workload Identity Federation

| | GKE Workload Identity | Workload Identity Federation |
|--|----------------------|------------------------------|
| **Source** | Kubernetes SA in GKE | External: GitHub, AWS, Azure, on-prem |
| **Mechanism** | K8s SA token → GSA via annotation | External OIDC/SAML/AWS token → STS |
| **Key files needed** | No | No |
| **Scope** | GKE only | Any external system |

### SAML vs. OIDC for Federation

| | SAML 2.0 | OIDC |
|--|---------|------|
| Format | XML assertions | JWT tokens |
| Common IdPs | ADFS, Okta (SAML), Ping | Okta (OIDC), Azure AD, GitHub |
| GCP Support | Workforce only | Both Workforce + Workload |
| Complexity | Higher | Lower |

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Creating SA keys for GitHub Actions | Use Workload Identity Federation |
| Mapping overly broad groups from IdP | Use attribute conditions to scope access |
| Not validating token issuer/audience | GCP validates this — ensure issuer-uri is accurate |
| Using federation without audit logs | Enable Data Access audit logs for STS token exchanges |

---

## Quick Reference: IAM Exam Decision Framework

```
Q: Which role should I assign?
    → Avoid Basic (Editor/Owner) in production → Use Predefined or Custom

Q: Where should I assign the role?
    → Lowest resource scope possible (resource > project > folder > org)

Q: How should I authenticate?
    → GKE: Workload Identity
    → External: Workload Identity Federation
    → Human: Workforce Identity Federation or Cloud Identity
    → Local dev: gcloud ADC
    → Avoid SA keys

Q: How do I enforce environment separation?
    → Separate projects per env, organized in folders
    → Separate SAs per env, no cross-env SA sharing
    → Org Policies scoped per folder

Q: How do I grant temporary/conditional access?
    → IAM Conditions (CEL) with time bounds or resource tags
    → Version 3 policy required
    → PAM for just-in-time admin access

Q: How do I manage users at scale?
    → Groups not individuals
    → Group names encode purpose and scope
    → Integrate with corporate IdP
```

---

## Frequently Tested Exam Scenarios

| Scenario | Correct Answer |
|----------|---------------|
| A developer needs to deploy to GKE but not manage clusters | `roles/container.developer` |
| A Cloud Function needs to write to a specific GCS bucket | Custom SA + `roles/storage.objectAdmin` on that bucket only |
| CI/CD pipeline in GitHub needs to deploy to Cloud Run | Workload Identity Federation — no SA key |
| Contractor needs GCS access for 30 days only | IAM Condition with `request.time < timestamp(...)` |
| Prevent any SA key creation across the org | Org Policy: `constraints/iam.disableServiceAccountKeyCreation` |
| Debug why a user can't access a BigQuery dataset | Policy Troubleshooter |
| Find who has admin access across all projects | Policy Analyzer |
| Reduce over-privileged roles across the org | IAM Recommender |
| On-prem service needs to call GCP APIs without a key file | Workload Identity Federation |
| Human users from Okta need GCP Console access | Workforce Identity Federation |
| GKE pod needs to read from Cloud Spanner | Workload Identity (KSA → GSA) |
| Share a GCS object temporarily and publicly | Signed URL — not `allUsers` |

---

*Study Guide Version: 1.0 | Aligned to PCA Exam Guide (2024–2025) | Good luck on your exam! 🎯*

# 📘 PCA IAM Enrichment Addendum (2026 Exam Focus)
> *Append this to your existing `cloud-iam-pca-study-guide.md`. All sections are mapped to PCA scoring domains, include 2026 exam updates, and highlight high-frequency test patterns.*

---

## 🔍 Missing & High-Yield Topics for PCA 2026

### 1. Privileged Access Manager (PAM) — *GA 2024*
| Feature | Exam Focus | PCA Catch |
|--------|------------|-----------|
| **Just-in-Time (JIT) Access** | Time-bound elevation with auto-expiry | Replaces long-lived admin roles; answers must mention PAM for "break-glass" scenarios |
| **Approval Workflows** | Multi-step approval for sensitive roles | Can require 1+ approvers; integrates with Cloud Identity/Workspace |
| **Access Grants & Audit** | All grants logged in Cloud Audit Logs | Required for compliance (SOC2, HIPAA, PCI); PAM + Logging = audit-ready |
| **Emergency Access** | Time-boxed admin with strict monitoring | Exam tests PAM over static `roles/owner` for emergency scenarios |

🎯 **Exam Decision Tree**: 
```
Need temporary elevated access?
  → PAM JIT grant with approval + auto-expire
Need permanent admin?
  → Custom role + IAM Conditions (time/resource bounds)
Need break-glass for outage?
  → PAM emergency access tier + mandatory approval + audit logging
```

### 2. IAM Deny Policies (v2) — *Evaluation Order*
| Rule | Behavior | Exam Implication |
|------|----------|------------------|
| **Explicit Deny > Implicit Deny > Allow** | Deny policies are evaluated BEFORE allow bindings | Overrides any `roles/editor` or `roles/owner` grant |
| **Scope Inheritance** | Org/Folder deny policies flow to projects/resources | Cannot be overridden at lower levels |
| **Use Cases** | Prevent public buckets, block external key uploads, restrict regions | Exam tests "enforce org-wide guardrails" → Deny Policies |

⚠️ **Exam Trap**: Questions may ask "How to prevent any user from creating service account keys across the org?" → Answer: `constraints/iam.disableServiceAccountKeyCreation` (Org Policy) OR IAM Deny Policy blocking `iam.serviceAccountKeys.create`. Both work, but Deny Policies are newer and override allows explicitly.

### 3. BeyondCorp Enterprise & Context-Aware Access
| Component | IAM Integration | PCA Focus |
|-----------|----------------|-----------|
| **Access Levels** | IP, device posture, OS version, location | Referenced in IAM Conditions via `request.auth.access_levels` |
| **Context-Aware Access** | Replaces VPN for zero-trust app access | Works with IAP + IAM to enforce device compliance |
| **Conditional IAM** | `request.auth.access_levels.matches("LEVEL_ID")` | Exam tests "secure access from unmanaged devices" → Context-Aware + IAM Conditions |

💡 **Exam Tip**: BeyondCorp is Google's zero-trust implementation. If a question mentions "verify device security before granting access," the answer combines **BeyondCorp Access Levels + IAM Conditions + IAP**.

### 4. AI/ML Service IAM Patterns (2026 Focus)
| Service | Training IAM | Inference IAM | Exam Trap |
|---------|--------------|---------------|-----------|
| **Vertex AI** | `roles/aiplatform.user` + `roles/storage.objectCreator` | `roles/aiplatform.predictor` (endpoint-level) | `aiplatform.admin` is too broad for prod inference |
| **BigQuery ML** | `roles/bigquery.dataEditor` + `roles/bigquery.jobUser` | `roles/bigquery.dataViewer` | Model training requires `jobUser`; querying models needs `dataViewer` |
| **Model Garden** | `roles/aiplatform.admin` for publishing | `roles/aiplatform.user` for deployment | Third-party model deployment requires `aiplatform.modelUploader` |
| **Sensitive Data Protection** | `roles/dlp.admin` | `roles/dlp.user` | DLP API requires separate IAM from storage; exam tests cross-service IAM chains |

🎯 **Exam Catch**: AI/ML IAM questions test **least privilege across pipeline stages**. Training ≠ Inference ≠ Data Access. Always split roles per stage.

### 5. Cross-Org & Multi-Tenant IAM Architecture
| Pattern | Implementation | PCA Use Case |
|---------|----------------|--------------|
| **Workload Identity Federation** | External OIDC/SAML → STS → GCP SA | GitHub Actions, AWS, Azure accessing GCP without keys |
| **Org Policy Inheritance** | Constraints flow down; cannot be relaxed | `constraints/iam.allowedPolicyMemberDomains` blocks external users |
| **PSC + IAM** | Service attachment requires `roles/servicedirectory.viewer` + `roles/compute.networkUser` | Third-party SaaS or cross-org API access |
| **Centralized IAM + Shared VPC** | Host project owns network; service projects get `networkUser` | Multi-project enterprises with centralized security |

⚠️ **Exam Trap**: VPC peering does NOT share IAM. IAM stays project-bound. Use **Shared VPC + subnet-level IAM** or **PSC + cross-org IAM bindings**.

---

## 🧩 Advanced PCA Decision Frameworks

### IAM + Security Controls Evaluation Matrix
| Requirement | Primary Control | Secondary Control | Exam Answer Pattern |
|-------------|----------------|-------------------|---------------------|
| Prevent public GCS buckets | IAM Deny Policy + Org Policy | VPC-SC + SCC | Deny policy evaluated first; blocks all allows |
| Temporary contractor access | IAM Conditions (time-bound) | PAM JIT grant | Conditions + version 3 policy |
| Secure GKE pod → BigQuery access | Workload Identity | Custom SA + `bigquery.dataViewer` | Never use SA keys; KSA→GSA binding |
| Cross-project CI/CD deployment | Workload Identity Federation | Impersonation SA + `roles/iam.serviceAccountTokenCreator` | Keyless federation preferred |
| Device compliance before API access | BeyondCorp Access Levels + IAM Conditions | Context-Aware Access | Zero-trust pattern, not VPN |

### Service Account Impersonation Chains
```
Developer (user@corp.com)
  → impersonates → ci-build-sa@project.iam.gserviceaccount.com
  → token created via → roles/iam.serviceAccountTokenCreator
  → deploys to → Cloud Run/GKE
  → uses → attached workload SA for runtime permissions

Exam Rule: 
  • TokenCreator = generates short-lived credentials
  • ServiceAccountUser = attaches SA to resource
  • Never grant both unless explicitly required for CI/CD pipelines
```

---

## ⚠️ 2026 Exam Traps & Anti-Patterns

| Trap in Question | Correct Approach | Why |
|------------------|------------------|-----|
| "Grant `roles/editor` to a service account for simplicity" | Use predefined/custom role scoped to resource | Violates least privilege; exam penalizes broad roles |
| "Use SA keys for GitHub Actions deployment" | Workload Identity Federation | Keys are security risk; federation is keyless & auditable |
| "Block public buckets with project-level IAM only" | Org-level IAM Deny Policy + Org Policy | Project IAM can be overridden; deny policies are absolute |
| "Share GCS data publicly via `allUsers` on bucket" | Signed URLs or Identity-Aware Proxy | `allUsers` is untraceable; exam favors controlled, time-bound access |
| "Use basic roles (`owner`/`editor`) for compliance workloads" | Custom roles + IAM Conditions + PAM | Basic roles fail audit; compliance requires explicit, auditable bindings |
| "Grant `aiplatform.admin` to inference service" | `roles/aiplatform.predictor` on endpoint | Admin includes training/model management; inference needs minimal scope |
| "Rely on VPC peering for cross-project IAM sharing" | Shared VPC + IAM bindings or PSC | Peering shares routes, not IAM policies |

---

## 📝 Scenario-Based Practice Questions (PCA 2026 Style)

### Question 1: Break-Glass Access & Compliance
> **Scenario**: A financial services company requires a "break-glass" procedure for SREs during production outages. Requirements:
> - Access must be time-bound (max 2 hours)
> - Requires manager approval before activation
> - All actions must be auditable for SOC2 compliance
> - Must not use long-lived admin roles
>
> **Question**: Which IAM configuration BEST meets these requirements?
>
> A) Create a custom role with `roles/owner` permissions and assign to SRE group with IAM Conditions for 2-hour expiry  
> B) Implement Privileged Access Manager (PAM) with JIT access, mandatory approval workflow, and 2-hour auto-expire  
> C) Store a service account key in Secret Manager and require SREs to check it out via ticketing system  
> D) Grant `roles/container.admin` to SREs with IAM Conditions restricting access to business hours only  
>
> **Answer**: B  
> **Rationale**: PAM provides JIT access with approval workflows, strict time limits, and automatic expiry. All grants are logged in Cloud Audit Logs, satisfying SOC2. Option A uses static IAM conditions that don't enforce approval or auto-revoke effectively; C introduces key management risk; D lacks approval and break-glass specificity. PAM is the Google-recommended pattern for privileged access.

### Question 2: AI/ML Pipeline Least Privilege
> **Scenario**: A data science team builds a fraud detection model:
> - Training uses Vertex AI with BigQuery data
> - Inference serves predictions via Vertex AI Endpoint
> - Model artifacts stored in Cloud Storage
> - Requirement: Separate permissions for training vs. inference; minimize blast radius
>
> **Question**: Which IAM assignment pattern BEST aligns with least privilege?
>
> A) Grant `roles/aiplatform.admin` to the team's service account for both training and inference  
> B) Training: `roles/aiplatform.user` + `roles/bigquery.dataViewer` + `roles/storage.objectCreator` | Inference: `roles/aiplatform.predictor` on endpoint only  
> C) Grant `roles/editor` to the training SA and `roles/viewer` to the inference SA  
> D) Use a single SA with `roles/aiplatform.user` and restrict via IAM Conditions to training hours  
>
> **Answer**: B  
> **Rationale**: Separates training (needs data read + model creation + storage write) from inference (only needs endpoint prediction). `aiplatform.predictor` is scoped to the endpoint, preventing model modification. Option A over-privileges inference; C violates least privilege; D doesn't separate pipeline stages. AI/ML IAM questions test stage-specific role separation.

### Question 3: Cross-Org CI/CD Federation
> **Scenario**: A startup uses GitHub Actions to deploy to Google Cloud. Requirements:
> - No service account keys stored in repositories
> - Deployments must be restricted to specific repos and branches
> - Must comply with SOC2 audit requirements
>
> **Question**: Which authentication strategy BEST meets these requirements?
>
> A) Create a GitHub secret containing a rotated service account JSON key  
> B) Use Workload Identity Federation with attribute mapping to restrict access by `attribute.repository` and `attribute.ref`  
> C) Run a self-hosted GitHub runner on GCE with attached service account  
> D) Use Cloud Build triggered by GitHub webhooks instead of GitHub Actions  
>
> **Answer**: B  
> **Rationale**: Workload Identity Federation enables keyless, auditable access from GitHub. Attribute mapping restricts access to specific repos/branches. All token exchanges are logged in Cloud Audit Logs. Option A violates keyless best practice; C ties identity to VM lifecycle, not repo; D avoids the question's constraint (GitHub Actions must be used).

### Question 4: Org-Wide Guardrails
> **Scenario**: A healthcare organization must enforce:
> - No GCS buckets can be publicly accessible
> - Service account key creation must be blocked org-wide
> - Existing policies at project level must not override these rules
>
> **Question**: Which combination of controls BEST enforces these requirements?
>
> A) Project-level IAM policies denying `allUsers` + Org Policy for key creation  
> B) IAM Deny Policy at org level blocking `storage.objects.get` for `allUsers` + `constraints/iam.disableServiceAccountKeyCreation`  
> C) VPC Service Controls perimeter + Cloud SCC alerts for public buckets  
> D) Custom IAM role removing public access + Cloud Scheduler key rotation job  
>
> **Answer**: B  
> **Rationale**: IAM Deny Policies are evaluated before allow policies and cannot be overridden at lower levels, enforcing org-wide guardrails. The Org Policy constraint blocks key creation natively. Option A can be overridden at project level; C detects but doesn't prevent; D is reactive and doesn't block creation. Deny policies + org policies = absolute enforcement.

### Question 5: Zero-Trust App Access
> **Scenario**: Employees need secure access to an internal analytics dashboard hosted on Cloud Run. Requirements:
> - Access must be blocked from unmanaged devices
> - Must verify corporate network or MDM compliance
> - No VPN client allowed
>
> **Question**: Which architecture BEST implements zero-trust access?
>
> A) Cloud IAP + Context-Aware Access with BeyondCorp access level requiring device posture compliance  
> B) Cloud Armor geo-blocking + API key authentication  
> C) VPC Service Controls perimeter + IAM conditions for corporate IP ranges  
> D) Cloud Endpoints with JWT validation + firewall rules for office IPs  
>
> **Answer**: A  
> **Rationale**: IAP verifies user identity; Context-Aware Access (BeyondCorp) evaluates device posture (MDM, OS version, encryption) before granting access. No VPN required. Option B lacks device verification; C restricts by IP but not device state; D relies on network trust, not zero-trust principles.

---

## ✅ How to Integrate This Addendum

1. **Append** the `Missing & High-Yield Topics` section after `10. Federated Authentication`.
2. **Insert** the `Advanced PCA Decision Frameworks` table before the `Quick Reference` section.
3. **Replace** any generic "exam traps" with the expanded `2026 Exam Traps & Anti-Patterns` table.
4. **Add** the `Scenario-Based Practice Questions` to your exam prep rotation. Time yourself: ~60-75 seconds per question.
5. **Cross-Reference** with your networking/security guides: PAM ↔ IAM Conditions, Deny Policies ↔ Org Policies, Federation ↔ CI/CD, BeyondCorp ↔ IAP/Zero-Trust.

---
*This enrichment aligns with the official Google Cloud PCA exam guide, 2024-2026 IAM GA announcements, and real candidate feedback. Always validate architecture decisions against current [GCP IAM Documentation](https://cloud.google.com/iam/docs).*
