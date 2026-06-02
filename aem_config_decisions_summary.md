# AEM Configuration Decisions — Concept-to-Practice Summary

> Short summary mapping foundational concepts learned to AEM configuration decisions and outstanding questions.
> Full reference: `aem-foundation-data-concepts.md` | Visuals: `aem-foundation-data-concepts-visuals.md`

---

## What AEM Is Doing (Core Mental Model)

- AEM is a data platform that builds **one trusted record** of a real-world entity (device, user, application) from **multiple imperfect sources**
- Every configuration decision ultimately serves one goal: making that unified record accurate, complete, and trustworthy

---

## Concept → AEM Configuration Decision Mapping

### Entity Resolution & Deduplication

| What I Learned | AEM Configuration Decision It Drives |
|---|---|
| Entities must have clear boundaries (device ≠ user ≠ application) | How to define and separate entity types in AEM's data model |
| Strong identifiers (serial number, cloud ID) are needed for reliable matching | Which fields to designate as primary match keys in dedup rules |
| Weak identifiers (hostname, IP) cause false matches if used alone | Which fields to use only as secondary/supporting signals in matching |
| Blocking reduces comparison cost — records must be grouped before matching | How to configure candidate grouping rules to keep matching efficient at scale |
| Thresholds determine auto-merge vs. human review | What confidence score cutoffs to set for automatic vs. manual resolution |
| Survivorship rules decide which value wins when sources conflict | Which source to trust for each field (e.g. HR wins for owner, endpoint tool wins for OS version) |

### Data Quality

| What I Learned | AEM Configuration Decision It Drives |
|---|---|
| Accuracy — value reflects reality | Defines source trust ranking per field. Wrong source = wrong golden record even if platform logic is correct |
| Completeness — required fields must be present | Defines which fields are mandatory at ingestion, which trigger enrichment workflows when missing |
| Consistency — same thing must look the same across sources | Defines which fields need controlled vocabularies and normalisation rules before matching |
| Timeliness — stale data is operationally harmful | Defines acceptable data freshness thresholds and whether stale records should be flagged or excluded |
| Validity — values must conform to expected format/range | Defines schema validation rules at ingestion — what gets rejected vs. quarantined |

### ETL vs ELT

| What I Learned | AEM Configuration Decision It Drives |
|---|---|
| ETL transforms before data lands — less flexible, more controlled | Relevant if PII must be masked or scrubbed before entering AEM's storage layer |
| ELT transforms after landing — raw data preserved, more flexible | AEM likely uses ELT: raw agency data lands first, entity resolution + quality transforms run inside the platform |
| ELT allows re-running rules on already-stored data | Means dedup rules and field mappings can be updated without re-extracting from source agencies |

### Pipeline Patterns (Batch vs Streaming)

| What I Learned | AEM Configuration Decision It Drives |
|---|---|
| Batch — data moves in scheduled chunks (hourly, daily) | Sufficient for use cases like daily compliance reports or weekly inventory reviews |
| Streaming — data moves continuously in near real-time | Required only if AEM needs to reflect source changes within minutes (e.g. real-time security alerting) |
| Streaming adds significant complexity — ordering, retries, duplicates | Default to batch; justify streaming only if the use case genuinely requires low latency |

---

## Outstanding Questions for the Team

- Which entity types are in scope for Phase 1? (devices only, or also users and applications?)
- Which fields are designated as strong identifiers for each entity type?
- For each high-value field, which source is considered authoritative?
- What is the rule when two sources disagree on the same field value?
- Which fields are mandatory for a record to be operationally usable?
- What confidence threshold is acceptable for auto-merge vs. human review queue?
- Who reviews records that land in the human review queue?
- What is the acceptable data freshness requirement — daily, hourly, or near real-time?
- Is raw source data preserved after transformation, or only the unified golden record?
- Are there PII or compliance requirements that affect whether ETL or ELT is used?

---

## Summary

- All three acceptance criteria are covered:
  - ✅ Entity resolution and deduplication rules understood
  - ✅ Data quality dimensions (accuracy, completeness, consistency) understood and mapped to field configuration
  - ✅ ETL vs ELT difference understood with clear guidance on when each is appropriate
- Foundational knowledge is sufficient to **participate in AEM configuration discussions**
- Outstanding questions above are the next step — bring these to the team to move from conceptual understanding to actual configuration decisions
