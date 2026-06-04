# Session Notes — 2026-06-04 — Data Concepts Extended

---

## Topics Covered

- Data modelling — what it is, three levels (conceptual, logical, physical), how it fits into AEM
- Data source evaluation — how to decide whether to onboard a source onto AEM
- Concurrent vs sequential workflow — data modelling and source profiling can run in parallel; gap analysis needs a draft model first
- Data contracts — what they are, what they contain, where they sit in the workflow (after onboarding decision, before pipeline build)
- How data modelling, source eval, and data contracts connect to entity resolution, data quality, ETL/ELT
- Full priority task sequence for AEM implementation
- Data glossary with annotated visuals

---

## Key Concepts

### Data Modelling
- Blueprint of what entities exist, what fields they have, how they relate
- Three levels: conceptual (what exists) → logical (what fields, what types) → physical (actual DB implementation)
- Must exist (even as rough draft) before gap analysis and field mapping can begin
- AEM configuration IS largely data model definition — entity types, fields, relationships

### Data Source Evaluation
- Per-source process: profile → gap analysis → field mapping → value statement → effort estimate
- Data profiling (what does the source actually contain?) requires zero model knowledge — start immediately
- Gap analysis requires draft model to compare against — do not start until model has first pass
- Output: onboard / defer / reject decision with rationale per source

### Concurrent Workflow
- Data modelling and data profiling: run in parallel from day one
- Gap analysis and field mapping: need draft model first — start week 2
- Two workstreams converge and refine each other iteratively

### Data Contracts
- Formal agreement between source team (producer) and AEM (consumer)
- Defines: schema, semantics, quality SLAs, delivery frequency, versioning, ownership
- Sits between onboarding decision and pipeline build
- Prevents silent schema drift from breaking entity resolution

### Priority Task Sequence for AEM
1. Draft data model (entity types + key fields)
2. Profile data sources (concurrent with #1)
3. Negotiate data contracts per onboarded source
4. Define data quality + normalisation rules
5. Define entity resolution + dedup rules
6. Design pipeline (ETL/ELT, batch vs streaming)

---

## Files Created This Session

| File | Description |
|---|---|
| `aem_platform_big_picture.md` | Priority task sequence with rationale; ⚠️ flags for Zscaler-specific unknowns |
| `data_glossary.md` | A–Z glossary + annotated ASCII visuals (relational DB, ERD, SQL, Kafka, entity/field/attribute, AEM single-source CrowdStrike example) |

---

## Files Modified This Session

| File | Change |
|---|---|
| `aem_config_decisions_summary.md` | Rescoped to exactly match Jira ticket AC (3 criteria only) |
| `data_glossary.md` | Added Part 0 (entity/field/attribute annotated visual) and Part 0b (AEM + CrowdStrike single-source end-to-end example) — **not yet committed/pushed** |

---

## Open Items / Next Steps

- [ ] Commit and push `data_glossary.md` (has unsaved edits from interrupted session)
- [ ] Verify Zscaler AEM-specific items flagged ⚠️ in `aem_platform_big_picture.md` with official Zscaler docs or team
- [ ] Begin actual data modelling work — define entity types and key fields for AEM in scope
- [ ] Run data profiling on first candidate source
- [ ] Determine which Jira ticket covers data modelling + source evaluation work

---

## Questions for the Team

- Which entity types are in scope for Phase 1?
- Which data sources are confirmed for onboarding vs. still being evaluated?
- Does Zscaler AEM have a built-in data model or does it need full configuration?
- What ETL/ELT pattern does Zscaler AEM use natively?
