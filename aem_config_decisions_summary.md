# AEM Configuration Decisions — Foundational Concepts Summary

---

## 1. Entity Resolution and Deduplication

**What entity resolution is:**
- Process of determining whether records from different sources refer to the same real-world entity (e.g. same device, same user)
- AEM ingests data from multiple tools — each may describe the same entity differently, partially, or inconsistently
- Entity resolution combines those records into one trusted unified record (the golden record)

**How deduplication rules work in practice:**
- **Blocking** — group records into candidate pairs before comparing (avoids comparing every record against every other)
- **Matching** — score similarity between candidate pairs using field-level rules
  - Strong identifiers (serial number, cloud instance ID) → high confidence match
  - Weak identifiers (hostname, IP address) → used as supporting signals only
- **Thresholding** — act on the confidence score:
  - High confidence → auto-merge
  - Borderline → route to human review queue
  - Low confidence → keep as separate records
- **Survivorship** — when two matched records have conflicting field values, a rule decides which value becomes the authoritative one (e.g. most recent, most trusted source, non-null preference)

---

## 2. Three Core Data Quality Dimensions and Field Configuration

**Accuracy**
- Definition: field value correctly reflects reality
- Applies to field config: determines which source to trust for each field
- Example: endpoint tool is most accurate for OS version; HR system is most accurate for device owner

**Completeness**
- Definition: required fields are populated — no missing or null values where data should exist
- Applies to field config: determines which fields are mandatory at ingestion, which trigger enrichment or review when missing
- Example: a device record with no owner and no serial number cannot be used for operations or matching

**Consistency**
- Definition: same entity represented the same way across all sources
- Applies to field config: determines which fields need controlled vocabularies and normalisation rules
- Example: `win11`, `Windows 11`, `Win 11` must all map to one standard value before matching rules can work

**How all three connect:**
- Poor consistency → matching fails (same device looks like two different records)
- Poor completeness → matching fails (can't match on a field that doesn't exist)
- Poor accuracy → survivorship picks the wrong value (golden record is built correctly but contains wrong data)

---

## 3. ETL vs ELT — Conceptual Difference and When to Use Each

**ETL (Extract → Transform → Load)**
- Transformation happens before data lands in the destination
- Destination receives clean, pre-processed data
- Use when:
  - Target system has limited compute
  - Data must be masked or scrubbed before storage (PII, compliance)
  - Only processed data needs to be stored

**ELT (Extract → Load → Transform)**
- Raw data lands in destination first; transformation runs inside the destination
- Use when:
  - Target is a powerful cloud data warehouse (BigQuery, Snowflake, Redshift)
  - Raw data must be preserved for audit, reprocessing, or multiple downstream consumers
  - Transformation logic is likely to evolve — re-run transforms without re-extracting from source

**Key difference:**
- ETL: control before ingest, less flexibility after
- ELT: flexibility after ingest, raw data preserved

**AEM context:**
- AEM likely uses ELT — raw agency data lands first, entity resolution and quality transforms run inside the platform, raw data retained for audit and rule reprocessing
