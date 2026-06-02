# AEM Data Concepts — Learning Notes

Personal study notes for building foundational knowledge in data platform concepts, specifically oriented around AEM configuration decisions.

---

## Contents

| File | Description |
|---|---|
| `aem-foundation-data-concepts.md` | Full reference notes — entity resolution, data quality, ETL/ELT, pipeline patterns, storage concepts. Beginner-friendly with analogies and worked examples. |
| `aem-foundation-data-concepts-visuals.md` | ASCII diagram companion — 9 visuals covering pipeline flow, entity resolution steps, worked example, ETL vs ELT, batch vs streaming, and how all concepts connect. |
| `aem-foundational-concepts.md` | Condensed summary of core concepts — entity resolution, data quality dimensions, ETL vs ELT. |
| `aem_config_decisions_summary.md` | Concept-to-practice mapping — links each concept learned to a concrete AEM configuration decision. Includes outstanding questions for the team. |

---

## Topics Covered

- **Entity resolution and deduplication** — what it is, why it's hard, blocking/matching/thresholding/survivorship, golden records
- **Data quality dimensions** — accuracy, completeness, consistency, timeliness, validity and how each affects platform reliability
- **ETL vs ELT** — where transformation happens, trade-offs, when to use each
- **Batch vs streaming** — data movement patterns, complexity trade-offs, decision guide
- **Data pipeline concepts** — pipeline stages, data lake vs warehouse vs lakehouse, Kafka at conceptual level
- **Field and entity design** — strong vs weak identifiers, normalisation, schema design

---

## How to Use

- Start with `aem-foundation-data-concepts.md` for full explanations
- Open `aem-foundation-data-concepts-visuals.md` alongside it to see each concept visually
- Use `aem_config_decisions_summary.md` when preparing for team configuration discussions
