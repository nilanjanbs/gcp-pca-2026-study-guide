# Google Cloud PCA — Security Services Study Guide

> Companion module to the main PCA Study Guide  
> Topics: Cloud Armor · Cloud CDN Security · Secret Manager · Security Command Center · Model Armor · Cloud IAP · Cloud Endpoints

---

## 📌 Legend
- 🔑 = High-frequency exam topic  
- ⚠️ = Common trap / tricky distinction  
- 💡 = Exam tip or trick  
- 🏗️ = Architecture pattern

---

# 1. Cloud Armor

## 1.1 What Is Cloud Armor?

Cloud Armor is Google Cloud's **DDoS protection and Web Application Firewall (WAF)** service. It sits in front of the **Google Cloud HTTP(S) Load Balancer** and evaluates traffic at Google's edge — before it reaches your backend.

```
Internet → [Cloud Armor Policy] → [HTTPS Load Balancer] → Backend (GKE / GCE / Cloud Run)
```

🔑 **Cloud Armor only works with:**
- Global HTTP(S) Load Balancer (Classic)
- Global HTTP(S) Load Balancer (Next Gen)
- External TCP/SSL Proxy Load Balancers

⚠️ **Trap:** Cloud Armor does **not** protect Internal Load Balancers or Cloud Run/Functions directly — traffic must flow through a supported external LB.

---

## 1.2 Security Policies

A **Security Policy** is the core Cloud Armor resource. It contains ordered **rules** evaluated from lowest priority number to highest.

### Rule Components
| Component | Description |
|---|---|
| **Priority** | Lower number = evaluated first (0 = highest priority) |
| **Match condition** | IP/CIDR, geo, request headers, URL path, or CEL expression |
| **Action** | `allow`, `deny(403/404/429/502)`, `throttle`, `redirect`, `rate-based-ban` |
| **Preview mode** | Log without enforcing; great for testing |

### Default Rule
- Every policy has a **default rule** (priority 2147483647)
- Default action is configurable: `allow` (permissive) or `deny` (default-deny posture)

### Rule Evaluation Order
```
Priority 1000 → match? → apply action → STOP
Priority 2000 → match? → apply action → STOP
...
Default rule → apply
```

💡 **Exam Tip:** Rules are evaluated in **ascending priority order** and evaluation **stops at the first match**. This is the same logic as firewall rules.

---

## 1.3 Types of Protection

### Layer 3/4 — DDoS Protection
- **Always-on** volumetric DDoS mitigation at Google's edge (included free)
- Absorbs large-scale attacks (SYN floods, UDP amplification, etc.)
- No configuration required; automatic

### Layer 7 — WAF (Web Application Firewall)
- Requires Cloud Armor **Standard or Plus tier**
- **Pre-configured WAF rules** (ModSecurity Core Rule Set):
  - SQLi (SQL Injection)
  - XSS (Cross-Site Scripting)
  - LFI (Local File Inclusion)
  - RFI (Remote File Inclusion)
  - RCE (Remote Code Execution)
  - Scanner detection
  - Protocol attack detection

🔑 Each pre-configured rule has a **sensitivity level** (0–4). Higher = more rules, higher false positive risk.

### IP Allow/Deny Lists
- Allow or block specific **IP addresses or CIDR ranges**
- Common use: **allowlist** corporate IP ranges; **denylist** known malicious actors

### Geo-Based Access Control
- Block or allow by **country** (ISO 3166-1 alpha-2 country codes)
- Use case: compliance (restrict service to specific regions), block high-risk geos

### Rate Limiting
| Feature | Description |
|---|---|
| **Throttle** | Limit requests per client per second; excess requests are rejected |
| **Rate-based ban** | Temporarily ban a client that exceeds a request threshold |

💡 **Exam Tip:** "Protect against brute force / credential stuffing" → Cloud Armor **rate-based ban**.

### Bot Management (Recaptcha Integration)
- Integrate with **reCAPTCHA Enterprise** for bot detection
- Actions: `allow`, `deny`, `redirect to reCAPTCHA challenge`
- Assign reCAPTCHA tokens in rules

### Adaptive Protection (ML-Based)
- **Cloud Armor Adaptive Protection**: ML model that detects and generates rules for L7 DDoS attacks in real time
- Available in **Cloud Armor Plus** tier
- Sends alerts and suggests mitigation rules during attacks

---

## 1.4 Tiers

| Feature | Cloud Armor Standard | Cloud Armor Plus |
|---|---|---|
| DDoS protection | ✅ | ✅ |
| IP/Geo rules | ✅ | ✅ |
| WAF pre-configured rules | ✅ | ✅ |
| Rate limiting | ✅ | ✅ |
| Adaptive Protection | Limited | Full (DDoS Response Team) |
| DDoS Response Team support | ❌ | ✅ |
| Pricing model | Per policy + rule | Monthly subscription |

---

## 1.5 Custom Rules with CEL (Common Expression Language)

Advanced rules using **CEL expressions** allow matching on:
- `request.path` — URL path
- `request.headers` — HTTP headers
- `request.method` — GET, POST, etc.
- `origin.region_code` — Country
- `request.query` — Query parameters

**Example:** Block requests where the User-Agent contains `curl`:
```
request.headers['user-agent'].contains('curl')
```

---

## 1.6 Logging and Monitoring

- All Cloud Armor rule evaluations are logged to **Cloud Logging**
- Log fields: `securityPolicyName`, `outcome` (ACCEPT/DENY), `matchedFieldType`, `previewMode`
- Use **preview mode** before enforcing rules to validate logic

---

## 1.7 Cloud Armor — CDN Use Case

### The Problem: Cache Poisoning & Origin Protection
When Cloud CDN sits in front of your backend, there are two threat surfaces:
1. **CDN cache** — served to end users
2. **Origin** — the backend being hit on cache misses

```
User → [Cloud Armor] → [HTTPS LB] → [Cloud CDN Cache]
                                           ↓ (cache miss)
                                      [Backend / Origin]
```

### Cloud Armor + CDN Integration
- Cloud Armor policies apply **at the load balancer**, before the CDN cache check
- On a **cache hit:** Cloud Armor still evaluates the request — blocked requests never reach cache
- On a **cache miss:** Cloud Armor evaluates → allowed requests are forwarded to origin

🔑 **Key insight:** Cloud Armor protects **both** the CDN and the origin. Even cached content requests are evaluated.

### Signed URLs / Signed Cookies with Cloud Armor
- Use **Cloud CDN signed URLs** to restrict who can access cached content
- Cloud Armor complements this by blocking IPs or geos before they even attempt to fetch a URL

### Origin Shielding Pattern
```
Internet → Cloud Armor → HTTPS LB → CDN → Origin Shield → Backend
```
- **Origin shield** (a CDN PoP designated as origin-facing) reduces direct hits on backend
- Cloud Armor filters malicious traffic before it consumes origin shield capacity

### Use Case: Protecting a Media Streaming Platform
1. Cloud Armor: block known malicious IPs, rate-limit scrapers
2. Cloud CDN: cache video segments at edge for low latency
3. Signed URLs: restrict content to authenticated users
4. Adaptive Protection: auto-detect and mitigate volumetric attacks

---

## 1.8 Exam Scenarios — Cloud Armor

| Scenario | Solution |
|---|---|
| Block traffic from specific countries | Geo-based rule with `deny` action |
| Protect against SQL injection | Pre-configured WAF rule: `sqli-v33-stable` |
| Rate-limit API callers | Rate-based rule with `throttle` action |
| Test a new rule without enforcing it | Enable **preview mode** |
| Auto-detect L7 DDoS and get mitigation rules | **Adaptive Protection** (Plus tier) |
| Allow only corporate IP range | IP allowlist rule + default-deny policy |
| Protect against OWASP Top 10 | Enable pre-configured WAF rules |

---

# 2. Secret Manager

## 2.1 What Is Secret Manager?

Secret Manager is a **fully managed secrets storage** service. It stores API keys, passwords, certificates, and any sensitive configuration data as **versioned secrets**.

⚠️ **Never store secrets in:**
- Source code / Git repositories
- Environment variables (in plain text)
- Cloud Storage buckets without encryption
- VM metadata without restriction

---

## 2.2 Core Concepts

| Term | Description |
|---|---|
| **Secret** | A named resource (e.g., `projects/my-project/secrets/db-password`) |
| **Secret Version** | Immutable payload associated with a secret; each update creates a new version |
| **Latest version** | Alias `latest` always points to the most recent version |
| **State** | `ENABLED`, `DISABLED`, `DESTROYED` |

### Secret Lifecycle
```
Create Secret → Add Version (payload) → Access Version → Disable/Destroy Version
```

- Disabling a version blocks access without deleting data
- Destroying a version is **permanent and irreversible**

---

## 2.3 Accessing Secrets

### IAM Roles
| Role | Capability |
|---|---|
| `secretmanager.viewer` | View secret metadata only |
| `secretmanager.secretAccessor` | **Access secret payload** (most commonly granted) |
| `secretmanager.secretVersionManager` | Add/disable/destroy versions |
| `secretmanager.admin` | Full control |

🔑 Grant `secretAccessor` to the **service account** of your app/Cloud Function/GKE pod — not to broad project roles.

### Access Patterns
```python
# Python example — access latest version
from google.cloud import secretmanager
client = secretmanager.SecretManagerServiceClient()
name = "projects/PROJECT_ID/secrets/SECRET_NAME/versions/latest"
response = client.access_secret_version(request={"name": name})
payload = response.payload.data.decode("UTF-8")
```

---

## 2.4 Rotation

- **Manual rotation:** add a new version; update app to use `latest`
- **Automatic rotation:** configure a **rotation schedule** (CRON); Secret Manager publishes a **Pub/Sub notification** on rotation
- App must handle rotation events (listen to Pub/Sub topic to reload secrets)

💡 **Exam Tip:** "Auto-rotate database passwords" → Secret Manager rotation + Pub/Sub + Cloud Functions (to update the DB password and create a new version).

---

## 2.5 Replication Policy

| Type | Description |
|---|---|
| **Automatic** | Google chooses replica locations; default |
| **User-managed** | You specify regions; required for data residency compliance |

⚠️ **Trap:** If data residency regulations require secrets to stay in specific regions → use **user-managed replication**.

---

## 2.6 Integration with GCP Services

| Service | How Secret Manager Integrates |
|---|---|
| **Cloud Functions** | Mount as volume or access via API; SA needs `secretAccessor` |
| **Cloud Run** | Mount as volume or env var (auto-fetched at deploy time) |
| **GKE** | Use **External Secrets Operator** or mount via **Secret Store CSI Driver** |
| **Cloud Build** | Reference secrets in build steps via `secretEnv` |
| **App Engine** | Access via API in application code |

🔑 **Cloud Run + Secret Manager:** You can configure secrets as **mounted volumes** (latest version auto-refreshed) or **environment variables** (injected at deployment — static).

---

## 2.7 CMEK with Secret Manager

- By default, secrets are encrypted with **Google-managed keys**
- Enable **CMEK** to encrypt with a **Cloud KMS key** you control
- Allows you to revoke access by disabling/destroying the KMS key

---

## 2.8 Exam Scenarios — Secret Manager

| Scenario | Solution |
|---|---|
| Store DB credentials securely | Secret Manager; grant SA `secretAccessor` |
| Rotate API key without downtime | Add new version; app reads `latest`; disable old version |
| Ensure secrets stay in EU | User-managed replication to EU regions |
| Audit who accessed a secret | Cloud Logging (Secret Manager data access audit logs) |
| Revoke access to all secrets immediately | Disable the KMS key (if CMEK configured) |
| CI/CD pipeline needs DB password | Cloud Build `secretEnv` referencing Secret Manager |

---

# 3. Security Command Center (SCC)

## 3.1 What Is SCC?

Security Command Center is Google Cloud's **centralised security and risk management platform**. It provides visibility into your security posture across all GCP resources.

---

## 3.2 SCC Tiers

| Tier | Features |
|---|---|
| **Standard** (free) | Security Health Analytics, Web Security Scanner (limited), IAM recommender |
| **Premium** | All Standard + Event Threat Detection, Container Threat Detection, VM Threat Detection, Compliance dashboards, Attack path simulation |
| **Enterprise** | All Premium + multi-cloud (AWS/Azure), AI-powered risk engine, case management, SOAR integrations |

💡 **Exam Tip:** For compliance reporting (PCI DSS, CIS, NIST) → SCC **Premium**. For basic misconfiguration detection → Standard.

---

## 3.3 Core Capabilities

### Security Health Analytics
- Scans GCP resources for **misconfigurations**
- Examples of findings:
  - Public storage buckets
  - Firewall rules allowing all ingress (`0.0.0.0/0`)
  - Primitive roles (Owner/Editor) assigned to users
  - MFA not enforced for admin accounts
  - Public SQL instances
  - Audit logging disabled

### Event Threat Detection (Premium)
- Analyzes **Cloud Logging streams** in real time for threats
- Detects:
  - Cryptomining (unusual GPU/CPU usage patterns)
  - Data exfiltration (abnormal API calls to GCS)
  - Brute-force SSH attacks
  - Outbound connections to known C2 servers
  - IAM anomalies (privilege escalation)

### Container Threat Detection (Premium)
- Runtime threat detection for **GKE workloads**
- Detects: reverse shells, unexpected processes in containers, crypto miners

### VM Threat Detection (Premium)
- Memory scanning of running VMs for malware signatures
- Detects: cryptomining malware, rootkits

### Web Security Scanner
- Scans App Engine, GKE, and Compute Engine web apps
- Detects: XSS, mixed content, outdated libraries, exposed credentials in URLs

---

## 3.4 Findings

- **Finding** = a security issue detected by SCC
- Each finding has: severity (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`), resource, category, and state (`ACTIVE`, `INACTIVE`, `MUTED`)
- Findings are exported to **Cloud Logging** and can trigger **Pub/Sub** notifications → **Cloud Functions** for auto-remediation

🔑 **Auto-remediation pattern:**
```
SCC Finding → Pub/Sub → Cloud Functions → Remediate (e.g., remove public IAM binding)
```

---

## 3.5 Compliance & Posture Management

- Pre-built compliance dashboards: **CIS GCP Benchmarks**, **PCI DSS**, **NIST 800-53**, **ISO 27001**, **HIPAA**
- **Security Posture** (Preview): define desired state as code; detect drift

---

## 3.6 Attack Path Simulation (Premium)

- Simulates potential attack paths to **high-value resources** (crown jewels)
- Shows an attacker's most likely route from a public endpoint to sensitive data
- Helps prioritise which findings to fix first

---

## 3.7 Exam Scenarios — SCC

| Scenario | Solution |
|---|---|
| Find all public GCS buckets across org | SCC Security Health Analytics |
| Detect cryptomining in real time | SCC Event Threat Detection (Premium) |
| Auto-remediate open firewall rules | SCC → Pub/Sub → Cloud Functions |
| Prove PCI DSS compliance to auditors | SCC Premium compliance dashboard |
| Detect malware in running VMs | SCC VM Threat Detection (Premium) |
| Monitor GKE for runtime threats | SCC Container Threat Detection |

---

# 4. Model Armor

## 4.1 What Is Model Armor?

Model Armor is Google Cloud's **managed safety and security layer for AI/LLM workloads**. It acts as a policy enforcement point between users and AI models (including Vertex AI models and third-party LLMs).

---

## 4.2 Core Function

Model Armor inspects:
- **Prompts (inputs)** before they reach the model
- **Responses (outputs)** before they reach the user

```
User Prompt → [Model Armor: Prompt Scan] → LLM → [Model Armor: Response Scan] → User
```

---

## 4.3 Threat Categories Addressed

| Threat | Description |
|---|---|
| **Prompt Injection** | Attacker tries to override system instructions via user input |
| **Jailbreaking** | Attempts to bypass model safety guidelines |
| **Sensitive Data in Prompts** | PII/PCI data sent to external models inadvertently |
| **Malicious URLs in Responses** | Model generates links to phishing/malware sites |
| **Toxic / Harmful Content** | Hate speech, violence, adult content in outputs |
| **Data Exfiltration via Prompts** | Attacker tries to extract training data or system prompts |

---

## 4.4 How It Works

- **Templates:** define which detectors to enable and sensitivity thresholds
- **Detectors:**
  - **Prompt injection detector**
  - **Malicious URL detector**
  - **Sensitive data (DLP) detector** — integrates with Cloud DLP
  - **Responsible AI content filters** (safety categories: harassment, hate speech, etc.)
- Actions: `BLOCK`, `FLAG` (allow but log), or `SANITISE`

---

## 4.5 Integration Points

| Integration | Description |
|---|---|
| **Vertex AI** | Native integration; apply templates to model endpoints |
| **Third-party LLMs** | Via REST API inspection before routing to external model |
| **Cloud Logging** | All scan results logged for audit |
| **SCC** | Model Armor findings can feed into SCC |

---

## 4.6 Exam Scenarios — Model Armor

| Scenario | Solution |
|---|---|
| Prevent users from injecting malicious instructions into a chatbot | Model Armor prompt injection detector |
| Ensure no PII is sent to an external LLM | Model Armor + DLP detector on prompts |
| Block harmful/toxic content in AI responses | Model Armor responsible AI content filters |
| Audit all prompts and responses for compliance | Model Armor logging to Cloud Logging |
| Protect a Vertex AI model from jailbreaking | Model Armor jailbreak detector template |

💡 **Exam Tip:** Model Armor is purpose-built for **GenAI/LLM security** — distinct from Cloud Armor (web/DDoS) and SCC (cloud posture). Don't confuse them.

---

# 5. Cloud IAP (Identity-Aware Proxy)

## 5.1 What Is Cloud IAP?

Cloud IAP enforces **identity-based access control** at the application layer. Instead of relying solely on network-level controls (VPN, firewall), IAP verifies who the user is (authentication) and whether they are authorised (authorisation) before allowing access.

```
User → Google Sign-In (OAuth 2.0) → [IAP] → Application / VM / Resource
```

---

## 5.2 How IAP Works

1. User requests access to a resource protected by IAP
2. IAP redirects unauthenticated users to **Google login**
3. After login, IAP checks the user's identity against **IAM policies**
4. If authorised (`roles/iap.httpsResourceAccessor`), the request is forwarded with a **signed JWT header** (`X-Goog-IAP-JWT-Assertion`)
5. Application can optionally **validate the JWT** for additional security

---

## 5.3 What IAP Protects

| Resource Type | Notes |
|---|---|
| **App Engine apps** | Natively supported |
| **GKE workloads** | Via Ingress + BackendConfig |
| **Compute Engine VMs** (HTTP apps) | Via HTTPS Load Balancer |
| **Cloud Run services** | Via HTTPS Load Balancer |
| **SSH/TCP tunnelling** | IAP TCP Forwarding for SSH/RDP without public IP |

---

## 5.4 IAP TCP Forwarding

🔑 This is a **very high-frequency exam topic**.

- Allows SSH/RDP access to VMs with **no public IP** and **no VPN**
- Traffic tunnelled through IAP using Google's infrastructure
- IAM controls who can create tunnels: `roles/iap.tunnelResourceAccessor`

```bash
# Connect to a private VM via IAP
gcloud compute ssh VM_NAME --tunnel-through-iap
```

**Architecture:**
```
Developer laptop → gcloud CLI → [Cloud IAP TCP] → Private VM (no external IP)
```

⚠️ **Trap:** IAP TCP Forwarding requires the **Cloud IAP API** to be enabled and firewall rules allowing ingress from `35.235.240.0/20` (IAP's IP range) on port 22.

---

## 5.5 IAP vs VPN

| Feature | Cloud IAP | Cloud VPN |
|---|---|---|
| Auth model | Identity (user/SA) | Network (IP-based) |
| Granularity | Per-app / per-VM | Network-level |
| No VPN client needed | ✅ | ❌ |
| Works with BeyondCorp | ✅ | ❌ |
| Best for | Zero-trust, developer access | Site-to-site connectivity |

💡 **Exam Tip:** "Zero trust access to internal app without VPN" → **Cloud IAP**. "Site-to-site network connectivity" → **Cloud VPN**.

---

## 5.6 BeyondCorp Enterprise

Cloud IAP is the underlying technology for **BeyondCorp Enterprise** — Google's zero-trust access solution.

Key principles:
- **Access is not granted based on network location** (inside/outside corp network)
- **Access is granted based on identity + device posture**
- Enforced at the application layer, not the network perimeter

---

## 5.7 Context-Aware Access

Extends IAP with **access levels** based on context:
- Device OS and patch status
- Corporate-managed device check
- IP address / geographic location
- User identity + group membership

Use case: Allow access only from **managed devices on the corporate network** using a single IAP policy.

---

## 5.8 IAP JWT Validation

When IAP forwards a request, it injects:
- `X-Goog-IAP-JWT-Assertion` — signed JWT containing user identity
- `X-Goog-Authenticated-User-Email` — user's email

🔑 **Application should validate the JWT** to ensure it was signed by IAP and not forged by a user who bypassed IAP (e.g., direct internal access).

---

## 5.9 Exam Scenarios — Cloud IAP

| Scenario | Solution |
|---|---|
| SSH to a VM with no public IP, no VPN | IAP TCP Forwarding |
| Restrict internal web app to specific Google Group | IAP + IAM binding on the resource |
| Zero-trust access to App Engine from contractors | IAP with Google Workspace identity |
| Access GKE-hosted app without VPN | IAP via Ingress + BackendConfig |
| Enforce device trust before granting app access | IAP + Context-Aware Access |
| Validate that a request truly came through IAP | Validate `X-Goog-IAP-JWT-Assertion` JWT |

---

# 6. Cloud Endpoints

## 6.1 What Is Cloud Endpoints?

Cloud Endpoints is an **API management platform** built on **Extensible Service Proxy (ESP)**. It provides: authentication, monitoring, logging, and rate-limiting for APIs running on GCP.

---

## 6.2 Supported Backends

| Backend | Notes |
|---|---|
| **Cloud Run** | Most common modern pairing |
| **GKE** | ESP runs as a sidecar container |
| **Compute Engine** | ESP runs on the VM |
| **App Engine** | Supported |

---

## 6.3 Core Features

| Feature | Description |
|---|---|
| **Authentication** | Verify API keys, JWT tokens (Firebase Auth, Auth0, Google ID tokens) |
| **API Key Management** | Issue and manage API keys; restrict by IP or referrer |
| **Monitoring** | Latency, error rate, request count in Cloud Monitoring |
| **Logging** | Request/response logs in Cloud Logging |
| **Quotas** | Set per-consumer quotas (e.g., 1000 requests/day/API key) |
| **OpenAPI / gRPC** | Supports OpenAPI 2.0 spec and gRPC service definitions |

---

## 6.4 Extensible Service Proxy (ESP)

- **ESP** is a **NGINX-based proxy** that runs alongside your API container/VM
- Handles auth, logging, monitoring without changing app code
- **ESPv2** is the next-gen version (Envoy-based); required for Cloud Run

```
Client → [ESP / ESPv2] → Your API Backend
           ↕
     Service Control API
    (auth check, quota, logging)
```

---

## 6.5 Authentication Methods

| Method | Description |
|---|---|
| **API Keys** | Simple; for server-to-server, usage tracking |
| **Google ID Tokens** | For Google-signed JWTs (service accounts, Firebase) |
| **JWT (Auth0, Firebase)** | Third-party identity providers |
| **Service Account Auth** | GCP-to-GCP calls via SA credentials |

⚠️ **Trap:** API keys identify the **calling application**, not the **user**. For user-level auth, use JWT/OAuth2.

---

## 6.6 OpenAPI vs gRPC

| Aspect | OpenAPI (REST) | gRPC |
|---|---|---|
| Protocol | HTTP/1.1 + JSON | HTTP/2 + Protobuf |
| Performance | Standard | High (binary, multiplexed) |
| Definition file | `openapi.yaml` | `.proto` file |
| Browser support | Native | Needs grpc-web or transcoding |
| Best for | Public APIs, mobile clients | Internal microservices, streaming |

---

## 6.7 Cloud Endpoints vs Apigee

| Feature | Cloud Endpoints | Apigee |
|---|---|---|
| Complexity | Simple | Enterprise-grade |
| Mediation / transforms | Limited | Full (policies, transforms) |
| Developer portal | No | Yes |
| Monetisation | No | Yes |
| Cost | Low | High |
| Best for | GCP-native API proxy | Full API lifecycle management |

💡 **Exam Tip:** "Simple API auth + monitoring for a GCP service" → **Cloud Endpoints**. "Full API gateway with developer portal, monetisation, mediation" → **Apigee**.

---

## 6.8 Exam Scenarios — Cloud Endpoints

| Scenario | Solution |
|---|---|
| Add authentication to a Cloud Run API | Cloud Endpoints (ESPv2) with JWT or API key |
| Rate-limit API consumers | Cloud Endpoints quotas per API key |
| Monitor API latency and error rates | Cloud Endpoints → Cloud Monitoring dashboard |
| Expose a gRPC microservice securely | Cloud Endpoints with gRPC + proto definition |
| Validate Firebase Auth tokens for API calls | Cloud Endpoints JWT auth with Firebase issuer |
| Log all API requests for audit | Cloud Endpoints → Cloud Logging |

---

# 7. Security Services — Comparison & Decision Framework

## 7.1 Service Selection Matrix

| Need | Service |
|---|---|
| Protect web app from DDoS / SQLi / XSS | **Cloud Armor** |
| Rate-limit or geo-block API/web traffic | **Cloud Armor** |
| Store and rotate API keys / DB passwords | **Secret Manager** |
| Detect cloud misconfigurations at org level | **SCC** (Standard) |
| Real-time threat detection (crypto mining, exfil) | **SCC** (Premium) |
| Prove PCI/HIPAA compliance | **SCC** (Premium) |
| Protect LLM prompts and responses | **Model Armor** |
| Prevent PII leakage to AI models | **Model Armor + DLP** |
| Zero-trust access to internal apps / VMs | **Cloud IAP** |
| SSH to private VM without VPN or public IP | **Cloud IAP TCP Forwarding** |
| Add auth + monitoring to an API | **Cloud Endpoints** |
| Full enterprise API management | **Apigee** |

---

## 7.2 Frequently Confused Pairs

| Pair | Distinction |
|---|---|
| **Cloud Armor vs Cloud IAP** | Armor = network/WAF protection. IAP = identity-based app access. |
| **Cloud Armor vs Firewall Rules** | Armor = Layer 7, at edge, WAF. Firewall = Layer 3/4, within VPC. |
| **Secret Manager vs KMS** | Secret Manager = stores secrets/credentials. KMS = manages encryption keys. |
| **SCC vs Cloud Logging** | SCC = security posture + threat detection. Logging = raw log storage/query. |
| **Model Armor vs Cloud Armor** | Model Armor = AI/LLM safety. Cloud Armor = web app/DDoS protection. |
| **Cloud Endpoints vs Apigee** | Endpoints = simple GCP-native proxy. Apigee = full enterprise API platform. |
| **IAP vs Cloud VPN** | IAP = identity-based, per-app, zero-trust. VPN = network-level, site-to-site. |

---

## 7.3 Layered Security Architecture (Reference)

```
                    ┌──────────────────────────────────┐
                    │         INTERNET TRAFFIC          │
                    └────────────────┬─────────────────┘
                                     │
                    ┌────────────────▼─────────────────┐
                    │          CLOUD ARMOR              │  ← DDoS, WAF, Geo, Rate Limit
                    └────────────────┬─────────────────┘
                                     │
                    ┌────────────────▼─────────────────┐
                    │       HTTPS LOAD BALANCER         │  ← TLS termination, routing
                    └────────────────┬─────────────────┘
                                     │
                    ┌────────────────▼─────────────────┐
                    │           CLOUD IAP               │  ← Identity verification (internal apps)
                    └────────────────┬─────────────────┘
                                     │
                    ┌────────────────▼─────────────────┐
                    │       CLOUD ENDPOINTS (ESP)       │  ← API auth, quotas, logging
                    └────────────────┬─────────────────┘
                                     │
                    ┌────────────────▼─────────────────┐
                    │        APPLICATION BACKEND        │  ← Uses Secret Manager for creds
                    │     (Cloud Run / GKE / GCE)       │     Monitored by SCC
                    └──────────────────────────────────┘
                              (AI workloads)
                    ┌──────────────────────────────────┐
                    │          MODEL ARMOR              │  ← Prompt/response safety
                    └──────────────────────────────────┘
```

---

## 7.4 Top Exam Tips — Security Module

1. 🔑 **Cloud Armor requires an HTTP(S) LB.** It cannot protect services without one.

2. 🔑 **IAP TCP Forwarding = SSH without public IP.** This is the most tested IAP scenario.

3. 🔑 **Secret Manager ≠ KMS.** Secret Manager stores *secrets*. KMS manages *encryption keys*. They complement each other (CMEK on secrets).

4. 🔑 **SCC Premium for compliance and threat detection.** Standard only covers misconfigurations.

5. 🔑 **Model Armor ≠ Cloud Armor.** Model = AI safety. Cloud = web security. Don't swap them under exam pressure.

6. 🔑 **Cloud Endpoints is for API auth and monitoring** — not a full API gateway. For enterprise needs, use Apigee.

7. 🔑 **Always prefer managed identity over credentials** — Workload Identity, IAP, service accounts with IAM vs API keys and passwords where possible.

8. 🔑 **Preview mode in Cloud Armor** = test before enforce. Safe rollout of WAF rules.

9. 🔑 **SCC auto-remediation pattern:** SCC Finding → Pub/Sub notification → Cloud Functions → fix the resource.

10. 🔑 **Data residency for secrets** = user-managed replication in Secret Manager. Always check compliance requirements.

---

*Security module — Google Professional Cloud Architect (PCA) Exam Study Guide.*  
*Cross-reference with: [Cloud Armor docs](https://cloud.google.com/armor/docs) | [Secret Manager docs](https://cloud.google.com/secret-manager/docs) | [SCC docs](https://cloud.google.com/security-command-center/docs)*
