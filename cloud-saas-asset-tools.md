# Cloud and SaaS Asset Category — Central Tools Reference

> Sources: `information.txt` + [Singapore Government Developer Portal](https://docs.developer.tech.gov.sg/)
> ✅ = confirmed from official public sources | ⚠️ = uncertain, verify with GovTech/agency contact

---

## Acronym Glossary

| Acronym | Full Name | Description |
|---------|-----------|-------------|
| **MDM** | Mobile Device Management | Manages, secures, and enforces policies on endpoint devices |
| **EDR** | Endpoint Detection and Response | Monitors endpoints for threats and enables incident response |
| **Vuln Scanner** | Vulnerability Scanner | Identifies known CVEs and misconfigurations in systems and apps |
| **ASM** | Attack Surface Management | Discovers and monitors internet-facing assets for exposure and risk |
| **GASSP** | Unknown ⚠️ | Acronym not confirmed in any public GovTech documentation — likely an internal/restricted GovTech platform; verify with GovTech or your agency |
| **CNAPP** | Cloud Native Application Protection Platform | Unified security platform covering CSPM, CWPP, and container security |
| **CASB** | Cloud Access Security Broker | Security control point between users and cloud/SaaS services; enforces DLP and access policies |
| **SGTS** | Singapore Government Technology Stack | Suite of central digital and data tools for Singapore government agencies |
| **GCC** | Government on Commercial Cloud | Singapore government programme for agencies to host workloads on commercial cloud (AWS, Azure, GCP) ✅ |
| **SEED** | Security Suite for Engineering Endpoint Devices | GovTech's MDM + endpoint security platform; mandatory for accessing SGTS and GCC ✅ |
| **DEEP** | Development Environment Endpoint Posture | The endpoint-layer component of SEED (Intune + CrowdStrike + Tanium) ✅ |
| **SHIP-HATS** | Secure Hybrid Integrated Pipeline — Hosting, Automated Testing & Security | GovTech's centralised CI/CD and DevSecOps platform ✅ |
| **Srs Code Mgmt** | Source Code Management | Version control and repository hosting — GitLab under SHIP-HATS ✅ |
| **CStack** | Container Stack | GovTech's Kubernetes-based managed container platform ✅ |
| **CloudSCAPE** | Cloud Security and Compliance Automation Platform Ecosystem | GovTech platform for automated daily IM8 compliance scanning on GCC 2.0 ✅ |
| **CodeSCAPE** | Code Security and Compliance Automation Platform Ecosystem | GovTech platform for automated daily security and compliance scanning of SHIP-HATS GitLab projects ✅ |
| **TechPass** | TechPass | SSO/IAM platform for all SGTS products; uses Azure AD, OAuth 2.0, OIDC, SAML 2.0 ✅ |
| **GCSOC** | Government Cyber Security Operations Centre | 24/7 WOG threat detection, monitoring and response centre ✅ |
| **WOG** | Whole-of-Government | Initiatives or tools shared across all Singapore government agencies |
| **IM8** | Instruction Manual 8 | Singapore government's ICT security policy and compliance framework (being reformed to ICT&SS Control Catalog) |
| **CIS** | Center for Internet Security | International benchmarks for hardening OS and cloud configurations |
| **ZTNA** | Zero Trust Network Access | Access model that verifies identity and device posture before granting network access |
| **SLSA** | Supply-chain Levels for Software Artifacts | Framework for software supply chain security and provenance attestation |
| **DORA** | DevOps Research and Assessment | Four key metrics measuring software delivery performance |
| **SAST** | Static Application Security Testing | Scans source code for vulnerabilities without running the app |
| **DAST** | Dynamic Application Security Testing | Tests running applications for vulnerabilities |
| **MAST** | Mobile Application Security Testing | Security testing specific to mobile applications |
| **SCA** | Software Composition Analysis | Scans open-source dependencies for vulnerabilities and licence risks |
| **XCA** | Extended Code Analysis | GovTech-custom SAST scanner with Singapore Government-specific rules, auto-enabled in SHIP-HATS ✅ |

---

## Asset Category Structure

```
Cloud Asset
├── Compute
│   ├── System      → MDM, EDR, Vuln Scanner, ASM, GASSP, CNAPP, Cloud Native Tools
│   ├── Application → MDM, Vuln Scanner, ASM, CNAPP, Cloud Native Tools
│   └── Development → Src Code Mgmt, CNAPP
├── Infra
│   ├── System      → Vuln Scanner, ASM, GASSP
│   └── Application → Vuln Scanner, ASM
└── Native Service
    └── Application → CNAPP, Cloud Native Tools

SaaS Asset
└── Native Service
    └── Application → CASB
```

---

## Cloud Asset — Compute

### System Sub-category

Scope: virtual machines, operating systems, servers hosted on cloud (GCC/AWS/Azure/GCP).

#### MDM — Mobile Device Management
- **Central tool: SEED** (Security Suite for Engineering Endpoint Devices) ✅
- SEED is a three-layer platform:
  - **TechPass** (identity layer) — SSO via Azure AD, OAuth 2.0, OpenID Connect, SAML 2.0
  - **Cloudflare** (network layer) — Cloudflare One (WARP client, replaces VPN) + Cloudflare Gateway (DNS/web filtering) + Cloudflare Access (zero-trust app access)
  - **DEEP** (endpoint posture layer) — Microsoft Intune (MDM) + CrowdStrike Falcon (EDR/vuln mgmt) + Tanium (asset and posture management)
- Cloudflare Access uses Tanium posture data for conditional access decisions — device must pass health checks before accessing SGTS/GCC resources (ZTNA)
- Mandatory prerequisite for accessing all SGTS products and GCC 2.0

> **Sources:**
> - SEED architecture (TechPass / Cloudflare / DEEP / Intune / CrowdStrike / Tanium): [developer.tech.gov.sg — SEED Features & Roadmap](https://www.developer.tech.gov.sg/products/categories/cybersecurity/seed/features-roadmap)
> - SEED as MDM platform for SGTS/GCC access: [docs.developer.tech.gov.sg — SEED product page](https://docs.developer.tech.gov.sg/docs?product=Security%20Suite%20for%20Engineering%20Endpoint%20Devices%20%28SEED%29)
> - SEED mandatory for GCC 2.0: [developer.tech.gov.sg — GCC Features & Roadmap](https://www.developer.tech.gov.sg/products/categories/infrastructure-and-hosting/gcc/features-roadmap)

#### EDR — Endpoint Detection and Response
- **Central tool (developer endpoints): CrowdStrike Falcon** (within SEED/DEEP) ✅
  - Used for vulnerability management and threat detection on developer/engineer devices
- **Central tool (server/workload endpoints): Cerberus** ✅
  - Full name: Centralised Endpoint Protection and Threat Detection/Response platform
  - Designed for WOG server workloads (not developer laptops)
  - Integrates with GCSOC (Government Cyber Security Operations Centre)
  - Subscription-based model for agencies

> **Sources:**
> - CrowdStrike Falcon as EDR within SEED/DEEP: [developer.tech.gov.sg — SEED Features & Roadmap](https://www.developer.tech.gov.sg/products/categories/cybersecurity/seed/features-roadmap)
> - Cerberus as WOG server EPP-EDR integrating with GCSOC: [docs.developer.tech.gov.sg — Cerberus product page](https://docs.developer.tech.gov.sg/docs?product=Cerberus) ⚠️ full details login-restricted

#### Vuln Scanner — Vulnerability Scanner
- Within SHIP-HATS CI/CD pipeline (code/app layer):
  - **Fortify on Demand (FoD) SAST** — static analysis ✅
  - **GitLab SAST** (native, part of GitLab Ultimate) ✅
  - **XCA** (Extended Code Analysis) — GovTech-custom SAST, auto-enabled for all SHIP-HATS tenants ✅
  - **SonarQube** (Community and Enterprise editions) — code quality + security ✅
  - **Nexus IQ** — SCA for open-source dependency vulnerabilities ✅
- Cloud infrastructure layer: **CloudSCAPE** (automated IM8 compliance scanning on GCC 2.0) ✅
- Dedicated infrastructure scanning tool (e.g., Tenable, Qualys): ⚠️ not confirmed in public GovTech documentation

> **Sources:**
> - Fortify on Demand, GitLab SAST, XCA, SonarQube, Nexus IQ in SHIP-HATS: [docs.developer.tech.gov.sg — SHIP-HATS Tools (raw doc)](https://docs.developer.tech.gov.sg/docs/ship-hats-docs/tools/ship-hats-tools.md)
> - SHIP-HATS product page: [docs.developer.tech.gov.sg — SHIP-HATS](https://docs.developer.tech.gov.sg/docs?product=SHIP-HATS)
> - CloudSCAPE automated compliance scanning, IM8, AWS/Azure, 70 agencies: [developer.tech.gov.sg — CloudSCAPE Overview](https://www.developer.tech.gov.sg/products/categories/cybersecurity/cloudscape/overview)
> - Dedicated infra vuln scanner: ⚠️ no public .gov.sg source found

#### ASM — Attack Surface Management
- No dedicated central ASM tool confirmed in public documentation ⚠️
- **CloudSCAPE** provides some adjacent capability: consolidates multi-cloud (AWS + Azure) security posture data across agencies and monitors baseline violations
- Full ASM capability may exist within **GCSOC** tooling (not publicly documented)
- Common commercial tools in this space: Tenable ASM, Palo Alto Cortex Xpanse, Microsoft Defender EASM ⚠️ verify central tool with GovTech

> **Sources:**
> - CloudSCAPE multi-cloud data consolidation (adjacent capability): [developer.tech.gov.sg — CloudSCAPE Overview](https://www.developer.tech.gov.sg/products/categories/cybersecurity/cloudscape/overview)
> - GCSOC listed as WOG cybersecurity product: [docs.developer.tech.gov.sg — Cybersecurity products](https://docs.developer.tech.gov.sg/docs?product=Cybersecurity)
> - Dedicated central ASM tool: ⚠️ no public .gov.sg source found — verify with GovTech

#### GASSP ⚠️
- Acronym not found in any public GovTech documentation
- Possible interpretations: Government Application Security Scanning Platform, Government Automated Security Scanning Platform
- Likely an internal/restricted GovTech tool — verify full name, scope, and responsible team directly with GovTech or your agency's CISO

> **Sources:**
> - ⚠️ No public .gov.sg source found for GASSP — searched docs.developer.tech.gov.sg, developer.tech.gov.sg, and GovTechSG GitHub. Acronym does not appear in any public documentation.

#### CNAPP — Cloud Native Application Protection Platform
- **CloudSCAPE** serves an adjacent CNAPP function for GCC 2.0 (cloud posture management, IM8 compliance) ✅
- Not explicitly self-described as CNAPP in GovTech documentation
- Commercial CNAPP tools (Wiz, Prisma Cloud, Microsoft Defender for Cloud): ⚠️ Prisma Cloud was previously in SHIP-HATS but was decommissioned; verify current central CNAPP tool with GovTech

> **Sources:**
> - CloudSCAPE as cloud posture/compliance tool on GCC 2.0: [developer.tech.gov.sg — CloudSCAPE Overview](https://www.developer.tech.gov.sg/products/categories/cybersecurity/cloudscape/overview)
> - GCC 2.0 CSP accounts auto-onboarded to CloudSCAPE: [developer.tech.gov.sg — GCC Features & Roadmap](https://www.developer.tech.gov.sg/products/categories/infrastructure-and-hosting/gcc/features-roadmap)
> - Prisma Cloud decommissioned from SHIP-HATS, replaced by GitLab Container Scanning: [docs.developer.tech.gov.sg — SHIP-HATS Tools (raw doc)](https://docs.developer.tech.gov.sg/docs/ship-hats-docs/tools/ship-hats-tools.md)
> - Dedicated central CNAPP tool: ⚠️ no public .gov.sg source found — verify with GovTech

#### Cloud Native Tools
- Provider-native security services available within GCC 2.0:
  - **AWS**: AWS Config (continuous compliance), AWS Security Hub, AWS GuardDuty, AWS Inspector ✅
  - **Azure**: Microsoft Defender for Cloud, Azure Security Centre ✅
  - **GCP**: Google Security Command Centre ✅ (GCP support on GCC from Jul 2023)
- GCC 2.0 uses Policy-as-Code (PaC) and AWS Config (and equivalents) for continuous security compliance across all workloads ✅

> **Sources:**
> - AWS, Azure, GCP as approved GCC cloud providers: [developer.tech.gov.sg — GCC Overview](https://www.developer.tech.gov.sg/products/categories/infrastructure-and-hosting/gcc/overview)
> - Policy-as-Code (PaC) on GCC 2.0, Cloudflare Access Control: [developer.tech.gov.sg — GCC Features & Roadmap](https://www.developer.tech.gov.sg/products/categories/infrastructure-and-hosting/gcc/features-roadmap)
> - Specific AWS/Azure/GCP native tool names (Security Hub, GuardDuty, etc.): ⚠️ standard cloud provider documentation — not cited from a .gov.sg source; GCC docs confirm the providers but do not enumerate every native tool by name

---

### Application Sub-category

Scope: applications and workloads running on cloud compute (containers, app servers, microservices).

- **MDM** — same as System; SEED ensures device compliance for developers accessing cloud applications

  > **Sources:**
  > - SEED as mandatory access gate for SGTS/GCC: [developer.tech.gov.sg — SEED Features & Roadmap](https://www.developer.tech.gov.sg/products/categories/cybersecurity/seed/features-roadmap)
  > - SEED docs portal: [docs.developer.tech.gov.sg — SEED](https://docs.developer.tech.gov.sg/docs?product=Security%20Suite%20for%20Engineering%20Endpoint%20Devices%20%28SEED%29)

- **Vuln Scanner** — SHIP-HATS pipeline tools scan application-layer code and containers:
  - Container scanning: **GitLab Container Scanning** ✅ (replaced Prisma Cloud, which was decommissioned)
  - DAST: **Fortify on Demand DAST** + **GitLab DAST** ✅
  - MAST (mobile): **Fortify on Demand MAST** ✅

  > **Sources:**
  > - GitLab Container Scanning, Fortify on Demand DAST/MAST, GitLab DAST in SHIP-HATS: [docs.developer.tech.gov.sg — SHIP-HATS Tools (raw doc)](https://docs.developer.tech.gov.sg/docs/ship-hats-docs/tools/ship-hats-tools.md)
  > - Prisma Cloud decommissioned, replaced by GitLab Container Scanning: [docs.developer.tech.gov.sg — SHIP-HATS Tools (raw doc)](https://docs.developer.tech.gov.sg/docs/ship-hats-docs/tools/ship-hats-tools.md)

- **ASM** — monitors external-facing application endpoints; no confirmed central tool (see System > ASM above)

  > **Sources:**
  > - ⚠️ No public .gov.sg source found for a dedicated central ASM tool — verify with GovTech

- **CNAPP** — CloudSCAPE extends posture visibility to cloud workloads; container security via GitLab Container Scanning

  > **Sources:**
  > - CloudSCAPE posture visibility on GCC 2.0 workloads: [developer.tech.gov.sg — CloudSCAPE Overview](https://www.developer.tech.gov.sg/products/categories/cybersecurity/cloudscape/overview)
  > - GitLab Container Scanning as container security in pipeline: [docs.developer.tech.gov.sg — SHIP-HATS Tools (raw doc)](https://docs.developer.tech.gov.sg/docs/ship-hats-docs/tools/ship-hats-tools.md)

- **Cloud Native Tools** — provider-native runtime security (e.g., AWS GuardDuty for anomaly detection on running workloads)

  > **Sources:**
  > - AWS, Azure, GCP as approved GCC providers: [developer.tech.gov.sg — GCC Overview](https://www.developer.tech.gov.sg/products/categories/infrastructure-and-hosting/gcc/overview)
  > - Specific native tool names (GuardDuty, etc.): ⚠️ standard cloud provider documentation — GCC docs confirm providers but do not enumerate every native tool by name

---

### Development Sub-category

Scope: developer tooling and pipelines used to build and deploy cloud applications.

#### Src Code Mgmt — Source Code Management
- **Central tool: GitLab Dedicated Ultimate (SaaS)** ✅
  - Hosted in Singapore on a dedicated instance at `sgts.gitlab-dedicated.com`
  - Managed by GitLab; login via TechPass
  - Full SCM: repositories, merge requests, code review, branch management, GitLab Flow
- Part of **SHIP-HATS** (Secure Hybrid Integrated Pipeline — Hosting, Automated Testing & Security)
- SHIP-HATS full toolchain:

| Category | Tools |
|---|---|
| Source Code Mgmt | GitLab Dedicated Ultimate ✅ |
| CI/CD pipeline | GitLab CI/CD (pipeline templates, shared runners, SLSA provenance, DORA metrics) ✅ |
| Issue/project tracking | Jira Cloud, Confluence Cloud, Jira Service Management (JSM) Enterprise ✅ |
| Artifact management | Nexus Repository Pro (Maven, npm, Docker, PyPI), GitLab Package Registry ✅ |
| SAST | GitLab SAST, Fortify on Demand SAST, XCA (GovTech-custom rules) ✅ |
| DAST | GitLab DAST, Fortify on Demand DAST ✅ |
| MAST | Fortify on Demand MAST ✅ |
| SCA / dependency | GitLab Dependency Scanning, Nexus IQ ✅ |
| Container scanning | GitLab Container Scanning ✅ (Prisma Cloud decommissioned) |
| IaC scanning | GitLab IaC Scanner, CloudSCAPE ✅ |
| Code quality | SonarQube (Community + Enterprise editions) ✅ |
| Mobile testing | pCloudy (Android + iOS real devices) ✅ |
| Security dashboard | GitLab Security Dashboard, Unified Quality Dashboard (UQD) ✅ |
| Compliance monitoring | CodeSCAPE (daily automated scans, IM8/SLSA/OWASP compliance) ✅ |

- Portal: [SHIP-HATS](https://docs.developer.tech.gov.sg/docs?product=SHIP-HATS)

#### CNAPP (in Development context)
- **CodeSCAPE** (Code Security and Compliance Automation Platform Ecosystem) ✅
  - GovTech-built governance layer on top of SHIP-HATS GitLab
  - Scans all SHIP-HATS GitLab projects **daily** for security and compliance posture
  - Tracks compliance against IM8 DevSecOps policies, CNCF, SLSA, and OWASP
  - Provides DORA metrics (delivery velocity and quality)
  - Currently in pilot — agencies can request access
  - Portal: [CodeSCAPE](https://docs.developer.tech.gov.sg/docs?product=Code%20Security%20and%20Compliance%20Automation%20Platform%20Ecosystem%20%28CodeSCAPE%29)
- GitLab Container Scanning: scans container images during CI/CD build before deployment

---

## Cloud Asset — Infra

Scope: networking, load balancers, databases, storage, DNS — underlying cloud infrastructure (not compute VMs).

### System Sub-category

- **Vuln Scanner** — scans infrastructure components (network configs, managed DB servers, storage) for CVEs and misconfigurations
  - CloudSCAPE handles infrastructure-layer IM8 compliance scanning on GCC 2.0 ✅
  - Dedicated infra vuln scanner (Tenable, Qualys): ⚠️ not confirmed in public docs

  > **Sources:**
  > - CloudSCAPE infra compliance scanning on GCC 2.0: [developer.tech.gov.sg — CloudSCAPE Overview](https://www.developer.tech.gov.sg/products/categories/cybersecurity/cloudscape/overview)
  > - Dedicated infra vuln scanner: ⚠️ no public .gov.sg source found

- **ASM** — discovers exposed infrastructure endpoints (open cloud storage buckets, exposed DB ports, unprotected APIs)
  - No confirmed central ASM tool; may be within GCSOC ⚠️

  > **Sources:**
  > - ⚠️ No public .gov.sg source found for a dedicated central ASM tool — verify with GovTech or agency CISO

- **GASSP** ⚠️ — applied at infrastructure layer; acronym and scope unconfirmed publicly

  > **Sources:**
  > - ⚠️ No public .gov.sg source found for GASSP — acronym not present in any publicly accessible GovTech documentation

### Application Sub-category

- **Vuln Scanner** — scans applications deployed on infrastructure (API gateways, managed DBs) for vulnerabilities

  > **Sources:**
  > - CloudSCAPE as compliance scanning tool covering GCC 2.0 infrastructure: [developer.tech.gov.sg — CloudSCAPE Overview](https://www.developer.tech.gov.sg/products/categories/cybersecurity/cloudscape/overview)
  > - Dedicated infra-app vuln scanner: ⚠️ no public .gov.sg source found — verify with GovTech

- **ASM** — monitors application-facing infrastructure surfaces for attack exposure

  > **Sources:**
  > - ⚠️ No public .gov.sg source found for a dedicated central ASM tool — verify with GovTech or agency CISO

---

## Cloud Asset — Native Service

Scope: fully managed cloud services consumed as-is from providers (e.g., AWS RDS, Azure Blob, managed Kubernetes).

### Application Sub-category

- **CNAPP**
  - CloudSCAPE detects misconfigurations in managed/native cloud services (e.g., public S3 buckets, unencrypted RDS) ✅
  - AWS Config, Azure Policy enforce compliance on native service usage within GCC 2.0 ✅

  > **Sources:**
  > - CloudSCAPE consolidated view across AWS and Azure accounts: [developer.tech.gov.sg — CloudSCAPE Overview](https://www.developer.tech.gov.sg/products/categories/cybersecurity/cloudscape/overview)
  > - GCC 2.0 CSP accounts auto-onboarded to CloudSCAPE; Policy-as-Code compliance check on all resources: [developer.tech.gov.sg — GCC Features & Roadmap](https://www.developer.tech.gov.sg/products/categories/infrastructure-and-hosting/gcc/features-roadmap)

- **Cloud Native Tools**
  - AWS: AWS Config, AWS Security Hub
  - Azure: Azure Policy, Microsoft Defender for Cloud
  - GCP: Google Security Command Centre
  - All operate within GCC 2.0's Policy-as-Code framework ✅

  > **Sources:**
  > - AWS, Azure, GCP as approved GCC providers: [developer.tech.gov.sg — GCC Overview](https://www.developer.tech.gov.sg/products/categories/infrastructure-and-hosting/gcc/overview)
  > - Policy-as-Code (PaC) applied to all GCC 2.0 resources: [developer.tech.gov.sg — GCC Features & Roadmap](https://www.developer.tech.gov.sg/products/categories/infrastructure-and-hosting/gcc/features-roadmap)
  > - Specific native tool names (AWS Config, Azure Policy, etc.): ⚠️ standard cloud provider documentation — GCC docs confirm providers but do not enumerate every native tool by name

---

## SaaS Asset — Native Service

Scope: SaaS applications consumed as external cloud services by government agencies (e.g., Microsoft 365, Salesforce, collaboration tools).

### Application Sub-category

#### CASB — Cloud Access Security Broker
- Sits between users and SaaS applications; enforces security policies for SaaS access
- Functions:
  - Visibility into SaaS app usage (including shadow IT discovery)
  - Data Loss Prevention (DLP) for data moving to/from SaaS
  - Threat protection — detects compromised accounts and malware via SaaS file sync
  - Compliance enforcement aligned with IM8
- **COE for SaaS** — GovTech's Centre of Excellence for SaaS; includes WOG Salesforce Knowledge Base ✅
  - Portal: [COE for SaaS](https://docs.developer.tech.gov.sg/docs?product=Centre%20of%20Excellence%20%28COE%29%20for%20SaaS)
- Central CASB tool: ⚠️ not confirmed in public documentation
  - For WOG Microsoft 365, **Microsoft Defender for Cloud Apps** is the most likely CASB (integrates natively with M365 and Cloudflare Access which SEED already uses)
  - Other common tools: Netskope, Zscaler — verify with GovTech

> **Sources:**
> - COE for SaaS product listing (WOG Salesforce Knowledge Base): [docs.developer.tech.gov.sg — COE for SaaS](https://docs.developer.tech.gov.sg/docs?product=Centre%20of%20Excellence%20%28COE%29%20for%20SaaS)
> - Central CASB tool: ⚠️ no public .gov.sg source found — verify with GovTech
> - "Most likely Microsoft Defender for Cloud Apps": ⚠️ inference based on WOG M365 adoption and SEED using Cloudflare — not stated in any .gov.sg source

---

## GCC Overview (Context for All Cloud Asset Categories)

| Attribute | Detail |
|---|---|
| **Cloud providers** | AWS ✅ (from May 2022), Azure ✅ (from Nov 2022), GCP ✅ (from Jul 2023) |
| **GCC 1.0 vs 2.0** | 2.0 rebuilt from scratch; replaced VPNs/Jumphosts with Cloudflare ZTNA; introduced PaC and CloudSCAPE |
| **Mandatory prerequisites** | SEED (device compliance) + TechPass (identity) for all GCC 2.0 access |
| **GCC+** | Separate tier for higher-sensitivity Confidential systems; details not publicly documented |
| **Compliance standard** | IM8 (Instruction Manual 8); GCC Standard Build Images are IM8 + CIS hardened |
| **Compliance scanning** | CloudSCAPE — daily automated IM8 compliance scans (70 agencies, 839 MAU, 99.9% uptime) ✅ |

---

## Key GovTech Central Tools Summary

| Tool | Type | Scope |
|------|------|-------|
| **SEED** | MDM + EDR + ZTNA | Developer/engineer endpoints; mandatory GCC/SGTS access gate |
| **DEEP** (within SEED) | Endpoint posture | Intune (MDM) + CrowdStrike Falcon (EDR) + Tanium (asset/posture) |
| **Cloudflare** (within SEED) | Network ZTNA | Replaces VPN; Cloudflare One + Gateway + Access |
| **TechPass** | SSO/IAM | Single sign-on for all SGTS products |
| **GCC** | Cloud platform | AWS, Azure, GCP hosting for government workloads |
| **SHIP-HATS** | CI/CD + DevSecOps | GitLab + Jira + Nexus + Fortify + SonarQube + more |
| **CloudSCAPE** | Cloud compliance | Daily IM8 scanning on GCC 2.0 (AWS + Azure) |
| **CodeSCAPE** | Code compliance | Daily DevSecOps compliance scanning on SHIP-HATS GitLab projects |
| **Cerberus** | EPP-EDR | Endpoint protection for WOG server workloads; integrates with GCSOC |
| **CStack** | Container platform | Kubernetes-based managed container hosting on GCC |
| **GCSOC** | SOC | 24/7 WOG threat monitoring and incident response |
| **COE for SaaS** | SaaS governance | Centre of excellence for SaaS adoption (incl. Salesforce) |

---

## Items to Verify with GovTech / Agency

1. **GASSP** — full acronym, scope, and responsible team (not in any public documentation)
2. **EDR for cloud workloads** — confirm whether Cerberus or another tool covers cloud VM/server EDR centrally
3. **Vuln Scanner for infra** — confirm if Tenable/Qualys or another tool is centrally mandated beyond SHIP-HATS/CloudSCAPE
4. **ASM central tool** — confirm if a dedicated ASM tool exists or if ASM capability lives within GCSOC
5. **CNAPP** — confirm if there is a central CNAPP tool beyond CloudSCAPE (e.g., Wiz, Prisma, Defender for Cloud)
6. **CASB** — confirm the central CASB tool for WOG SaaS access
7. **GCC+ details** — architecture and tooling for higher-sensitivity systems
