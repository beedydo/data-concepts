# AEM Foundational Concepts: Entity Resolution, Data Quality, ETL vs ELT

---

## 1. Entity Resolution & Deduplication

**What it is:** Process of determining whether two or more records refer to same real-world entity (person, org, device) across different data sources.

**Why it matters in AEM:** Data platform ingests from multiple sources — same person may appear as "John Tan", "J. Tan", "John TAN" across CRM, HR system, ticketing tool. Without ER, same entity = multiple records = inflated counts, broken joins, wrong analytics.

### How Deduplication Rules Work

| Step | What Happens |
|------|-------------|
| **Blocking** | Narrow candidate pairs to compare. Don't compare all-vs-all (O(n²)). Group by phonetic name hash, email domain, etc. |
| **Matching** | Score similarity between candidate pairs. Rules-based (exact match on NRIC) or ML-based (probabilistic score on name + DOB + address combo) |
| **Thresholding** | Score ≥ X → auto-merge. Score in [Y, X] → human review queue. Score < Y → treat as distinct |
| **Survivorship** | When merging, which field value wins? Most recent? Most complete? Most trusted source? |

**Key distinction:** Exact match dedup (same email = same person) vs. fuzzy match dedup (similar name + same org = probably same person). Production systems combine both.

**Practical rule example:**

```
IF email.exact_match → MATCH (confidence: 1.0)
ELSE IF (name.similarity > 0.85 AND org.exact_match AND phone.exact_match) → MATCH (confidence: 0.9)
ELSE IF (name.similarity > 0.9 AND dob.exact_match) → REVIEW (confidence: 0.7)
ELSE → NO MATCH
```

---

## 2. Three Core Data Quality Dimensions

### Accuracy

**Definition:** Field value reflects ground truth in the real world.

- Wrong value stored even if format is valid
- Example: Age = 150, postal code points to wrong district, email typo passes format check
- Checked via: reference data lookups, cross-field validation, domain expert review
- Hard to automate fully — often requires external authoritative source

### Completeness

**Definition:** Required fields are populated; no missing/null values where data should exist.

- Example: Phone number column 60% null means platform can't do SMS-based workflows
- Measured as: `(non-null count / total record count) × 100%`
- Tiers matter: mandatory fields (completeness = 100% enforced) vs. optional fields (tracked but not blocked)
- Incomplete ≠ inaccurate — a present value can be wrong; an absent value is simply missing

### Consistency

**Definition:** Same entity represented the same way across systems and time.

- Example: DOB stored as `DD/MM/YYYY` in system A, `YYYY-MM-DD` in system B → same person, inconsistent format
- Or: Gender = "M" in one record, "Male" in another for same person
- Two types:
  - **Cross-system consistency:** Value matches across data sources
  - **Temporal consistency:** Value doesn't change illogically over time (e.g., age decreases)
- Most relevant to AEM entity resolution — inconsistency across sources is root cause of most dedup failures

### How They Interact

```
Accurate but incomplete:  "John Tan, NRIC: S1234567A" — correct but no contact info
Complete but inaccurate:  all fields filled, phone number is wrong
Consistent but inaccurate: same wrong value across all systems

All three needed for reliable entity resolution.
```

---

## 3. ETL vs ELT

**Core difference:** Where transformation happens.

```
ETL: Extract → Transform → Load
     [Source] → [Transform in pipeline] → [Target/DW]

ELT: Extract → Load → Transform
     [Source] → [Load raw into DW/lake] → [Transform inside DW]
```

### ETL

- Transformation happens **before** data lands in destination
- Destination receives clean, structured data
- Pipeline (e.g., Spark job, custom script) does the work
- **When appropriate:**
  - Destination has limited compute (legacy data warehouses, relational DBs)
  - Data must be cleaned/masked before storage (PII, compliance)
  - Source data very messy — raw form useless to store
  - Low storage budget — only store what's needed

### ELT

- Raw data lands first; transform happens **inside** destination using its compute
- **When appropriate:**
  - Destination is cloud DW with cheap compute (BigQuery, Snowflake, Redshift, Azure Synapse)
  - Need to preserve raw data for reprocessing / schema evolution
  - Multiple downstream consumers need different transforms of same raw data
  - Exploration-first: analysts run ad-hoc transforms before standardizing

### Comparison Table

| Factor | ETL Wins | ELT Wins |
|--------|----------|----------|
| Destination compute | Weak | Strong/elastic |
| PII/compliance | Must mask before store | Less critical |
| Schema stability | Known upfront | Evolving |
| Reprocessing needs | Low | High |
| Storage cost | Sensitive | Cheap |

### AEM Context

AEM as data platform likely uses **ELT pattern** — ingest raw entity data from agency sources, store in lake/DW layer, apply entity resolution and data quality transforms inside platform compute. Raw data preserved for audit trail and reprocessing when dedup rules change.

---

## TL;DR — How All Three Connect

Entity resolution quality depends on data quality. Completeness and consistency problems are top reasons dedup rules fail:

- Can't match what's **missing** (completeness)
- Can't match what's **formatted differently** (consistency)
- Can match but produce wrong golden record if values are **inaccurate** (accuracy)

ETL/ELT choice determines whether transforms to fix those quality issues happen before or after ingestion. In a modern platform like AEM, likely ELT — so raw data is preserved and dedup rules can be re-run as they evolve.
