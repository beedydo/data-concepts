# Zscaler AEM — Big Picture and Priority Tasks

> Framework built from foundational data concepts. Zscaler AEM-specific implementation details marked ⚠️ — verify with official Zscaler documentation or your team before acting on those points.

---

## What AEM Is Trying to Do

- AEM (Asset Exposure Management) is a **data platform** — its core job is to build a single trusted picture of every asset across your environment and understand its exposure
- "Asset" = anything that has a security posture: devices, users, applications, cloud workloads, services
- "Exposure" = what risks each asset carries: unpatched vulnerabilities, misconfigurations, excessive access, shadow IT
- To do this, AEM must ingest data from multiple security and IT tools, unify that data, and produce accurate, complete records per asset

- **The fundamental problem AEM solves:**
  - Each security tool sees a different partial view of the same asset
  - No single tool has the full picture
  - Without unification, you can't answer: "what is the actual risk posture of this device/user/application?"

---

## Why This Is Hard — The Data Problem

- Each data source:
  - Uses different field names for the same thing
  - Represents the same asset differently (formatting, casing, missing fields)
  - Has different data quality (some fields complete, others sparse)
  - Updates at different frequencies (some real-time, some daily, some weekly)
- Without solving the data problem first, AEM produces unreliable results regardless of how good the detection logic is

---

## The Data Sources in Scope

> ⚠️ The sources below are based on earlier mapping work done in this project. Verify actual onboarded sources with your team and Zscaler AEM documentation.

- **MDM** (e.g. SEED / Microsoft Intune) — device inventory, ownership, compliance state
- **EDR** (e.g. CrowdStrike Falcon via DEEP) — endpoint telemetry, threat detections, process activity
- **Vulnerability Scanner** (e.g. CloudSCAPE, Tenable) — vulnerability findings per asset
- **CNAPP** (e.g. Wiz, Prisma Cloud) — cloud workload posture, misconfiguration findings
- **Cloud Native Tools** (e.g. AWS Security Hub, Azure Security Centre) — cloud asset inventory and alerts
- **ASM** — attack surface discovery, externally visible assets
- **CASB** — SaaS application usage and access
- **Source Code Management** (e.g. SHIP-HATS GitLab) — application/development asset context

---

## The Big Picture — How Everything Connects

```
PRIORITY 1: DATA MODEL
Define what entities AEM tracks and what fields matter
(device, user, application, cloud workload — what properties describe each?)
        ↓
PRIORITY 2: DATA SOURCE EVALUATION  [runs concurrently with Priority 1]
Profile each source — what entities does it contain? what fields? what quality?
Does it add value? Does it fill gaps in the data model?
        ↓
PRIORITY 3: DATA CONTRACTS
For each onboarded source: formalise what it will deliver, in what format,
at what quality, and how often
        ↓
PRIORITY 4: DATA QUALITY + NORMALISATION RULES
Define completeness thresholds, controlled vocabularies,
standardisation rules for each field used in matching
        ↓
PRIORITY 5: ENTITY RESOLUTION + DEDUP RULES
Define matching keys, confidence thresholds, survivorship rules per entity type
        ↓
PRIORITY 6: PIPELINE DESIGN (ETL/ELT, batch vs streaming)
Design how data moves from sources into AEM — where transforms happen,
how often, how raw data is preserved
        ↓
OUTPUT: GOLDEN RECORDS PER ASSET
Unified, trusted, complete record per device / user / application
used for exposure scoring, alerting, reporting
```

---

## Priority Tasks — What to Do, Why, and in What Order

### Priority 1 — Draft the Data Model

- **What:** define entity types AEM tracks, what fields describe each, and how entities relate to each other
- **Why first:** everything else depends on this — you can't evaluate sources, write contracts, or build matching rules without knowing what you're trying to build
- **Minimum viable starting point:**
  - List entity types (device, user, application, cloud workload, etc.)
  - List 5–10 key fields per entity type
  - Mark which fields are strong identifiers vs. descriptive attributes
  - Map relationships (device owned-by user, application runs-on device, etc.)
- **Output:** draft entity-relationship map + field list per entity type
- **Dependency:** nothing — start here on day one

---

### Priority 2 — Profile and Evaluate Each Data Source

- **What:** examine actual sample data from each source — what entities does it contain, what fields, what quality, what coverage
- **Why:** determines which sources are worth onboarding and what value each adds
- **Run concurrently with Priority 1** — data profiling findings feed back into and refine the data model
- **For each source, answer:**
  - What entity types does this source cover?
  - What fields does it have? Are any strong identifiers?
  - How complete are the key fields?
  - How consistent is the formatting?
  - How fresh is the data? How frequently updated?
  - What does this source add that other sources don't?
  - Can its fields be mapped to the data model?
- **Gate before proceeding:** gap analysis and field mapping require a draft data model to compare against — don't attempt these until Priority 1 has at least a rough first pass
- **Output:** per-source evaluation with onboard / defer / reject decision and rationale

---

### Priority 3 — Negotiate Data Contracts Per Source

- **What:** formalise the agreement with each source team on what they will deliver
- **Why:** without a contract, sources can silently change schema or drop quality → pipeline breaks, entity resolution fails, golden records become unreliable
- **Do this before building the integration pipeline** — the contract defines what the pipeline is built to expect
- **Each contract must define:**
  - Schema: what fields, what types, which are required
  - Semantics: what field values mean (controlled vocabulary, definitions)
  - Quality SLAs: completeness thresholds per field, validity rules
  - Delivery frequency and latency
  - Versioning and change notification process
  - Source-side owner / point of contact
- **Output:** signed or agreed data contract document per onboarded source

---

### Priority 4 — Define Data Quality and Normalisation Rules

- **What:** define the rules that transform and validate incoming data before it reaches entity resolution
- **Why:** entity resolution fails silently when data quality is poor — matching rules cannot match what's missing, inconsistently formatted, or inaccurate
- **Rules to define per field used in matching:**
  - **Validity:** what format must the value conform to? (e.g. IP address format, date format)
  - **Completeness:** is this field mandatory? What happens when it's null?
  - **Consistency:** does this field need normalisation? (casing, OS name mapping, environment label mapping)
  - **Source trust ranking:** when two sources provide the same field, which one is authoritative?
- **Output:** normalisation ruleset + quality thresholds per field + source trust ranking per entity type

---

### Priority 5 — Define Entity Resolution and Deduplication Rules

- **What:** define the matching logic that determines when two records from different sources refer to the same real-world entity
- **Why:** without this, AEM has multiple partial records per asset instead of one unified golden record — exposure scoring and reporting are unreliable
- **Rules to define:**
  - **Blocking rules:** how to group candidate pairs (e.g. same serial number prefix, same email domain)
  - **Matching keys:** which fields are used for matching, in what priority order
  - **Confidence thresholds:** what score triggers auto-merge, human review, or keep-separate
  - **Survivorship rules:** which source wins per field when values conflict
  - **Review queue process:** who reviews borderline cases and how
- **Output:** dedup ruleset per entity type + survivorship rules + review process

---

### Priority 6 — Design the Data Pipeline

- **What:** decide how data physically moves from each source into AEM — where transforms happen, at what frequency, how raw data is handled
- **Why:** pipeline design determines data freshness, reprocessing capability, and operational complexity
- **Key decisions:**
  - ETL or ELT? (⚠️ verify what Zscaler AEM supports natively)
    - ELT preferred if raw data must be preserved for audit and rule reprocessing
    - ETL preferred if PII or classified data must be masked before landing
  - Batch or streaming per source?
    - Batch sufficient for: daily inventory syncs, vulnerability scan results, compliance snapshots
    - Streaming only if: real-time asset visibility or event-driven response is a genuine requirement
  - Where does raw data land and who can access it?
- **Output:** pipeline architecture decision per source + ingestion frequency + raw data retention policy

---

## Why Each Step Is Required — Summary

| Step | What breaks if skipped |
|---|---|
| Data model | No target to map sources against. Can't write matching rules. Can't evaluate source value. |
| Source evaluation | Onboard sources that add no value or have too-poor quality to use. Wasted integration effort. |
| Data contracts | Sources change schema silently. Pipeline breaks. Entity resolution produces wrong results. No accountability. |
| Data quality + normalisation | Matching rules fail — same asset stays as multiple records. Wrong values enter golden records. |
| Entity resolution rules | Platform has thousands of partial records per asset. No unified view. Exposure scoring is unreliable. |
| Pipeline design | Data is stale, reprocessing is impossible, or operational complexity is higher than necessary. |

---

## Open Questions to Verify with Your Team / Zscaler Docs

- ⚠️ What entity types does Zscaler AEM natively support? (device, user, application, cloud workload — which are in scope?)
- ⚠️ Does Zscaler AEM have a built-in data model, or does it need to be configured?
- ⚠️ Which data sources does Zscaler AEM support native connectors for vs. requiring custom integration?
- ⚠️ Does Zscaler AEM handle entity resolution internally, or does the team need to configure matching rules?
- ⚠️ What ETL/ELT pattern does Zscaler AEM use for ingestion?
- ⚠️ Does Zscaler AEM preserve raw source data, or only the unified record?
- ⚠️ What data contract or schema agreement process does Zscaler AEM require from source teams?
