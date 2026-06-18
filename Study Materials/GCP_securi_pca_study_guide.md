# Google Cloud Professional Cloud Architect (PCA) Exam Study Guide
## Security Domain: Comprehensive Study Materials

> **Target Certification**: Google Cloud Professional Cloud Architect (PCA)  
> **Focus Area**: Security Services & Best Practices  
> **Last Updated**: April 2026  
> **Format**: Markdown (.md)

---

## 📋 Table of Contents

1. [Exam Overview & Structure](#exam-overview--structure)
2. [PCA Exam Tips & Tricks](#pca-exam-tips--tricks)
3. [Cloud Armor Introduction](#cloud-armor-introduction)
4. [Cloud Armor - CDN Use Case](#cloud-armor---cdn-use-case)
5. [Secret Manager](#secret-manager)
6. [Security Command Center](#security-command-center)
7. [Model Armor](#model-armor)
8. [Cloud Identity-Aware Proxy (IAP)](#cloud-identity-aware-proxy-iap)
9. [Cloud Endpoints](#cloud-endpoints)
10. [Practice Scenarios & Questions](#practice-scenarios--questions)
11. [Quick Reference Tables](#quick-reference-tables)

---

## Exam Overview & Structure

### PCA Exam Domains (2026)

| Domain | Weight | Key Focus Areas |
|--------|--------|----------------|
| Designing & Planning Cloud Solution Architecture | ~30% | Business requirements, compliance, cost optimization |
| Managing & Provisioning Solution Infrastructure | ~25% | Compute, networking, storage, deployment |
| **Designing for Security & Compliance** | **~20%** | **IAM, encryption, network security, audit** |
| Analyzing & Optimizing Technical/Business Processes | ~15% | Monitoring, reliability, cost management |
| Managing Implementation | ~10% | CI/CD, migration, operations |

> ⚠️ **Security is a cross-cutting concern** - expect security questions in ALL domains, not just the dedicated security section [[1]][[3]]

### Exam Format
- **Duration**: 2 hours
- **Questions**: 50-60 multiple-choice/multiple-select
- **Case Studies**: 20-30% of questions reference fictional company scenarios [[82]]
- **Passing Score**: Not publicly disclosed (scaled scoring)
- **Delivery**: Online proctored or test center

---

## PCA Exam Tips & Tricks 🎯

### 🧠 Strategic Approach

1. **Master the Case Studies First**
   - Read ALL case studies BEFORE starting questions [[73]]
   - Take notes on: company size, compliance needs, existing infrastructure, budget constraints
   - Case studies appear throughout the exam - quick reference saves time

2. **Focus on Solutions, Not Just Services**
   - PCA tests architectural thinking, not memorization [[86]]
   - Ask: "What problem does this solve?" not just "What does this service do?"
   - Understand trade-offs: cost vs. performance, security vs. usability

3. **Know the "Why" Behind Best Practices**
   - Least privilege IAM → reduces blast radius
   - VPC Service Controls → prevent data exfiltration
   - Cloud Armor at edge → stop attacks before they reach origin

4. **Time Management Strategy**
   - ~45 seconds per question average [[84]]
   - Flag difficult questions, return later
   - Don't overthink - first instinct is often correct

5. **Hands-On Practice is Non-Negotiable**
   - Use Qwiklabs or your own GCP project with $300 free credit [[43]]
   - Practice: deploying Cloud Armor policies, configuring IAP, managing secrets

### 🔍 Question Analysis Framework

```
When you see a security question:
1. IDENTIFY the threat model (DDoS? data leak? unauthorized access?)
2. LOCATE the attack surface (edge? application? data layer?)
3. MATCH to appropriate GCP service(s)
4. VERIFY compliance/regulatory requirements
5. CHECK for cost/performance implications
```

### 🚫 Common Pitfalls to Avoid

| Mistake | Correct Approach |
|---------|-----------------|
| Choosing most secure option regardless of cost | Balance security with business requirements |
| Forgetting shared responsibility model | Customer secures IN cloud, Google secures OF cloud |
| Overlooking IAM conditions | Use attributes (time, location, device) for fine-grained access |
| Ignoring data residency requirements | Use regional secrets, specify location constraints |
| Assuming one service solves everything | Layer defenses: Cloud Armor + IAP + SCC + Secret Manager |

---

## Cloud Armor Introduction

### What is Cloud Armor?

Cloud Armor is Google Cloud's **edge security service** that provides DDoS protection and WAF (Web Application Firewall) capabilities at the Google Front End (GFE) layer [[16]].

### Key Exam Concepts

```yaml
Core Capabilities:
  - DDoS Protection: 
    • Layer 3/4 volumetric attack mitigation
    • Automatic detection and mitigation
  - WAF (Web Application Firewall):
    • Preconfigured OWASP Top 10 rules
    • Custom rules with CEL (Common Expression Language)
  - Bot Management:
    • reCAPTCHA Enterprise integration
    • Rate limiting rules
  - Adaptive Protection:
    • ML-based anomaly detection
    • Auto-recommendations for new rules
```

### Architecture Flow

```
Internet → Google Front End (GFE) → Cloud Armor Policy → Load Balancer → Backend
                          ↑
                  Security evaluation happens HERE (edge)
```

### PCA Exam Focus Areas

✅ **Must-Know for Exam**:
- Cloud Armor attaches to **backend services** of global external Application Load Balancer [[10]]
- Rules evaluated in **priority order** (lower number = higher priority)
- Use **preview mode** to test rules before enforcement [[10]]
- **JSON parsing** must be enabled to inspect request bodies for WAF rules [[10]]

✅ **Rule Priority Best Practice**:
```
Priority Order (Highest to Lowest):
1. Explicit deny (malicious IPs, regions) - Priority 1-10
2. Trusted allow (scanners, internal) - Priority 11-20  
3. Security rules (OWASP, custom) - Priority 21-90
4. General allow rules - Priority 91-999
5. Default deny all - Priority 1000+
```
*Leave gaps of 10 between priorities for future insertions* [[10]]

### Sample Exam Question

> **Scenario**: A financial services company needs to block traffic from high-risk countries while allowing legitimate international customers. They also want to protect against SQL injection attacks.
>
> **Question**: Which Cloud Armor configuration BEST meets these requirements?
>
> A) Single rule: `origin.region_code in ['CN', 'RU', 'IR']` with deny action  
> B) Two rules: (1) Geo-blocking rule priority 10, (2) OWASP SQLi rule priority 20  
> C) Enable Adaptive Protection only, no custom rules  
> D) Use rate limiting rule with threshold of 100 requests/minute  
>
> **Answer**: B  
> **Rationale**: Layered approach with explicit geo-blocking (high priority) followed by WAF rules (medium priority) provides defense-in-depth. Adaptive Protection alone may not catch targeted attacks; rate limiting addresses volume but not attack vectors [[10]][[16]].

---

## Cloud Armor - CDN Use Case

### Cloud Armor + Cloud CDN Integration

When protecting applications using **Cloud CDN**, Cloud Armor can be configured with two policy types [[65]]:

| Policy Type | Applies To | Use Case |
|------------|-----------|----------|
| **Edge Security Policy** | All requests (cached + uncached) | Filter malicious requests BEFORE cache lookup |
| **Backend Security Policy** | Cache misses only (dynamic content) | Protect origin server for non-cacheable content |

### Architecture Diagram

```
User Request
     ↓
[Cloud CDN Edge POP]
     ↓
┌─────────────────────┐
│ Edge Security Policy │ ← Filters ALL requests
└─────────────────────┘
     ↓
[Cache Hit?] → Yes → Serve from cache → User
     ↓ No
┌─────────────────────┐
│ Backend Security    │ ← Filters cache-miss requests  
│ Policy (if configured)│
└─────────────────────┘
     ↓
[Origin Server: GCE/GKE/Cloud Run]
```

### PCA Exam Scenarios

✅ **When to use Edge vs Backend policies**:
- **Edge policy**: Block malicious bots, geo-restrictions, basic WAF - apply to ALL traffic
- **Backend policy**: Complex business logic validation, user-specific rules - only for origin requests

✅ **Critical Configuration Notes**:
- Edge policies evaluated FIRST, backend policies only if edge allows [[65]]
- For hybrid deployments (on-prem origin), Cloud Armor still protects at GFE layer
- Serverless backends (Cloud Run) require disabling default URLs to prevent bypass [[65]]

### Sample Exam Question

> **Scenario**: An e-commerce site uses Cloud CDN for static assets and dynamic product pages. They need to:
> - Block malicious scanners globally
> - Apply stricter validation only to checkout API calls (dynamic content)
>
> **Question**: What is the MOST efficient Cloud Armor configuration?
>
> A) Single backend security policy with complex rules for all traffic  
> B) Edge policy for scanner blocking + backend policy for checkout validation  
> C) Two edge policies with different priorities  
> D) Use Cloud Endpoints instead of Cloud Armor  
>
> **Answer**: B  
> **Rationale**: Edge policy efficiently blocks scanners at the edge for ALL requests. Backend policy applies additional validation only to cache-miss requests (checkout API), reducing unnecessary processing for cached content [[65]][[69]].

---

## Secret Manager

### What is Secret Manager?

A centralized, secure service for storing API keys, passwords, certificates, and other sensitive data with fine-grained access control and audit logging [[23]].

### Key Exam Concepts

```yaml
Core Features:
  - Versioning: 
    • Each secret has immutable versions (projects/*/secrets/*/versions/*)
    • Reference by version number (not "latest") for reproducible deployments
  - Replication:
    • Automatic (global) vs Regional (data residency compliance)
    • Regional required for strict sovereignty requirements
  - Access Control:
    • IAM roles: roles/secretmanager.admin, roles/secretmanager.viewer
    • Secret-level IAM bindings for least privilege
  - Integration:
    • Application Default Credentials (ADC) preferred over service account keys
    • Workload Identity for GKE, Cloud Run, Cloud Functions
```

### PCA Exam Best Practices [[20]]

✅ **Access Control**:
- Apply **principle of least privilege** at secret level, not just project level
- Use **IAM Conditions** for time-based or attribute-based access
- Avoid exporting secrets to environment variables or filesystems (risk of leakage)

✅ **Administration**:
- **Pin to specific secret versions** in deployments (not `latest` alias)
- **Disable** versions before destroying (reversible safety net)
- **Rotate secrets** periodically; use automation where possible
- Enable **Data Access Audit Logs** for `AccessSecretVersion` events

✅ **Data Residency**:
- Choose **regional secrets** when compliance requires data stay in specific geography
- Enforce location constraints via `constraints/gcp.resourceLocations` org policy

### Code Pattern: Secure Secret Access

```python
# ✅ RECOMMENDED: Using client library with ADC
from google.cloud import secretmanager

def access_secret(project_id, secret_id, version_id="1"):
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version_id}"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

# ❌ AVOID: Passing secrets via environment variables
# os.environ["API_KEY"] = secret_value  # Risk: logs, debug endpoints may leak
```

### Sample Exam Question

> **Scenario**: A healthcare application stores patient API tokens in Secret Manager. Compliance requires:
> - Tokens never leave the `us-central1` region
> - Only the production GKE service account can access version 3+
> - All access must be audited
>
> **Question**: Which configuration BEST meets requirements?
>
> A) Global secret with IAM condition: `resource.name.endsWith("/versions/3") && request.auth.principal == "prod-sa"`  
> B) Regional secret (us-central1) + secret-level IAM + Data Access logs enabled  
> C) Global secret + VPC Service Controls perimeter + Cloud Audit Logs  
> D) Regional secret + environment variable injection in GKE pod spec  
>
> **Answer**: B  
> **Rationale**: Regional secret satisfies data residency. Secret-level IAM enforces least privilege for specific versions. Data Access logs provide required audit trail. Option A uses global replication (violates residency); D leaks secrets to pod environment [[20]][[23]].

---

## Security Command Center

### What is Security Command Center (SCC)?

A cloud-native **risk management platform** that helps prevent, detect, and respond to security threats across Google Cloud resources [[28]].

### Service Tiers Comparison

| Feature | Standard (Free) | Premium (Paid) |
|---------|----------------|----------------|
| Vulnerability Detection | ✅ Basic findings | ✅ Advanced + container scanning |
| Threat Detection | ❌ | ✅ Malware, crypto mining, DDoS alerts |
| Security Posture Mgmt | ❌ | ✅ CIS benchmarks, custom policies |
| Data Security | ❌ | ✅ Sensitive Data Protection integration |
| Attack Path Analysis | ❌ | ✅ Visualize exploitation paths |
| Compliance Reporting | ❌ | ✅ NIST, HIPAA, PCI-DSS mappings |

### Key Exam Concepts

✅ **Findings Sources**:
- **Built-in**: SCC-native detectors (misconfigurations, public buckets)
- **Integrated**: Cloud Armor, Sensitive Data Protection, Event Threat Detection
- **Third-party**: Marketplace partners (CrowdStrike, Snyk, etc.)

✅ **Critical Integrations for PCA**:
- **Cloud Armor**: Exports "Allowed Traffic Spike" and "Increasing Deny Ratio" findings [[10]]
- **Secret Manager**: Detects exposed credentials, overly permissive access
- **Asset Inventory**: Feed for compliance reporting and resource discovery

✅ **Response Workflow**:
```
Detection → Finding in SCC → Mute/Resolve → Export to BigQuery/Pub/Sub → SIEM integration
```

### PCA Exam Focus

✅ **When to recommend SCC Premium**:
- Organizations with compliance requirements (HIPAA, PCI)
- Need for proactive threat detection (not just misconfigurations)
- Multi-project environments requiring centralized visibility

✅ **Cost Optimization Tip**:
- Start with Standard tier for baseline visibility
- Enable Premium only for projects with sensitive workloads
- Use mute rules to reduce noise from known/accepted risks

### Sample Exam Question

> **Scenario**: A financial institution needs to:
> - Continuously monitor for CIS benchmark violations across 50+ projects
> - Receive alerts for potential data exfiltration attempts
> - Export findings to their existing Splunk SIEM
>
> **Question**: Which SCC configuration is MOST appropriate?
>
> A) Standard tier + Cloud Logging export to Pub/Sub  
> B) Premium tier + BigQuery export + custom dashboards  
> C) Premium tier + Pub/Sub export to Splunk connector  
> D) Standard tier + Security Health Analytics only  
>
> **Answer**: C  
> **Rationale**: Premium tier required for compliance monitoring (CIS) and advanced threat detection. Pub/Sub export enables real-time SIEM integration. BigQuery is better for analytics than real-time alerting [[28]][[30]].

---

## Model Armor

### What is Model Armor?

A **specialized security service for AI/ML applications** that screens LLM prompts and responses for security risks, sensitive data, and policy violations [[48]].

### Architecture Flow

```
User Prompt → Model Armor (Input Screening) → LLM → Model Armor (Output Screening) → User Response
                          ↑                           ↑
                  Prompt Injection Check      Sensitive Data Leak Prevention
                  Jailbreak Detection         Harmful Content Filtering
```

### Key Exam Concepts

✅ **Filter Categories**:
| Filter Type | Purpose | Confidence Levels |
|------------|---------|------------------|
| Responsible AI Safety | Block hate speech, harassment, dangerous content | High / Medium+ / Low+ |
| Prompt Injection/Jailbreak | Detect attempts to manipulate LLM behavior | High / Medium+ / Low+ |
| Sensitive Data Protection | Redact PII, credentials, financial data | Likelihood-based (Very Likely → Unlikely) |
| Malicious URL Detection | Block phishing/malware links in responses | Binary (malicious/benign) |

✅ **Enforcement Modes**:
- **Inspect Only**: Log violations without blocking (testing/tuning phase)
- **Inspect and Block**: Actively prevent policy violations (production)

✅ **Template Strategy**:
- **Decouple input/output templates**: Different risk profiles for prompts vs responses
- **Start with High confidence** for Responsible AI filters to minimize false positives
- Use **Medium+ for prompt injection** in most cases; High for high-value targets

### PCA Exam Focus Areas

✅ **When to recommend Model Armor**:
- Customer-facing chatbots or AI agents
- Applications processing sensitive data (PII, IP, financial)
- Compliance requirements for AI output governance

✅ **Integration Points**:
- Works with Vertex AI, third-party LLMs, custom deployments
- Requires Private Service Connect for VPC-SC environments
- Tokens processed count toward pricing (monitor usage)

✅ **Common Exam Trap**:
> Model Armor is **NOT** a replacement for:
> - Cloud Armor (network-layer DDoS/WAF)
> - Secret Manager (credential storage)
> - IAP (user authentication)
> 
> It specifically addresses **AI application-layer risks**

### Sample Exam Question

> **Scenario**: A bank deploys an AI customer service chatbot. Requirements:
> - Prevent prompt injection attacks attempting to extract account data
> - Block responses containing competitor product recommendations
> - Redact any accidentally shared account numbers in user prompts
>
> **Question**: Which Model Armor configuration BEST addresses these needs?
>
> A) Single template with Low+ confidence for all filters  
> B) Input template: High confidence prompt injection + SDP; Output template: High confidence Responsible AI + custom topic filter  
> C) Enable only Sensitive Data Protection with basic configuration  
> D) Use Inspect Only mode to avoid blocking legitimate customer queries  
>
> **Answer**: B  
> **Rationale**: Decoupled templates allow tailored protection: input focuses on injection/data leaks, output on brand safety. High confidence balances security with user experience. Basic SDP lacks customization for account numbers; Inspect Only doesn't enforce protection [[48]][[52]].

---

## Cloud Identity-Aware Proxy (IAP)

### What is Cloud IAP?

A **zero-trust access control service** that verifies user identity and enforces authorization at the application layer, replacing traditional VPNs for app access [[38]].

### Key Exam Concepts

✅ **Supported Backends**:
- App Engine (standard & flexible)
- Cloud Run / Cloud Run functions
- Compute Engine (via TCP forwarding)
- GKE (via Ingress or Gateway API)
- On-premises apps (via IAP tunneling)

✅ **Authentication Flow**:
```
User → IAP (Google login) → Identity Verification → Context-Aware Policy Check → Backend App
                              ↑
                    Device security, location, IP, group membership
```

✅ **Critical PCA Distinctions**:
| Feature | IAP | Traditional VPN |
|---------|-----|----------------|
| Access Scope | Application-level | Network-level |
| User Experience | Direct browser access | Client software + tunnel |
| Policy Granularity | Per-app, per-user, contextual | Per-IP, per-subnet |
| Audit Trail | Per-request user identity | Connection-level logs |

### PCA Exam Best Practices

✅ **When to choose IAP**:
- Securing web applications with user-based access control
- Implementing zero-trust architecture (never trust, always verify)
- Providing secure access without managing VPN infrastructure

✅ **Configuration Tips**:
- Use **OAuth scopes** for programmatic access (not user credentials)
- Combine with **Cloud Armor** for layered security (IAP after Cloud Armor for global LB)
- For Compute Engine: Enable **IAP TCP forwarding** + firewall rules allowing only IAP proxies

✅ **Common Exam Scenario**:
> "Users need secure access to internal admin panel without VPN"
> → **Answer**: Enable IAP for the backend service + restrict access to specific Google Groups

### Sample Exam Question

> **Scenario**: A company has an internal analytics dashboard on Cloud Run. Requirements:
> - Only employees in the "data-team" Google Group can access
> - Access must be blocked from unmanaged devices
> - No VPN client installation allowed
>
> **Question**: Which solution BEST meets requirements?
>
> A) Cloud Armor geo-blocking + API key authentication  
> B) IAP with context-aware access policy + required device security level  
> C) VPC Service Controls perimeter + IAM conditions  
> D) Cloud Endpoints with JWT validation  
>
> **Answer**: B  
> **Rationale**: IAP provides user identity verification + device posture checks via context-aware access. No VPN client needed (browser-based). Cloud Armor lacks user identity; VPC-SC is for data exfiltration prevention; Endpoints handles API auth but not device context [[38]][[43]].

---

## Cloud Endpoints

### What is Cloud Endpoints?

An **API management system** that provides authentication, monitoring, logging, and quota management for APIs using OpenAPI or gRPC specifications [[61]].

### Architecture Options

| Proxy | Based On | Supported Backends | Best For |
|-------|----------|-------------------|----------|
| **ESP** | NGINX | App Engine Flex, GKE, GCE, Kubernetes | Legacy deployments, complex NGINX configs |
| **ESPv2** | Envoy | App Engine Standard, Cloud Run, GKE, Cloud Functions | Modern serverless, gRPC, OpenAPI v3 |

### Key Exam Concepts

✅ **Security Features**:
- **API Keys**: Simple authentication for public APIs
- **JWT Authentication**: Validate Google-signed tokens or custom JWTs
- **IAM Integration**: Use `roles/servicemanagement.admin` for management
- **ESPv2 Auth**: Support for Firebase Auth, Auth0, custom OIDC providers

✅ **PCA Exam Focus**:
- Endpoints is an **API gateway**, not a full-service mesh (vs Anthos Service Mesh)
- Handles **authentication/authorization at API layer**, not network layer
- Logs/metrics integrate with Cloud Monitoring and Cloud Logging

✅ **When to choose Endpoints**:
- Exposing REST/gRPC APIs to external developers
- Need for API key management, quota enforcement, usage analytics
- Simpler alternative to full API management platforms

### Sample Exam Question

> **Scenario**: A startup exposes a public weather API. Requirements:
> - Free tier: 100 requests/day per API key
> - Premium tier: 10,000 requests/day with JWT auth
> - Monitor usage per API key for billing
>
> **Question**: Which GCP service combination BEST meets requirements?
>
> A) Cloud Endpoints with API keys + ESPv2 quota configuration  
> B) Apigee with developer portal + monetization  
> C) Cloud Functions + Firestore for usage tracking  
> D) Cloud Load Balancing + Cloud Armor rate limiting  
>
> **Answer**: A  
> **Rationale**: Cloud Endpoints natively supports API keys, quota management per key, and usage logging for billing. Apigee is overkill for simple tiering; Functions+FIRESTORE requires custom development; Load Balancer+Armor lacks API-key granularity [[56]][[61]].

---

## Practice Scenarios & Questions

### Scenario 1: Multi-Layer Security Architecture
> **Company**: Global e-commerce platform  
> **Requirements**:  
> - Protect against DDoS and OWASP Top 10 attacks  
> - Secure admin portal with employee-only access  
> - Store payment API keys securely with rotation  
> - Monitor for misconfigurations across 200+ projects  
> - Ensure AI product recommendations don't leak PII  
>
> **Task**: Design a security architecture using GCP services.
>
> **Sample Answer Framework**:
> ```
> Edge Layer: 
>   - Cloud Armor (global external LB backend service) 
>     • Edge policy: Geo-blocking + OWASP rules 
>     • Backend policy: Custom rules for checkout API
>
> Access Control:
>   - IAP for admin portal (Cloud Run backend)
>     • Context-aware policy: data-team group + managed devices
>
> Secrets Management:
>   - Secret Manager (regional us-central1)
>     • Payment keys: version-pinned, rotated quarterly
>     • IAM: payment-service SA + secret-level bindings
>
> Monitoring:
>   - Security Command Center Premium
>     • CIS benchmark monitoring across org
>     • Export findings to BigQuery for SIEM
>
> AI Security:
>   - Model Armor for recommendation engine
>     • Input: SDP for PII redaction
>     • Output: Responsible AI filters + competitor blocking
> ```

### Scenario 2: Compliance-Driven Design
> **Company**: Healthcare provider (HIPAA requirements)  
> **Challenge**: Secure patient data API with strict audit requirements  
>
> **Exam-Style Question**:  
> Which combination ensures HIPAA compliance for secret storage AND access auditing?
>
> A) Secret Manager global replication + Cloud Audit Logs (Admin Activity)  
> B) Secret Manager regional (us-east1) + Data Access Audit Logs enabled at org level  
> C) Compute Engine encrypted disks + VPC Flow Logs  
> D) Cloud KMS with customer-managed keys + SCC Standard  
>
> **Answer**: B  
> **Rationale**: Regional secrets satisfy data residency; Data Access logs capture `AccessSecretVersion` events required for HIPAA audit trails. Admin Activity logs don't capture secret access; C and D don't address secret management specifically [[20]][[28]].

---

## Quick Reference Tables

### Security Service Decision Matrix

| Requirement | Primary Service | Secondary/Complementary |
|------------|----------------|------------------------|
| Block DDoS/WAF attacks | Cloud Armor | Cloud CDN (for caching) |
| Secure API access | Cloud Endpoints | IAP (for user auth) |
| Store credentials | Secret Manager | Cloud KMS (for encryption keys) |
| User identity verification | Cloud IAP | Cloud Identity (for user management) |
| Monitor security posture | Security Command Center | Cloud Logging + Monitoring |
| Protect AI applications | Model Armor | Secret Manager (for API keys) |
| Prevent data exfiltration | VPC Service Controls | SCC Premium + DLP |

### IAM Roles Cheat Sheet

| Role | Scope | PCA Exam Relevance |
|------|-------|-------------------|
| `roles/compute.securityAdmin` | Project | Manage Cloud Armor policies |
| `roles/secretmanager.admin` | Project/Secret | Full secret lifecycle management |
| `roles/securitycenter.admin` | Org/Folder | Configure SCC Premium features |
| `roles/iap.admin` | Project | Manage IAP access policies |
| `roles/servicemanagement.admin` | Project | Deploy Endpoints configurations |
| `roles/modelarmor.admin` | Project | Configure Model Armor templates |

### Common Exam Keywords → Service Mapping

| Keyword in Question | Likely Service | Why |
|--------------------|---------------|-----|
| "edge security", "DDoS mitigation" | Cloud Armor | Operates at Google Front End |
| "zero-trust access", "no VPN" | Cloud IAP | Application-layer identity verification |
| "rotate API keys", "audit secret access" | Secret Manager | Built-in versioning + audit logging |
| "CIS benchmarks", "compliance monitoring" | SCC Premium | Pre-built compliance frameworks |
| "prompt injection", "AI output safety" | Model Armor | Specialized for LLM security |
| "API quota", "developer portal" | Cloud Endpoints | Native API management features |
| "data residency", "regional secrets" | Secret Manager (regional) | Explicit location control |

---

## Final Exam Day Checklist ✅

### 24 Hours Before
- [ ] Review case study notes (company requirements, constraints)
- [ ] Revisit trade-off frameworks: cost vs security vs complexity
- [ ] Practice 5-10 scenario questions under timed conditions

### Exam Morning
- [ ] Verify testing environment (if online proctored)
- [ ] Have scratch paper ready for architecture diagrams
- [ ] Hydrate and take breaks between sections

### During Exam
- [ ] Read questions TWICE before answering
- [ ] Eliminate obviously wrong answers first
- [ ] Flag questions with "choose TWO" or complex scenarios
- [ ] Trust your preparation - avoid second-guessing

### Post-Exam
- [ ] Note topics that felt challenging for recertification planning
- [ ] Celebrate! 🎉 PCA is one of Google Cloud's most respected certifications

---

> 💡 **Pro Tip**: The PCA exam rewards **architectural thinking** over rote memorization. When in doubt, ask: *"What would a senior cloud architect recommend to balance security, cost, and business needs?"* That mindset is your greatest asset.

*This study guide is based on official Google Cloud documentation and PCA exam objectives as of April 2026. Always verify with the [official exam guide](https://cloud.google.com/certification/cloud-architect) for updates.*

---
*© 2026 PCA Study Materials. For educational purposes only. Not affiliated with Google Cloud.*