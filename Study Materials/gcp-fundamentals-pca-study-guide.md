# ☁️ GCP Fundamentals — PCA Exam Study Guide
> **Target:** Google Cloud Professional Cloud Architect (PCA) Exam  
> **Domains:** GCP Core Concepts, Billing, Resource Hierarchy, Access Methods  
> **Topics Covered:** 11 Core Areas

---

## Table of Contents
1. [Google Cloud SDK and gcloud](#1-google-cloud-sdk-and-gcloud)
2. [Cloud Shell](#2-cloud-shell)
3. [Project Name, ID, and Number](#3-project-name-id-and-number)
4. [Governance and Compliance with Org Policies](#4-governance-and-compliance-with-org-policies)
5. [Org Level Roles](#5-org-level-roles)
6. [Cloud Billing — Introduction](#6-cloud-billing--introduction)
7. [Cloud Billing — Exports](#7-cloud-billing--exports)
8. [Cloud Billing — Resource Labels](#8-cloud-billing--resource-labels)
9. [Cloud Billing — Billing Account Access](#9-cloud-billing--billing-account-access)
10. [Accessing GCP — Console, SDK, and APIs](#10-accessing-gcp--console-sdk-and-apis)
11. [GCP Resource Hierarchy — Introduction](#11-gcp-resource-hierarchy--introduction)

---

## 1. Google Cloud SDK and gcloud

### What is the Google Cloud SDK?
The **Google Cloud SDK** is a set of tools for interacting with GCP services from the command line and from code. It is the primary developer and operator interface outside of the Console.

### SDK Components

| Component | Type | Purpose |
|-----------|------|---------|
| **gcloud** | CLI | Manage GCP resources: compute, IAM, GKE, storage, etc. |
| **gsutil** | CLI | GCS-specific operations (being replaced by `gcloud storage`) |
| **bq** | CLI | BigQuery operations: run queries, load data, manage datasets |
| **kubectl** | CLI (bundled) | Interact with Kubernetes/GKE clusters |
| **Cloud Client Libraries** | SDK Libraries | Programmatic access from Python, Java, Go, Node, etc. |
| **gcloud alpha/beta** | CLI | Preview features not yet GA |

> **Exam Note:** `gsutil` is being deprecated in favor of `gcloud storage`. For new architectures, use `gcloud storage`. For legacy references in exam questions, `gsutil` still appears.

### gcloud CLI — Core Structure

```
gcloud [GROUP] [SUB-GROUP] [COMMAND] [FLAGS] [POSITIONAL ARGS]

Examples:
gcloud compute instances list --project=my-project --zone=us-central1-a
gcloud container clusters get-credentials CLUSTER --region=northamerica-northeast1
gcloud iam service-accounts create my-sa --display-name="App SA"
gcloud storage cp file.txt gs://my-bucket/
```

### gcloud Configuration & Profiles

```bash
# View active config
gcloud config list

# Set core defaults
gcloud config set project PROJECT_ID
gcloud config set compute/region northamerica-northeast1
gcloud config set compute/zone northamerica-northeast1-a

# Named configurations (for multi-project/account work)
gcloud config configurations create dev-profile
gcloud config configurations activate dev-profile
gcloud config configurations list
```

> **Architecture Pattern:** Use named configurations to switch between dev/staging/prod projects cleanly without manually setting project flags on every command.

### gcloud Authentication

```bash
# Interactive login (human user)
gcloud auth login

# Application Default Credentials (for SDKs/code)
gcloud auth application-default login

# Impersonate a service account
gcloud auth print-access-token --impersonate-service-account=SA_EMAIL

# Use a service account key file (avoid in prod)
gcloud auth activate-service-account --key-file=key.json
```

### gcloud vs. gsutil vs. gcloud storage

| Operation | gsutil (legacy) | gcloud storage (current) |
|-----------|----------------|--------------------------|
| Copy file | `gsutil cp file.txt gs://bucket/` | `gcloud storage cp file.txt gs://bucket/` |
| List bucket | `gsutil ls gs://bucket/` | `gcloud storage ls gs://bucket/` |
| Set ACL | `gsutil acl set ...` | `gcloud storage buckets update --acl=...` |
| Parallel composite upload | `gsutil -m cp ...` | Built-in (automatic) |
| Rsync | `gsutil rsync ...` | `gcloud storage rsync ...` |

### SDK vs. Cloud Client Libraries

| | gcloud CLI | Cloud Client Libraries |
|--|-----------|----------------------|
| **Used by** | Operators, DevOps, scripts | Application developers |
| **Language** | Shell/CLI | Python, Java, Go, Node, etc. |
| **Auth** | gcloud auth / ADC | ADC (automatic) |
| **Use case** | Infrastructure management | App-level GCP integration |
| **Idempotency** | Manual (`--quiet`, scripting) | Built into library methods |

### Design Constraints Driving Choice

| Situation | Tool |
|-----------|------|
| One-off admin task | `gcloud` CLI |
| Bash automation/CI scripts | `gcloud` CLI with `--format=json` |
| Application reading from GCS | Cloud Client Library (Python/Go) |
| Large GCS file operations | `gcloud storage` with parallel upload |
| BigQuery from app | `google-cloud-bigquery` library |
| Kubernetes cluster management | `gcloud container` + `kubectl` |
| Infrastructure as Code | Terraform GCP provider (uses APIs under the hood) |

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Running `gcloud` without setting project | Always set `--project` or `gcloud config set project` |
| Using `gsutil` in new scripts | Migrate to `gcloud storage` |
| `gcloud auth login` in CI/CD | Use Workload Identity / SA impersonation instead |
| Mixing alpha/beta commands in prod automation | Use only GA commands in production pipelines |

---

## 2. Cloud Shell

### What is Cloud Shell?
Cloud Shell is a **browser-based, ephemeral Linux environment** (Debian) pre-authenticated with your Google identity, available in the GCP Console at no cost. It provides instant access to `gcloud`, `kubectl`, `terraform`, `git`, editors, and GCP APIs without any local setup.

### Core Capabilities

| Feature | Detail |
|---------|--------|
| **Free** | No charge; comes with every GCP account |
| **Pre-authenticated** | Runs as your Google identity automatically |
| **Persistent home dir** | 5 GB of `$HOME` storage persists across sessions |
| **Ephemeral VM** | e2-small Compute Engine VM, provisioned per session (up to 60 min idle timeout) |
| **Pre-installed tools** | gcloud, gsutil, kubectl, terraform, docker, git, python, node, java, go |
| **Web Preview** | Expose a local port (8080) to preview web apps in browser |
| **Cloud Shell Editor** | VS Code-based editor via `cloudshell edit .` |
| **Boost Mode** | Temporarily upgrades to e2-medium for heavier workloads |

### When Cloud Shell is Recommended

| Scenario | Cloud Shell? |
|----------|-------------|
| Quick admin tasks without local SDK setup | ✅ Yes |
| Learning / exploration / labs | ✅ Yes |
| Demo environments at conferences | ✅ Yes |
| Pre-authenticated gcloud in tutorials | ✅ Yes |
| Production automation / CI/CD pipelines | ❌ No — use Cloud Build or dedicated runners |
| Persistent long-running processes | ❌ No — 60-min idle timeout kills session |
| Heavy data processing | ❌ No — e2-small (2 vCPU, 1.7 GB RAM) |
| Custom OS/tooling requirements | ❌ No — use a Compute Engine VM |

### Cloud Shell vs. Local SDK vs. Cloud Build

| | Cloud Shell | Local gcloud SDK | Cloud Build |
|--|------------|-----------------|-------------|
| **Auth** | Auto (your identity) | Manual `gcloud auth login` | SA key or Workload Identity |
| **Setup time** | Zero | Installation required | Pipeline config required |
| **Persistence** | Session-based (VM), 5 GB home | Local machine | Ephemeral (per build) |
| **Use case** | Exploration, quick ops | Day-to-day dev/ops | CI/CD automation |
| **Cost** | Free | Free (SDK) | Pay per build-minute |

### Architecture Design Decisions

```
Need to run a one-off gcloud command?
    → Cloud Shell (no setup, pre-auth)

Need to automate infrastructure changes in CI/CD?
    → Cloud Build + Terraform (not Cloud Shell)

Need persistent tooling environment for a team?
    → Cloud Workstations (managed dev environments)

Need to preview a local web app on port 8080?
    → Cloud Shell Web Preview
```

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Relying on Cloud Shell for automation | Use Cloud Build, GitHub Actions + WIF instead |
| Storing critical files only in Cloud Shell VM | Store in `$HOME` (persisted 5 GB) or GCS |
| Forgetting 60-min idle timeout | Use `screen` or `tmux` for longer sessions (but still ephemeral) |
| Cloud Shell as a "always-on" bastion | Use Identity-Aware Proxy + Compute VM instead |

### Key Exam Points
- Cloud Shell VM is **not the same** as your `$HOME` directory — the VM is ephemeral, the home dir (5 GB) persists
- Cloud Shell is authenticated as **your user identity**, not a SA — avoid using it in automated pipelines
- Cloud Shell runs in **Google's infrastructure** — it has network access to GCP APIs by default
- For the exam, Cloud Shell = **exploration/learning tool**, not a production operations tool

---

## 3. Project Name, ID, and Number

### The Three Project Identifiers

Every GCP project has **three distinct identifiers**, each serving a different purpose:

| Identifier | Format | Set By | Mutable? | Used In |
|-----------|--------|--------|----------|---------|
| **Project Name** | Human-readable string, any chars | You | ✅ Yes | Console display only |
| **Project ID** | Globally unique, lowercase, 6–30 chars, hyphens allowed | You (or auto-generated) | ❌ No | gcloud commands, APIs, resource URLs |
| **Project Number** | Globally unique integer | Google (auto-assigned) | ❌ No | Internal GCP references, service agents, audit logs |

### Examples

```
Project Name:   "Digital Identity Platform - Production"
Project ID:     digital-identity-prod-3a9f
Project Number: 784512093847
```

### Where Each is Used

```bash
# Project ID — used in gcloud, Terraform, APIs
gcloud config set project digital-identity-prod-3a9f
gcloud compute instances list --project=digital-identity-prod-3a9f

# Project Number — appears in service account emails, API resource paths
# Default compute SA: 784512093847-compute@developer.gserviceaccount.com
# Resource name:  //cloudresourcemanager.googleapis.com/projects/784512093847

# Project Name — display in Console, search; not used in code
```

### Project ID Rules

```
- 6 to 30 characters
- Lowercase letters, digits, hyphens only
- Must start with a lowercase letter
- Cannot end with a hyphen
- Globally unique across ALL GCP projects (all customers)
- Cannot be reused after project deletion (even after 30-day lull period)
```

> ⚠️ **Critical Pitfall:** Once a Project ID is chosen, it **cannot be changed**. If you delete a project, the ID cannot be reclaimed — not even by you. Plan your ID naming convention carefully.

### Recommended Naming Conventions

```
Pattern: <org-prefix>-<service>-<environment>-<suffix>

Examples:
scotiabank-identity-prod-7f2a
scotiabank-payments-dev-3c1b
scotiabank-sharedvpc-prod-9e4d
scotiabank-ml-staging-2a8c
```

> Use a random suffix to ensure global uniqueness while maintaining readability.

### Project ID in Terraform

```hcl
resource "google_project" "identity_prod" {
  name            = "Digital Identity Platform - Production"
  project_id      = "scotiabank-identity-prod-7f2a"   # Cannot change after creation
  org_id          = var.org_id
  billing_account = var.billing_account_id
  folder_id       = google_folder.production.name
}
```

### Design Constraints

| Decision | Guidance |
|----------|---------|
| Project ID convention | Establish org-wide naming standard before creating any projects |
| Project per environment | Yes — separate project IDs per dev/staging/prod |
| Manual vs. auto-generated ID | Always specify manually — auto-generated IDs are ugly and meaningless |
| ID after deletion | Plan for sunset — ID is permanently gone after deletion |

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Using Project Name in scripts | Use Project ID — Name is mutable and not unique |
| Using Project Number in Terraform | Use Project ID for readability |
| Not planning ID naming upfront | Establish convention before first project |
| Expecting to reuse deleted project IDs | Not possible — use a fresh unique ID |

---

## 4. Governance and Compliance with Org Policies

### What are Org Policies?
Organization Policies are **centralized, hierarchical constraints** applied to GCP resources across your org, regardless of IAM permissions. They define **what configurations are allowed** — independently of who is doing the configuration.

> **Key distinction:** IAM controls *who* can act. Org Policies control *what* configurations are permitted.

### Core Components

| Component | Description |
|-----------|-------------|
| **Constraint** | A specific governance rule (e.g., restrict external IPs on VMs) |
| **Policy** | The application of a constraint at a node in the resource hierarchy |
| **Resource hierarchy node** | Organization, Folder, or Project where the policy is applied |
| **Enforcement** | Org Policy Service evaluates requests against policies before allowing |

### Constraint Types

| Type | Behavior | Example |
|------|---------|---------|
| **Boolean** | On/Off toggle | `constraints/compute.disableSerialPortAccess` |
| **List** | Allow or deny a set of values | `constraints/gcp.resourceLocations` |

### Commonly Tested Org Policy Constraints

| Constraint | What It Controls |
|-----------|----------------|
| `constraints/gcp.resourceLocations` | Restrict which regions resources can be created in |
| `constraints/iam.allowedPolicyMemberDomains` | Restrict IAM bindings to specific Cloud Identity domains |
| `constraints/iam.disableServiceAccountKeyCreation` | Prevent SA key downloads |
| `constraints/iam.disableServiceAccountKeyUpload` | Prevent uploading external SA keys |
| `constraints/compute.requireShieldedVm` | Require Shielded VM features on all VMs |
| `constraints/compute.vmExternalIpAccess` | Restrict or deny external IP assignment to VMs |
| `constraints/compute.skipDefaultNetworkCreation` | Prevent auto-creation of default VPC on new projects |
| `constraints/storage.uniformBucketLevelAccess` | Enforce uniform bucket IAM (disable object ACLs) |
| `constraints/compute.restrictCloudNATUsage` | Limit which projects/VPCs can use Cloud NAT |
| `constraints/run.allowedIngress` | Restrict Cloud Run ingress settings |

### Policy Inheritance and Enforcement

```
Organization (policy set here)
    → Inherited by all Folders
        → Inherited by all Projects
            → Inherited by all Resources

ALLOW_ALL → no restriction at this level (but parent may still restrict)
DENY_ALL  → block everything regardless of lower levels
```

- **Inheritance is downward.** A policy at Org level applies to everything beneath it.
- **Child nodes can further restrict**, but cannot **relax** a parent restriction (unless `inheritFromParent: false` + override, and override is permitted by the constraint).
- **`ALLOW` overrides are possible** only on specific constraints that support it via `restoreDefault`.

### Applying an Org Policy (gcloud)

```bash
# Boolean constraint — enforce at org level
gcloud org-policies enable-enforce \
  constraints/compute.skipDefaultNetworkCreation \
  --organization=ORG_ID

# List constraint — restrict resource locations to Canada
cat > policy.yaml << EOF
name: organizations/ORG_ID/policies/gcp.resourceLocations
spec:
  rules:
  - values:
      allowedValues:
      - in:northamerica-northeast1-locations
      - in:northamerica-northeast2-locations
EOF

gcloud org-policies set-policy policy.yaml
```

### Applying Per Folder (Environment-Specific Policies)

```bash
# Enforce Shielded VMs only in production folder
gcloud org-policies enable-enforce \
  constraints/compute.requireShieldedVm \
  --folder=PROD_FOLDER_ID

# Relax SA key restriction in dev folder (if parent allows override)
gcloud org-policies delete \
  constraints/iam.disableServiceAccountKeyCreation \
  --folder=DEV_FOLDER_ID
```

### Org Policy vs. IAM vs. VPC Firewall

| Mechanism | Controls | Applied At |
|-----------|---------|-----------|
| Org Policy | What resource configurations are allowed | Org/Folder/Project (preventive) |
| IAM | Who can perform which actions | Resource (access control) |
| VPC Firewall | Network traffic in/out | VPC/Subnet level |
| VPC-SC | API-level data perimeter | Project/resource group |

### Custom Org Policies (GA 2024)
You can now write custom constraints using **CEL expressions** against resource properties.

```yaml
# Custom constraint: Require all GKE clusters to have binary authorization enabled
name: organizations/ORG_ID/customConstraints/custom.gkeRequireBinaryAuth
resourceTypes:
  - container.googleapis.com/Cluster
methodTypes:
  - CREATE
  - UPDATE
condition: "resource.binaryAuthorization.evaluationMode == 'PROJECT_SINGLETON_POLICY_ENFORCE'"
actionType: ALLOW
displayName: "Require Binary Authorization on GKE Clusters"
```

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Confusing IAM deny with Org Policy | Org Policy = config guardrails; IAM Deny = access control |
| Setting restrictive policy at wrong hierarchy level | Test at project first; then promote to folder/org |
| Forgetting inheritance can't be upward | Child can restrict; cannot relax parent policy |
| Not auditing org policy violations | Use SCC + org policy findings + Cloud Audit Logs |

---

## 5. Org Level Roles

### What are Org-Level Roles?
Org-level roles are IAM roles granted at the **Organization node** of the resource hierarchy. Because of downward policy inheritance, these roles propagate to **all folders, projects, and resources** in the org.

> ⚠️ **Principle of Least Privilege:** Org-level role grants should be **minimal and carefully controlled**. Most roles should be granted at the project or folder level, not the org.

### Organization-Scoped Predefined Roles

| Role | Purpose | Who Should Have It |
|------|---------|-------------------|
| `roles/resourcemanager.organizationAdmin` | Manage org IAM policies, create folders | ≤3 break-glass admins |
| `roles/resourcemanager.folderAdmin` | Create/manage folders | Platform team leads |
| `roles/resourcemanager.projectCreator` | Create new projects | Automation SA, platform admins |
| `roles/billing.admin` | Manage billing accounts | Finance + cloud admins |
| `roles/billing.viewer` | View billing data | Finance teams |
| `roles/orgpolicy.policyAdmin` | Set org policies | Security/governance team |
| `roles/securitycenter.admin` | Manage SCC findings | Security operations |
| `roles/logging.admin` | Manage org-level logs and sinks | Central logging SA |
| `roles/iam.organizationRoleAdmin` | Create/manage custom roles at org level | Platform IAM admins |
| `roles/accesscontextmanager.policyAdmin` | Manage VPC-SC + access levels | Security architects |

### Org Admin vs. Project Owner

| | `roles/resourcemanager.organizationAdmin` | `roles/owner` (project) |
|--|------------------------------------------|------------------------|
| **Scope** | Entire organization | Single project |
| **Controls** | Org IAM, folder structure, billing link | All project resources + IAM |
| **Risk** | Extreme — controls all projects | High — full project control |
| **Who** | ≤3 break-glass identities (not daily-use accounts) | Project leads |

### Recommended Org IAM Architecture

```
Organization
    ├── roles/resourcemanager.organizationAdmin → group:org-admins@corp.com (≤3 members, MFA enforced)
    ├── roles/orgpolicy.policyAdmin             → group:security-governance@corp.com
    ├── roles/billing.admin                     → group:cloud-billing-admins@corp.com
    ├── roles/securitycenter.admin              → group:soc-team@corp.com
    └── roles/logging.admin                     → serviceAccount:log-export-sa@logging-project.iam.gserviceaccount.com

Folder: Production
    ├── roles/resourcemanager.folderAdmin       → group:platform-leads@corp.com
    └── roles/container.admin                   → group:prod-sre@corp.com

Folder: Development
    └── roles/editor                            → group:dev-team@corp.com (acceptable in dev only)

Project: identity-platform-prod
    └── roles/container.developer              → group:identity-devs@corp.com
```

### Super Admin vs. Org Admin

| | Google Workspace Super Admin | GCP Org Admin |
|--|------------------------------|---------------|
| **Controls** | Workspace users, domains, devices | GCP resource hierarchy, IAM, org policies |
| **GCP access** | Becomes Org Admin by default | Explicitly granted |
| **Risk** | Very high — can delegate everything | Very high — controls all GCP |
| **Best practice** | Dedicated account, not daily-use | Group with ≤3 members, hardware key MFA |

### Key Exam Points
- `roles/resourcemanager.organizationAdmin` does NOT automatically grant access to all resources — it controls the org IAM structure, not data
- `roles/owner` at the **project** level does not propagate upward to the org
- **Never assign** `roles/editor` or `roles/owner` at the org level — this would grant edit access to every project in the org
- Use **groups** for all org-level role bindings — never individual user accounts
- Enforce **MFA / hardware security keys** for org-level admin accounts

---

## 6. Cloud Billing — Introduction

### What is Cloud Billing?
Cloud Billing is the GCP service that **links financial responsibility** (payment method, budgets, invoicing) to GCP resource consumption. Every GCP resource must be linked to a billing account to incur charges.

### Core Components

| Component | Description |
|-----------|-------------|
| **Billing Account** | The financial entity that pays for GCP usage. Links payment method to projects |
| **Self-serve account** | Paid by credit/debit card. Immediate activation. Standard for most orgs |
| **Invoiced account** | Paid by invoice (monthly). Requires approval from Google. For large enterprises |
| **Billing Account ID** | Globally unique alphanumeric ID: `XXXXXX-XXXXXX-XXXXXX` |
| **Subaccounts** | Child billing accounts under a reseller/master account |
| **Budget & Alerts** | Thresholds that trigger notifications or actions when spend reaches a level |
| **Cost Table / Reports** | UI-based spend analysis |
| **Billing Export** | Streaming of detailed billing data to BigQuery for analysis |

### Billing Account — Project Relationship

```
Billing Account
    ├── Project A (linked)
    ├── Project B (linked)
    └── Project C (linked)
```

- **One project → one billing account** (at a time)
- **One billing account → many projects**
- A project with **no linked billing account** cannot create paid resources
- Projects can be **re-linked** to a different billing account (requires `billing.resourceAssociations.create`)

### Billing Hierarchy

```
Organization
    └── Billing Account (linked to Org)
            └── Projects (linked to Billing Account)
```

- Billing accounts can exist **outside an org** (self-serve) or be **associated with an org** (enterprise control)
- Org-associated billing accounts allow org admins to enforce which projects can use which billing accounts

### Key Billing Use Cases

| Use Case | Solution |
|---------|---------|
| Separate charges by business unit | Multiple billing accounts per BU, or projects with labels |
| Alert when spend exceeds $X | Budget Alerts (email, Pub/Sub) |
| Analyze spend by team/env/service | Billing Export to BigQuery + labels |
| Prevent unexpected charges | Budget + org policy to disable billing if needed |
| Reseller managing client billing | Subaccounts under master billing account |

### Budget Alerts

```bash
# Create a budget with 50%, 90%, 100% alert thresholds
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Monthly Prod Budget" \
  --budget-amount=10000USD \
  --threshold-rule=percent=0.5 \
  --threshold-rule=percent=0.9 \
  --threshold-rule=percent=1.0 \
  --notifications-rule-pubsub-topic=projects/PROJECT/topics/billing-alerts
```

> **Exam Note:** Budget alerts do **NOT automatically disable or stop resources**. They only trigger notifications. To auto-stop resources, connect the Pub/Sub topic to a Cloud Function that disables billing programmatically.

### Disabling Billing Programmatically

```python
# Cloud Function triggered by Pub/Sub budget alert
from googleapiclient import discovery

def disable_billing(project_id):
    billing = discovery.build('cloudbilling', 'v1')
    billing.projects().updateBillingInfo(
        name=f'projects/{project_id}',
        body={'billingAccountName': ''}  # Empty = unlink billing
    ).execute()
```

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Assuming budget alerts stop resources | They don't — implement Pub/Sub + Cloud Function for auto-disable |
| Single billing account for all teams | Use labels + exports for chargeback, or separate accounts per BU |
| Not linking billing account during project creation | Resources fail silently or can't be created |
| Billing account outside org | Lose org-level governance controls — link to org |

---

## 7. Cloud Billing — Exports

### What is Billing Export?
Billing Export automatically sends detailed billing and cost data to **BigQuery** (recommended) or **Cloud Storage** (legacy CSV/JSON). It enables powerful cost analysis, chargeback, and forecasting.

### Export Types

| Export Type | Destination | Granularity | Use Case |
|------------|-------------|-------------|---------|
| **Standard Usage Cost** | BigQuery | Daily | General cost analysis, chargeback |
| **Detailed Usage Cost** | BigQuery | Hourly + resource-level | Deep SKU/resource analysis |
| **Pricing Export** | BigQuery | Current pricing | Cost modeling, forecasting |
| **Legacy CSV/JSON** | Cloud Storage | Daily | Simple reports (avoid for new setups) |

> **Exam Rule:** Always recommend **BigQuery export** over Cloud Storage export. BQ allows SQL querying, dashboards, and joins with other datasets.

### Setting Up BigQuery Export

```bash
# Enable billing export via Console:
# Billing → Billing Export → BigQuery Export → Edit Settings

# The dataset must be in a project linked to the billing account
# Dataset should be in the same region as your analysis tools

# After setup, Google auto-creates these tables:
# gcp_billing_export_v1_XXXXXX_XXXXXX_XXXXXX (standard)
# gcp_billing_export_resource_v1_XXXXXX_... (detailed)
```

### BigQuery Billing Table Schema (Key Fields)

| Field | Description |
|-------|-------------|
| `billing_account_id` | Billing account identifier |
| `service.description` | GCP service (Compute Engine, GKE, etc.) |
| `sku.description` | Specific SKU (N1 Predefined Instance Core) |
| `usage_start_time` | When usage began |
| `project.id` | Project that incurred cost |
| `project.labels` | Project-level labels |
| `labels` | Resource-level labels |
| `location.region` | Region of resource |
| `cost` | Net cost after discounts |
| `credits` | Committed Use Discounts, Sustained Use Discounts, promotions |
| `usage.amount` | Amount of resource consumed |
| `resource.name` | Specific resource name (detailed export only) |

### Sample Billing Queries

```sql
-- Total spend by project last 30 days
SELECT
  project.id,
  SUM(cost) AS total_cost
FROM `project.dataset.gcp_billing_export_v1_*`
WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY project.id
ORDER BY total_cost DESC;

-- Cost by service and environment label
SELECT
  service.description,
  (SELECT value FROM UNNEST(labels) WHERE key = 'environment') AS environment,
  SUM(cost) AS total_cost
FROM `project.dataset.gcp_billing_export_v1_*`
WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY 1, 2
ORDER BY total_cost DESC;

-- Credits applied (CUDs, SUDs)
SELECT
  project.id,
  credit.type,
  SUM(credit.amount) AS total_credits
FROM `project.dataset.gcp_billing_export_v1_*`,
  UNNEST(credits) AS credit
GROUP BY 1, 2;
```

### Export Timing

| Export Type | Data Availability |
|-------------|------------------|
| Standard | Data appears within 24 hours of usage |
| Detailed | Data appears within several hours |
| Historical | Only available from the date export was enabled — no retroactive data |

> ⚠️ **Exam Pitfall:** Billing export is **not retroactive**. Enable it immediately when setting up a new billing account or you lose historical data.

### Architecture: Cost Analysis Pipeline

```
GCP Resources (all projects under billing account)
    ↓ (automatic, daily)
BigQuery Dataset (billing export)
    ↓
Looker Studio / Looker dashboards (cost visualization)
    +
Scheduled BigQuery queries → Pub/Sub → Alerts (anomaly detection)
    +
BigQuery → Sheets (for finance chargeback reports)
```

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Not enabling export when account is created | Enable immediately — no retroactive data |
| Using Cloud Storage (CSV) export | Use BigQuery for all new setups |
| Querying without partition filter | Always filter on `_PARTITIONTIME` to reduce cost |
| Forgetting credits in cost calculations | Include `credits` array in queries for net cost |

---

## 8. Cloud Billing — Resource Labels

### What are Resource Labels?
Labels are **key-value metadata pairs** attached to GCP resources, used for **organization, cost allocation, filtering, and automation**. They appear in billing export data and enable fine-grained cost attribution.

### Label Format

```
key: value

Rules:
- Keys: 1–63 chars, lowercase letters, digits, hyphens, underscores. Must start with a letter.
- Values: 0–63 chars, lowercase letters, digits, hyphens, underscores. Can be empty.
- Max 64 labels per resource
- Keys must be unique per resource

Examples:
environment: prod
team: identity-platform
cost-center: cc-1042
app: payment-gateway
managed-by: terraform
```

### Recommended Label Taxonomy

| Key | Example Values | Purpose |
|-----|---------------|---------|
| `environment` | `prod`, `staging`, `dev` | Cost by env |
| `team` | `platform`, `identity`, `payments` | Cost by team |
| `cost-center` | `cc-1042`, `cc-2088` | Finance chargeback |
| `app` | `payment-gateway`, `auth-service` | Cost by application |
| `managed-by` | `terraform`, `manual` | Governance tracking |
| `owner` | `nilanjan-das` | Individual accountability |
| `data-classification` | `public`, `confidential`, `restricted` | Security automation |

### Applying Labels

```bash
# Label a GCE instance
gcloud compute instances add-labels INSTANCE_NAME \
  --labels=environment=prod,team=identity,cost-center=cc-1042 \
  --zone=northamerica-northeast1-a

# Label a GCS bucket
gcloud storage buckets update gs://my-bucket \
  --update-labels=environment=prod,app=data-pipeline

# Label a GKE cluster
gcloud container clusters update CLUSTER_NAME \
  --update-labels=environment=prod,team=platform \
  --region=northamerica-northeast1
```

### Labels in Terraform (Enforced via Module)

```hcl
variable "mandatory_labels" {
  type = map(string)
  default = {
    environment = "prod"
    team        = "identity-platform"
    cost-center = "cc-1042"
    managed-by  = "terraform"
  }
}

resource "google_compute_instance" "app_vm" {
  name   = "app-vm"
  labels = var.mandatory_labels
  # ...
}
```

### Labels vs. Tags vs. Network Tags

| | Labels | Resource Tags (IAM) | Network Tags |
|--|--------|--------------------|----|
| **Purpose** | Cost, filtering, automation | IAM Conditions, Org Policy | Firewall rules |
| **Scope** | Most GCP resources | Resource hierarchy-aware | GCE instances only |
| **Used in billing?** | ✅ Yes | ❌ No | ❌ No |
| **Used in IAM?** | ❌ No | ✅ Yes (conditions) | ❌ No |
| **Used in Firewall?** | ❌ No | ❌ No | ✅ Yes |

### Enforcing Labels via Org Policy (Custom Constraint)

```yaml
# Require 'environment' label on all Compute Engine instances
name: organizations/ORG_ID/customConstraints/custom.requireEnvLabel
resourceTypes:
  - compute.googleapis.com/Instance
methodTypes:
  - CREATE
condition: "'environment' in resource.labels"
actionType: ALLOW
displayName: "Require environment label on all VMs"
```

### Labels in Billing Queries

```sql
-- Cost breakdown by team label
SELECT
  (SELECT value FROM UNNEST(labels) WHERE key = 'team') AS team,
  SUM(cost) AS total_cost
FROM `billing_project.dataset.gcp_billing_export_v1_*`
WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY team
ORDER BY total_cost DESC;
```

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Inconsistent label values (`Prod` vs `prod` vs `production`) | Enforce via Terraform variables and custom org policy |
| Labels not propagated to child resources | Labels are per-resource — apply at provisioning time via IaC |
| Unlabeled legacy resources | Use Cloud Asset Inventory to find, then remediate via script |
| Confusing labels with network tags | Labels = metadata/billing; Network tags = firewall rules |
| Not labeling before billing export is queried | Labels must exist on resource at time of billing event |

---

## 9. Cloud Billing — Billing Account Access

### Billing IAM: Separate from Resource IAM
Billing access is controlled by **IAM roles on the billing account itself** — distinct from project-level IAM. Someone can have editor access to a project but no visibility into billing, and vice versa.

### Billing IAM Roles

| Role | Scope | Capabilities |
|------|-------|-------------|
| `roles/billing.admin` | Billing Account | Full control: link/unlink projects, manage payment, view costs, export |
| `roles/billing.viewer` | Billing Account | View costs and transactions only |
| `roles/billing.user` | Billing Account | Link billing account to projects |
| `roles/billing.projectManager` | Billing Account | Link/unlink projects to billing account |
| `roles/billing.costsManager` | Billing Account | Manage budgets, view costs — not payment info |

### Who Needs What

| Persona | Role | Why |
|---------|------|-----|
| Cloud Finance Admin | `billing.admin` | Full billing control |
| Cloud Platform Engineer | `billing.user` | Can link new projects to billing |
| Finance Analyst | `billing.viewer` | Read-only cost visibility |
| Budget Owner | `billing.costsManager` | Can set budgets, can't change payment |
| Project Creator SA | `billing.user` | Required to link project to billing at creation time |

### Granting Billing IAM

```bash
# Grant billing viewer to finance team group
gcloud billing accounts add-iam-policy-binding BILLING_ACCOUNT_ID \
  --member="group:finance-team@corp.com" \
  --role="roles/billing.viewer"

# Grant billing user to platform automation SA
gcloud billing accounts add-iam-policy-binding BILLING_ACCOUNT_ID \
  --member="serviceAccount:project-factory-sa@admin-project.iam.gserviceaccount.com" \
  --role="roles/billing.user"
```

### Linking a Project to a Billing Account

```bash
# Link project to billing account (requires billing.user or billing.admin)
gcloud billing projects link PROJECT_ID \
  --billing-account=BILLING_ACCOUNT_ID

# Unlink (stops billing — resources still exist but cannot create paid resources)
gcloud billing projects unlink PROJECT_ID
```

### Billing Account Hierarchy in Enterprise

```
Google (invoiced billing master)
    └── Master Billing Account (Scotiabank)
            ├── Subaccount: Digital Banking BU
            │       ├── Project: identity-prod
            │       └── Project: payments-prod
            ├── Subaccount: Retail Banking BU
            │       └── Project: retail-prod
            └── Subaccount: Shared Services
                    └── Project: shared-vpc-prod
```

- **Subaccounts** let business units see and manage their own spend independently
- The master account admin can see all subaccounts
- Subaccounts still receive a **consolidated invoice** at the master level

### Separation of Duties: Billing vs. IAM

| Concern | Controlled By |
|---------|--------------|
| Who can create resources | Project IAM |
| Who can see costs | Billing Account IAM |
| Who can link/unlink billing | Billing Account IAM (`billing.user`) |
| Who can set budgets | `billing.costsManager` or `billing.admin` |
| Who can disable billing (stop charges) | `billing.admin` |

> **Exam Scenario:** A developer has `roles/owner` on a project but cannot see billing costs. Why? Because billing visibility requires `roles/billing.viewer` on the **billing account**, not the project.

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Granting `billing.admin` to all admins | Restrict to ≤3 finance+cloud admins |
| Project owners assuming they can manage billing | Billing IAM is separate — grant `billing.viewer` explicitly |
| No billing access for developers | Grant `billing.viewer` at billing account level for cost awareness |
| Automation SA missing `billing.user` | Project factory patterns fail to link billing — add this role |
| Forgetting that unlinking billing stops paid resource creation | Unlink only for shutdown, not cost control (use budgets instead) |

---

## 10. Accessing GCP — Console, SDK, and APIs

### Three Primary Access Methods

| Method | Interface | Auth | Best For |
|--------|-----------|------|---------|
| **Cloud Console** | Web browser GUI | Google account (OIDC) | Exploration, monitoring, one-off changes |
| **Cloud SDK (gcloud)** | CLI / shell | gcloud auth / ADC | Scripting, DevOps, automation |
| **GCP REST/gRPC APIs** | HTTP/gRPC | OAuth 2.0 access token | Programmatic app integration |

### Cloud Console

```
URL: https://console.cloud.google.com

Key Features:
- Project switcher (top bar)
- Resource-specific UIs (GKE, GCS, BigQuery, etc.)
- Cloud Shell (built-in terminal)
- IAM & Admin UI
- Billing dashboards
- Audit logs viewer
- APIs & Services enablement
- Marketplace
```

**Design Constraints:**
- Not suitable for automation or CI/CD
- Good for initial setup, visualizing resources, debugging
- Changes in Console are reflected in APIs and vice versa (same underlying API)
- Multi-region operations are more visible in Console (though CLI is faster)

### Cloud SDK (gcloud) — Recap in Access Context

```bash
# Auth for human user
gcloud auth login

# Auth for local development (ADC)
gcloud auth application-default login

# Output formats for scripting
gcloud compute instances list --format=json
gcloud compute instances list --format="table(name,status,zone)"
gcloud compute instances list --format="value(name)"

# Quiet mode for automation (no prompts)
gcloud compute instances delete INSTANCE --quiet

# Filter results
gcloud compute instances list --filter="status=RUNNING AND zone:us-central1"
```

### GCP REST APIs

All GCP services expose **REST APIs** (and many expose gRPC APIs). The Console and gcloud both use these APIs internally.

```
Base URL pattern: https://SERVICE.googleapis.com/VERSION/RESOURCE

Examples:
https://compute.googleapis.com/compute/v1/projects/PROJECT/zones/ZONE/instances
https://container.googleapis.com/v1/projects/PROJECT/locations/REGION/clusters
https://iam.googleapis.com/v1/projects/PROJECT/serviceAccounts
```

**Calling APIs directly:**
```bash
# Get an access token
ACCESS_TOKEN=$(gcloud auth print-access-token)

# Call Compute Engine API
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://compute.googleapis.com/compute/v1/projects/PROJECT_ID/zones/ZONE/instances"
```

### API Enablement
GCP APIs must be **explicitly enabled per project** before use.

```bash
# Enable APIs
gcloud services enable container.googleapis.com
gcloud services enable iam.googleapis.com
gcloud services enable storage.googleapis.com
gcloud services enable bigquery.googleapis.com

# List enabled APIs
gcloud services list --enabled

# Disable an API (careful — may break services)
gcloud services disable container.googleapis.com
```

### Comparing Access Methods

| Dimension | Console | gcloud CLI | REST API |
|-----------|---------|-----------|---------|
| **Auth** | Browser OAuth | gcloud auth | Bearer token |
| **Auditability** | Cloud Audit Logs | Cloud Audit Logs | Cloud Audit Logs |
| **Automation** | ❌ Not suitable | ✅ Scriptable | ✅ Full programmatic |
| **Discoverability** | ✅ Excellent UI | Good with `--help` | Requires API docs |
| **Speed (bulk ops)** | Slow | Fast | Fastest |
| **Learning curve** | Low | Medium | High |
| **CI/CD** | ❌ No | ✅ Yes | ✅ Yes |

### API Versions: v1 vs. alpha vs. beta

| Version | Stability | Use In |
|---------|----------|--------|
| `v1` | GA — stable | Production |
| `beta` | Feature-complete, not final | Staging / early adoption |
| `alpha` | Experimental | Development / preview only |

### Cloud Console vs. gcloud: Design Choice

```
Choose Cloud Console when:
    - First-time exploration of a new service
    - Debugging an issue visually
    - One-off configuration you won't repeat
    - Non-technical stakeholders need access

Choose gcloud when:
    - Repeatable operations (scripted)
    - CI/CD integration
    - Bulk operations across multiple resources
    - Output needs to be parsed or piped

Choose REST API when:
    - Application code needs to call GCP dynamically
    - Using a language without a Cloud Client Library
    - Building a custom dashboard or tool
    - Need fine-grained API control beyond SDK capabilities
```

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Making prod changes via Console without audit trail awareness | All Console actions ARE logged in Cloud Audit Logs |
| Forgetting to enable APIs before Terraform apply | Add `google_project_service` resources in Terraform |
| Using `gcloud auth login` in pipelines | Use Workload Identity / SA impersonation |
| REST API calls without checking API enablement | Enable APIs first; 403 errors are often just disabled APIs |

---

## 11. GCP Resource Hierarchy — Introduction

### What is the GCP Resource Hierarchy?
The GCP Resource Hierarchy is a **tree-structured organizational model** that governs resource ownership, access control, policy inheritance, and billing. It mirrors how enterprises structure their teams and responsibilities.

### Hierarchy Levels

```
Google (root, implicit)
    └── Organization (1 per Cloud Identity / Workspace domain)
            └── Folder (optional, multi-level)
                    └── Project
                            └── Resources (VMs, GCS buckets, GKE clusters, etc.)
```

### Each Level Explained

| Level | Description | IAM / Policy Behavior |
|-------|-------------|----------------------|
| **Organization** | Root node for an enterprise. 1:1 with Cloud Identity domain | Policies here apply to all below |
| **Folder** | Logical grouping (BU, environment, team). Can nest up to 10 levels | Policies inherit down; can restrict further |
| **Project** | Primary isolation unit. All resources belong to a project | Resources inherit project + ancestor policies |
| **Resource** | Individual GCP service instance (VM, bucket, dataset) | Inherits project + folder + org policies |

### Why the Hierarchy Matters

```
1. Policy Inheritance — IAM roles and Org Policies flow downward
2. Billing — each project links to one billing account
3. Quota — quotas are set at the project level
4. API Enablement — per project
5. Isolation — project boundary is the primary security boundary
6. Blast Radius — compromise of one project doesn't affect others
```

### Organization Node

- Created automatically when you set up **Cloud Identity** or **Google Workspace**
- The org is identified by your domain (e.g., `scotiabank.com`)
- All projects created by identities in the org belong to the org by default (if configured)
- **No org?** Projects are personal, not governable at enterprise scale

```bash
# Get your org ID
gcloud organizations list

# Describe org
gcloud organizations describe ORG_ID
```

### Folders — Design Patterns

**Pattern 1: By Environment**
```
Org
├── Folder: Production
├── Folder: Staging
└── Folder: Development
```

**Pattern 2: By Business Unit**
```
Org
├── Folder: Digital Banking
├── Folder: Retail Banking
└── Folder: Shared Services
```

**Pattern 3: Hybrid (Recommended for Large Orgs)**
```
Org
├── Folder: Digital Banking
│       ├── Folder: Production
│       ├── Folder: Staging
│       └── Folder: Development
├── Folder: Shared Services
│       ├── Folder: Production
│       └── Folder: Development
└── Folder: Security & Governance
```

### Projects — The Core Isolation Unit

- Every GCP resource belongs to exactly **one project**
- Projects are the boundary for: billing, quotas, APIs, IAM, VPC networks
- You can have up to **unlimited projects** per org (soft quota, can be increased)
- Default project quota: **50 projects per billing account** (increaseable)

```bash
# Create a project in a folder
gcloud projects create PROJECT_ID \
  --name="Digital Identity - Production" \
  --folder=FOLDER_ID

# Link to billing
gcloud billing projects link PROJECT_ID \
  --billing-account=BILLING_ACCOUNT_ID

# Move project to a different folder
gcloud beta projects move PROJECT_ID \
  --folder=NEW_FOLDER_ID
```

### Resource Hierarchy and VPC

```
Project A (prod)          Project B (dev)
    └── VPC: prod-vpc          └── VPC: dev-vpc

→ By default, VPCs are project-scoped
→ Shared VPC allows one host project's VPC to be shared with service projects
→ VPC Peering allows connectivity between project VPCs

Shared VPC pattern:
    shared-vpc-host-prod (host project)
        └── VPC: prod-shared-vpc
                ├── Project: identity-prod (service project)
                └── Project: payments-prod (service project)
```

### Policy Inheritance Deep Dive

```
Org Policy: restrict to northamerica regions (gcp.resourceLocations)
    ↓ Inherited
Folder: Production
    ↓ Inherited (and can further restrict)
Project: identity-platform-prod
    → ALL resources in this project can ONLY be in northamerica regions
    → Even if a user has roles/compute.admin, they cannot create in us-east1
```

**Critical rules:**
1. Policies flow **downward only**
2. Child can **restrict further** but not **relax** parent policy
3. IAM effective policy = **union** of all ancestor policies
4. Org Policy effective policy = **intersection** (most restrictive wins)

### Hierarchy vs. Alternative Isolation Approaches

| Approach | Isolation Level | Use Case |
|----------|----------------|---------|
| Separate Organizations | Maximum | Separate companies / M&A |
| Separate Folders | BU/Env level | Standard enterprise pattern |
| Separate Projects | Workload/app level | Standard minimum isolation |
| GKE Namespaces | App-level within cluster | Multi-tenancy within GKE |
| VPC Subnets | Network-level within project | Network segmentation |

> **Exam Rule:** The recommended GCP isolation unit is the **Project**. For enterprise governance, use **Folders** within a single **Organization**.

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| All workloads in one project | One project per application/environment minimum |
| No Organization node | Set up Cloud Identity — enables org-level governance |
| Folders too deep (>10 levels) | GCP max is 10 folder levels — keep to 3-4 for manageability |
| Projects outside an org | No org-level policy control; billing governance lost |
| Moving a project between orgs | Not natively supported — requires migration steps |

---

## Quick Reference: GCP Fundamentals Decision Framework

```
Q: Which identifier do I use to reference a project in scripts?
    → Project ID (not Name, not Number)

Q: Where should I run one-off admin commands?
    → Cloud Shell (zero setup, pre-auth) or local gcloud

Q: How should CI/CD authenticate to GCP?
    → Workload Identity Federation (GitHub) or Workload Identity (GKE)
    → Never: gcloud auth login or SA key files

Q: How should I structure my GCP org for an enterprise bank?
    → Org → BU Folders → Environment Sub-folders → One project per app/env

Q: How do I prevent VMs from being created in US regions?
    → Org Policy: constraints/gcp.resourceLocations (restrict to Canada regions)

Q: Who should have org-level IAM roles?
    → Minimal: ≤3 org admins, security team, billing admins — all via groups

Q: How do I allocate costs to teams?
    → Labels + Billing Export to BigQuery → team/cost-center label in queries

Q: Budget alert fired — will resources be stopped?
    → No — alerts are notifications only. Need Pub/Sub + Cloud Function to act.

Q: Can a project belong to multiple billing accounts?
    → No — one billing account at a time per project

Q: How do I enforce label requirements on all new resources?
    → Custom Org Policy Constraint with CEL condition checking for required label keys
```

---

## Frequently Tested Exam Scenarios

| Scenario | Correct Answer |
|----------|---------------|
| Engineer needs to run gcloud without installing SDK | Cloud Shell |
| Production automation needs to call GCP APIs | REST API with SA + Workload Identity |
| Need to restrict GCP resource creation to Canada only | Org Policy: `constraints/gcp.resourceLocations` |
| New project keeps getting default VPC | Org Policy: `constraints/compute.skipDefaultNetworkCreation` |
| Developer can't see billing costs despite project owner role | Grant `roles/billing.viewer` on the billing account |
| Billing spike detected — how to auto-stop resources | Budget alert → Pub/Sub → Cloud Function to unlink billing |
| Need to attribute costs to individual teams | Resource labels + BigQuery billing export |
| Billing export was set up last week — need 6 months of data | Not available — export is not retroactive |
| Prevent SA key creation across entire org | Org Policy: `constraints/iam.disableServiceAccountKeyCreation` |
| Cloud Shell session died mid-operation | Normal behavior — 60-min idle timeout. Move to Cloud Build for automation |
| Project created with wrong ID — need to rename | Project ID is immutable — delete and recreate with correct ID |
| How to give GitHub Actions access to deploy to Cloud Run | Workload Identity Federation — no SA key |
| Multiple teams need isolated billing visibility | Subaccounts per BU under master billing account |
| Query billing data without partition filter | Always filter `_PARTITIONTIME` to avoid full table scan costs |

---

# 📘 PCA Fundamentals Enrichment Addendum (2026 Exam Focus)
> *Append this to your existing `gcp-fundamentals-pca-study-guide.md`. All sections are mapped to PCA scoring domains, include 2026 exam updates, and highlight high-frequency test patterns.*

---

## 🔍 Missing & High-Yield Topics for PCA 2026

### 1. Tags vs. Labels vs. Network Tags — *Critical Distinction*
| Feature | Purpose | Scope | Exam Use |
|---------|---------|-------|----------|
| **Labels** | Cost allocation, filtering, automation | Key-value pairs on most resources | Billing export, resource grouping, Org Policy constraints (custom CEL) |
| **Tags** | IAM conditions, policy binding | Hierarchical (org/folder/project/resource) | `resource.matchTag()` in IAM Conditions, dynamic policy application |
| **Network Tags** | Firewall rule targeting | GCE VMs only | `--target-tags` in firewall rules |

🎯 **Exam Decision Tree**: 
```
Need to allocate costs to finance? → Labels
Need to grant IAM access based on metadata? → Tags + IAM Conditions
Need to open firewall port 8080 to a VM group? → Network Tags
```

### 2. Committed Use Discounts (CUDs) & Billing Mechanics — *SUDs Deprecated*
| Concept | 2026 Exam Reality |
|---------|------------------|
| **Sustained Use Discounts (SUDs)** | ❌ Deprecated (2023). Exam questions referencing SUDs are outdated. |
| **Compute CUDs** | 1 or 3-year commitment for VMs. Auto-apply or project-specific. |
| **Storage/Memory CUDs** | Flexible commitments for PD, GCS, BigQuery slots. |
| **Billing Export `credits` field** | CUD savings appear here. Exam tests net vs gross cost queries. |
| **Auto-Apply vs. Specific** | Auto-apply maximizes utilization across org; specific ties to project. |

💡 **Exam Tip**: Questions about "predictable billing for steady-state workloads" → **Compute CUDs**. Questions about "flexible commitment for variable storage" → **Storage/Memory CUDs**.

### 3. Custom Org Policy Constraints (CEL) — *Compliance-as-Code*
| Feature | Exam Focus | PCA Catch |
|--------|------------|-----------|
| **CEL Expressions** | Write custom validation logic (e.g., require labels, restrict zones) | Replaces manual auditing; evaluated at resource creation/update |
| **Constraint Definition** | YAML with `condition`, `resourceTypes`, `methodTypes` | Test in staging; custom constraints bypass standard constraint validation |
| **Integration** | Cloud Build + Terraform + CI/CD gates | Fail fast: block non-compliant deployments before they reach prod |

⚠️ **Exam Trap**: Custom constraints evaluate **resource properties**, not IAM identities. If a question asks "prevent users from creating VMs without `env` label", answer: Custom Org Policy Constraint checking `resource.labels.env`.

### 4. Cloud Asset Inventory & Resource Manager APIs
| Feature | Exam Focus | PCA Catch |
|--------|------------|-----------|
| **Asset Discovery** | `cloudasset.googleapis.com` for org-wide resource/policy tracking | Used for compliance audits, drift detection, IAM cleanup |
| **Policy Search** | `SearchAllIamPolicies()` to find "who has access to X?" | Replaces manual console checks; required for large-org governance |
| **History Tracking** | Track resource/policy changes over time | Answers "when was this bucket made public?" or "who changed this route?" |

🎯 **Exam Catch**: When a question mentions "audit all IAM bindings across 200 projects" or "track configuration drift", the answer involves **Cloud Asset Inventory** + **BigQuery export** or **Security Command Center**.

### 5. Project Quota Management & IaC Integration
| Concept | Exam Reality |
|---------|--------------|
| **Quota Scope** | Per-project, per-region, per-service |
| **Monitoring** | `serviceruntime.googleapis.com/quota/exceeded` metric |
| **Automation** | Quota increase requests via API/Console; can be scripted with Cloud Build + Service Account |
| **IaC Pattern** | Terraform `google_compute_reservation` or preemptive quota requests to avoid `QUOTA_EXCEEDED` in CI/CD |

💡 **Exam Tip**: Questions about "Terraform pipeline failing due to insufficient IPs/CPUs" → **Request quota increase via Cloud Console/API before execution**, implement monitoring alerts, use reservation or flexible scheduling.

---

## 🧩 Advanced PCA Decision Frameworks

### 🛡️ Governance Enforcement Matrix
| Requirement | Primary Control | Secondary Control | Exam Answer Pattern |
|-------------|----------------|-------------------|---------------------|
| Prevent public GCS buckets | Org Policy (`storage.uniformBucketLevelAccess`) + IAM Deny | VPC-SC + SCC alerts | Preventive config > reactive detection |
| Allocate costs to business units | Resource Labels + Billing Export → BigQuery | Cost Table + Looker Studio | Labels must exist at billing event time |
| Restrict VM creation to Canada | Org Policy: `constraints/gcp.resourceLocations` | Folder-level policy override | Parent restricts; child cannot relax |
| Grant access based on `env:prod` | Tags + IAM Conditions (`resource.matchTag()`) | VPC-SC for API perimeter | Tags enable dynamic, context-aware IAM |
| Auto-rotate credentials | Workload Identity / WIF + Secret Manager | IAM Conditions (time-bound) | Keyless > key-based > manual rotation |

### 💰 Billing & Cost Allocation Architecture
```
GCP Resources (all projects under billing account)
    ↓ (automatic, daily)
BigQuery Billing Export (filter by `_PARTITIONTIME`)
    ↓
Cost Analysis Pipeline:
    • Team/Env attribution → `labels` array in queries
    • CUD tracking → `credits.type == 'COMMITTED_USE_DISCOUNT'`
    • Anomaly detection → Cloud Monitoring + Pub/Sub alerts
    • Finance chargeback → Looker Studio / Sheets automation

Exam Rule: Labels must be applied BEFORE the billing event. Retroactive labeling does NOT fix historical cost attribution.
```

### 🔑 Access & Identity Flow (2026 Patterns)
```
Human User → Workforce Identity Federation (Okta/Azure AD) → BeyondCorp Access → IAP → GCP APIs
CI/CD Pipeline → Workload Identity Federation (GitHub/GitLab) → STS → GSA Impersonation → Deploy
GKE Pod → Workload Identity (KSA → GSA binding) → ADC → GCP Services
On-Prem App → WIF + Service Account Key (legacy fallback) → Secret Manager → Rotate quarterly

Exam Trap: Cloud Shell is authenticated as YOUR user identity. NEVER use it in production CI/CD. Use WIF + Cloud Build.
```

---

## ⚠️ 2026 Exam Traps & Anti-Patterns

| Trap in Question | Correct Approach | Why |
|------------------|------------------|-----|
| "Apply labels retroactively to fix billing attribution" | Not possible — labels only apply forward | Billing events are timestamped; historical data lacks label context |
| "Use Sustained Use Discounts for new architecture" | Use Committed Use Discounts (CUDs) | SUDs deprecated; CUDs are the current mechanism |
| "Grant `roles/billing.admin` to all project owners" | Grant `roles/billing.viewer` or `billing.projectManager` | Violates least privilege; billing roles are separate from resource IAM |
| "Use labels in IAM Conditions for dynamic access" | Use Tags + `resource.matchTag()` | Labels ≠ Tags. Tags are hierarchical and IAM-native |
| "Org Policy overrides IAM allow bindings" | Org Policy restricts config; IAM restricts access | Use IAM Deny Policies to explicitly block access regardless of allow |
| "Cloud Shell for production automation" | Use Cloud Build/GitHub Actions + WIF | Cloud Shell is ephemeral, user-authenticated, not CI/CD-ready |
| "Project ID can be renamed after creation" | Project ID is immutable | Must delete and recreate; plan naming convention upfront |
| "Billing export includes 6 months of historical data" | Export is NOT retroactive | Starts from enable date; no backfilling |
| "Network tags can be used for IAM conditions" | Network tags only work for firewall rules | IAM uses Tags (hierarchical) or Labels (filtering) |
| "Quota increases are automatic with CUDs" | Quota must be requested separately | CUDs guarantee pricing, not capacity; request quota via API/Console |

---

## 📝 Scenario-Based Practice Questions (PCA 2026 Style)

### Question 1: Cost Attribution & Billing Export
> **Scenario**: A financial services company enables BigQuery billing export today. Finance needs a chargeback report for the last quarter showing costs by team (`team` label) and environment (`environment` label). Many resources were created before labels were enforced.
> 
> **Question**: How should the architect deliver accurate cost attribution moving forward?
> 
> A) Apply missing labels via Cloud Asset Inventory script; query historical export with `IFNULL` fallback  
> B) Use Cloud Recommender to auto-tag resources; export will backfill historical costs  
> C) Acknowledge historical data lacks labels; implement Org Policy to enforce mandatory labels; use billing export for future chargeback  
> D) Switch to Cloud Storage CSV export for better label parsing of historical data  
> 
> **Answer**: C  
> **Rationale**: Billing export is not retroactive. Historical events lack label metadata. The correct approach is to enforce labels going forward via Org Policy/custom constraints, use the export for future attribution, and supplement historical data with manual allocation or asset inventory snapshots. Option A/B/D incorrectly assume retroactive label application or export backfilling.

### Question 2: Governance & Compliance Enforcement
> **Scenario**: A healthcare organization must ensure all new Cloud SQL instances use CMEK encryption. Development teams need flexibility to use GMEK in dev environments, but prod must strictly enforce CMEK. The solution must prevent non-compliant instances from being created.
> 
> **Question**: Which governance approach BEST meets these requirements?
> 
> A) IAM Condition requiring `resource.properties.cmek: true` on Cloud SQL  
> B) Folder-level Org Policy with custom CEL constraint checking `resource.name.contains('/prod/') ? resource.encryptionConfig.kmsKeyName != '' : true`  
> C) Cloud Function triggered on Cloud SQL creation to validate encryption and delete non-compliant instances  
> D) Security Command Center alert + manual remediation ticket  
> 
> **Answer**: B  
> **Rationale**: Custom Org Policy constraints evaluate at creation time and can use CEL to conditionally enforce rules based on resource paths or tags. This prevents non-compliant resources from being created. Option A misuses IAM (controls access, not config); C is reactive and risky; D doesn't prevent creation.

### Question 3: Resource Hierarchy & Isolation
> **Scenario**: An enterprise runs 50+ microservices. Requirements:
> - Dev, staging, and prod must be isolated at the billing and IAM level
> - Platform team needs to manage shared networking across all environments
> - Finance needs to view costs per environment without seeing resource details
> 
> **Question**: Which hierarchy and IAM design BEST meets these requirements?
> 
> A) Single project with namespaces; platform team gets `roles/editor`; finance gets `roles/billing.viewer`  
> B) Separate projects per environment under respective folders; Shared VPC host project; finance granted `roles/billing.viewer` on billing account; platform gets `roles/compute.networkAdmin` on host  
> C) Separate folders per environment; all projects share one billing account; platform gets `roles/owner`; finance gets `roles/viewer` on projects  
> D) VPC peering between dev/staging/prod projects; platform gets `roles/container.admin`; finance gets project-level billing access  
> 
> **Answer**: B  
> **Rationale**: Separate projects per environment provide true IAM/billing isolation. Shared VPC enables centralized network management. `roles/billing.viewer` at the billing account level gives cost visibility without resource access. Option A lacks isolation; C violates least privilege (`roles/owner`); D lacks billing/account separation and uses peering incorrectly for shared services.

### Question 4: CI/CD Authentication & Automation
> **Scenario**: A startup uses GitHub Actions to deploy Cloud Run services. Requirements:
> - No long-lived credentials in repositories
> - Deployments must be restricted to the `main` branch only
> - Must comply with SOC2 audit requirements
> 
> **Question**: Which authentication strategy BEST meets these requirements?
> 
> A) Store a rotated service account JSON key in GitHub Secrets with branch protection rules  
> B) Workload Identity Federation with attribute mapping to `attribute.ref == 'refs/heads/main'`  
> C) Deploy a self-hosted runner on GCE with attached service account  
> D) Use Cloud Build triggered by GitHub webhooks instead of GitHub Actions  
> 
> **Answer**: B  
> **Rationale**: Workload Identity Federation enables keyless, auditable access from GitHub. Attribute mapping restricts access to specific branches. All token exchanges are logged in Cloud Audit Logs. Option A violates keyless best practice; C ties identity to VM lifecycle, not repo; D avoids the question's constraint (GitHub Actions must be used).

### Question 5: Quota Management & IaC Pipeline
> **Scenario**: A company runs Terraform in Cloud Build to provision GKE clusters. The pipeline frequently fails with `QUOTA_EXCEEDED` for `CPUS_ALL_REGIONS` during peak deployment times. Requirements:
> - Prevent pipeline failures without manual intervention
> - Maintain cost predictability
> - Comply with change management approval for quota increases
> 
> **Question**: Which solution BEST addresses these constraints?
> 
> A) Request a permanent 10x quota increase via Console and hardcode in Terraform  
> B) Implement Cloud Monitoring alert on quota utilization → Pub/Sub → Approval workflow → Automated quota increase request via API → Retry pipeline  
> C) Use preemptible VMs exclusively to bypass quota limits  
> D) Disable quota checks in Terraform provider configuration  
> 
> **Answer**: B  
> **Rationale**: Quota increases require approval but can be automated via the Quota API. Monitoring alerts trigger proactive requests before failures. This balances automation, compliance, and reliability. Option A bypasses change management; C doesn't solve CPU quota limits and affects reliability; D is impossible (quota checks are enforced by GCP, not Terraform).

---

## ✅ How to Integrate This Addendum

1. **Append** the `Missing & High-Yield Topics` section after `11. GCP Resource Hierarchy — Introduction`.
2. **Insert** the `Advanced PCA Decision Frameworks` table before the `Frequently Tested Exam Scenarios` section.
3. **Replace** any generic "exam traps" with the expanded `2026 Exam Traps & Anti-Patterns` table.
4. **Add** the `Scenario-Based Practice Questions` to your exam prep rotation. Time yourself: ~60-75 seconds per question.
5. **Cross-Reference** with your networking/IAM/AI guides: Tags ↔ IAM Conditions, CUDs ↔ Cost Optimization, WIF ↔ CI/CD, Custom Org Policy ↔ Compliance.

---
*This enrichment aligns with the official Google Cloud PCA exam guide, 2024-2026 service GA announcements, and real candidate feedback. Always validate architecture decisions against current [GCP Fundamentals Documentation](https://cloud.google.com/docs).*


*Study Guide Version: 1.0 | Aligned to PCA Exam Guide (2024–2025) | Topics: SDK, Cloud Shell, Projects, Org Policy, Billing, Resource Hierarchy*
