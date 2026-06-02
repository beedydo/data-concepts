# Foundational Notes for AEM-Oriented Data Concepts

> **Visual companion:** See `aem-foundation-data-concepts-visuals.md` for diagrams of all concepts below.

---

## How to Read This Document

- This document builds up from zero — no prior data engineering background assumed
- Every section starts with "what it is", then "why it matters", then "how it works in practice"
- Analogies are used throughout to make abstract ideas concrete
- The goal is not to memorise tools — it is to understand the concepts well enough to ask the right questions in design discussions

---

## The Core Mental Model

- AEM is a data platform — its job is to build a **single trustworthy record** of a real-world thing (a device, a user, an application) from **multiple imperfect sources**
- Think of it like piecing together a complete profile of a person from five different partial address books — each one has some info but none is fully correct or complete
- To do this well, the platform needs to answer four questions:
  - What is the thing being tracked? → **Entity definition**
  - What properties describe it? → **Field/attribute design**
  - How do you know two records are the same thing? → **Entity resolution and deduplication**
  - How do you decide which value to trust when sources disagree? → **Survivorship logic**

---

## 1. Core Data Modeling Concepts

### What is an Entity?

- An entity is the **real-world thing** you are tracking — a device, a user, an application, a server, a cloud account
- In a database or platform, each entity is represented as **one row** (or one record/object)
- Example: a laptop called "WKSTN-007" owned by Alice is one entity
- Bad entity design = confusion later
  - Example: if laptops, virtual machines, and user accounts are all lumped into one generic "asset" type, deduplication and ownership rules become impossible to write clearly
  - Rule of thumb: each entity type should represent one kind of real-world object

### What is a Field (Attribute)?

- A field (also called an attribute) is a **property that describes an entity**
- Examples for a device entity: hostname, IP address, serial number, OS version, owner, last-seen date, department
- Fields are the raw material for:
  - **Matching** — deciding if two records are the same thing
  - **Enrichment** — filling in missing info from another source
  - **Filtering** — finding all devices in a given department
  - **Reporting** — aggregating counts, statuses, ownership
- Not all fields are equal — some are strong identifiers, some are weak:
  - **Strong identifiers**: serial number, cloud instance ID, device certificate, NRIC — stable, unique, rarely reused
  - **Weak identifiers**: hostname, IP address, display name — can be shared, recycled, or formatted differently per source
  - Rule: prefer strong identifiers for matching rules; use weak identifiers only as supporting signals

### What is a Schema?

- A schema is the **blueprint** of a dataset — it defines what fields exist, what type each field is, and which fields are required
- Example schema for a device:

  ```
  device_id    : string   (required)
  hostname     : string   (optional)
  serial_number: string   (optional)
  os_version   : string   (optional)
  owner_email  : string   (optional)
  last_seen_at : datetime (required)
  ```

- Why schemas matter:
  - Without a shared schema, two sources describing the same device may have completely different field names and formats
  - The platform needs to map ("transform") incoming data to a common schema before comparing records

### What are Relationships?

- A relationship links one entity to another — "this device is owned by this user", "this application runs on this server", "this server belongs to this business unit"
- Relationships make the data model useful for real operational questions:
  - "Which devices belong to the Finance department?" → device-to-department relationship
  - "Which users have admin rights on production servers?" → user-to-server relationship with permission attribute
- In a database, relationships are often represented as foreign keys — one record stores the ID of another record it is linked to

### What is Normalization / Standardization?

- **Normalization** (in the platform sense, not the strict database sense) means converting incoming values into a **consistent, standard format** before using them
- Why this is needed:
  - Source A says `Windows 11`
  - Source B says `win11`
  - Source C says `Microsoft Windows 11 (64-bit)`
  - All three mean the same thing — but a computer sees three different strings and would not match them without normalization
- Common normalization steps:
  - **Casing**: convert all hostnames to lowercase (`WKSTN-007` → `wkstn-007`)
  - **Date formats**: convert `DD/MM/YYYY` and `YYYY-MM-DD` to one standard
  - **OS names**: map variants to a controlled vocabulary (`win11`, `Windows 11`, `Win 11` → `Windows 11`)
  - **Null handling**: decide whether missing = `null`, `""`, `"unknown"`, `"N/A"` — pick one and enforce it
- Normalization happens before matching — poor normalization causes false mismatches even when the underlying data refers to the same thing

---

## 2. Entity Resolution and Deduplication

### What is Entity Resolution?

- Entity resolution = the process of determining whether **two or more records from different sources refer to the same real-world entity**
- It is not the same as removing duplicates within a single spreadsheet
- It is a **cross-system matching problem** — sources have different schemas, different data quality, and partially overlapping identifiers
- Analogy: imagine you have a contact named "Jon Tan" in your phone, "Jonathan Tan" in your email, and "J. Tan (Finance)" in a shared directory — entity resolution is the logic that decides all three are the same person

### Why Entity Resolution is Hard

- Sources are **incomplete**: one record has a serial number but no owner; another has an owner but no serial number
- Sources are **inconsistent**: hostname formatted differently per source, OS names not standardized, dates in different timezones
- Sources are **stale**: one system updated yesterday, another last updated six months ago
- The central trade-off:
  - **False positive**: two different entities incorrectly merged into one → dangerous (wrong ownership, wrong device counts)
  - **False negative**: same entity left as two separate records → wastes the value of having a unified platform
  - Tighter rules → fewer false positives, more false negatives
  - Looser rules → fewer false negatives, more false positives
  - Good design tunes the threshold and provides a human review path for borderline cases

### How Deduplication Rules Work — Step by Step

- **Step 1: Blocking**
  - Problem: comparing every record against every other record is O(n²) — too expensive at scale
  - Solution: first group records into candidate pairs that are likely to match, then only compare within groups
  - Example: only compare records that share the same email domain, or the same first three characters of the serial number
  - Goal: dramatically reduce the number of comparisons without missing real matches

- **Step 2: Matching**
  - Compare candidate pairs and score their similarity
  - Two approaches:
    - **Deterministic (rules-based)**: hard rules — if serial numbers match exactly, it's a match
    - **Probabilistic (confidence-based)**: weighted score — name similarity 85% + same department + same IP range → combined confidence score of 0.87
  - Example rule set:
    ```
    IF serial_number.exact_match                              → MATCH (confidence: 1.0)
    IF (hostname.fuzzy_match > 0.9 AND owner.exact_match)    → MATCH (confidence: 0.85)
    IF (hostname.fuzzy_match > 0.7 AND same_department)      → REVIEW (confidence: 0.6)
    ELSE                                                      → NO MATCH
    ```

- **Step 3: Thresholding**
  - Decide what to do based on confidence score:
    - Score ≥ 0.9 → auto-merge (high confidence, no human needed)
    - Score between 0.6–0.9 → send to human review queue (borderline case)
    - Score < 0.6 → treat as distinct entities

- **Step 4: Survivorship**
  - After two records are matched and merged, the platform needs one final value for each field
  - This is called **survivorship logic** — which value "survives" into the golden record?
  - Common strategies:
    - **Most recent**: use the value from whichever source was updated last
    - **Source ranking**: source A is trusted more than source B for OS version, so always prefer source A
    - **Non-null preference**: if one record has a value and the other doesn't, use the one with a value
    - **Most complete**: prefer the record with the fewest blank fields

### What is a Golden Record?

- A **golden record** (also called master record or unified record) is the single authoritative version of an entity after deduplication and survivorship
- It is the output of entity resolution — one record that represents the best known truth about a real-world object, assembled from all available sources
- Think of it as the "official profile" that downstream systems and users see and trust

### Deduplication vs Unification

- **Deduplication**: identifying multiple records that are the same thing and collapsing them into one
- **Data unification**: broader — includes deduplication plus:
  - Standardizing field formats
  - Filling in missing values from secondary sources (enrichment)
  - Applying survivorship rules to resolve conflicts
  - Creating the golden record
- Teams often think they only need dedup, but in practice they also need enrichment, source prioritization, and conflict resolution — plan for all of these

---

## 3. Data Quality Fundamentals

### Why Data Quality Matters

- A unified platform with poor data quality can be **worse** than separate systems
  - Because errors are now centralized and appear authoritative
  - Downstream users trust the platform — if it says Alice owns a server that she doesn't, decisions get made on false information
- Good data quality = downstream decisions can be trusted
- Bad data quality = garbage in, garbage out, no matter how sophisticated the pipeline

### Accuracy

- **Definition**: does the field value correctly reflect reality?
- A field can be filled in but still wrong:
  - Owner field says Alice, but laptop was reassigned to Bob six months ago
  - Postal code is formatted correctly but points to the wrong district
  - Serial number was entered with a typo that passes format validation
- Why it matters for configuration:
  - Accuracy tells you **which source to trust for which field**
  - Example: endpoint management tool is most accurate for OS version; HR system is most accurate for department and user status
  - Build source trust rules around accuracy — don't blindly take any field from any source
- Hard to detect automatically — often requires cross-referencing against an authoritative external source or spot-checking

### Completeness

- **Definition**: are required fields populated — no missing or null values where data should exist?
- Measured as: `(count of non-null values / total record count) × 100%`
- Why it matters:
  - A device record with no owner, no serial number, and no environment tag cannot be used for matching or operations
  - 60% completeness on phone number = platform cannot run any SMS-based workflow
- Tiers of completeness to define upfront:
  - **Mandatory**: must be non-null for record to be accepted (e.g., device ID)
  - **Required for matching**: must be non-null for entity resolution to work (e.g., serial number or hostname)
  - **Optional but tracked**: missing triggers enrichment workflow or review queue (e.g., department)
- Important distinction: **incomplete ≠ inaccurate**
  - A present value can be wrong (inaccurate)
  - An absent value is simply missing (incomplete)
  - Both need fixing, but by different means

### Consistency

- **Definition**: is the same entity represented the same way across systems and over time?
- Two types of consistency:
  - **Cross-system consistency**: same entity described the same way in source A and source B
    - Example: `prod`, `Production`, `PRD`, `PROD` all mean the same environment — inconsistent across sources
  - **Temporal consistency**: values don't change in logically impossible ways over time
    - Example: device age decreasing, creation date jumping forward, OS version downgrading without an event
- Why it matters:
  - Inconsistency is the root cause of most deduplication failures
  - Matching rules compare field values — if `Windows 11` ≠ `win11` to the system, the same device is not matched
  - Fix consistency with normalization (controlled vocabularies, canonical formats) — do this before matching, not after
- Practical step: define a controlled vocabulary for every field used in matching rules

### Timeliness

- **Definition**: is the data current enough to be useful for the intended use case?
- Even accurate, complete, and consistent data becomes harmful if it is stale:
  - Asset inventory last updated six months ago means patch status is unreliable
  - User ownership record from before an org restructure means alerts go to the wrong person
- Questions to ask:
  - How frequently does each source push updates?
  - What is the maximum acceptable age of data for each use case?
  - Should stale records be flagged or excluded from operations automatically?

### Validity

- **Definition**: do values conform to the expected format, type, or allowed range?
- Examples of invalid data that passes basic ingestion:
  - IP address field contains `999.0.0.1` — valid-looking string but not a real IP
  - Date field contains `2099-01-01` — valid date format but clearly wrong
  - Email field contains `user@` — passes string check but is not a valid email
- Validity checks are enforced at the ingestion layer using schema validation and format rules
- Data that fails validity checks should be rejected, flagged, or quarantined — not silently accepted

### How All Five Dimensions Interact

```
Accurate but incomplete:   correct serial number, but owner field is blank
Complete but inaccurate:   all fields filled, but owner was not updated after reassignment
Consistent but inaccurate: every system says "win11" — consistently wrong OS label
Complete and consistent but stale: all fields present and matching — last updated 8 months ago
Valid but inaccurate:      IP address is valid format — but belongs to a different device

All five dimensions need to be addressed for entity resolution to work reliably.
```

---

## 4. ETL vs ELT

### The Core Idea — Where Does Transformation Happen?

```
ETL: Extract → Transform → Load
     [Source] → [Clean/shape in pipeline] → [Target system receives clean data]

ELT: Extract → Load → Transform
     [Source] → [Raw data lands in target] → [Transform inside target using its compute]
```

### ETL — Extract, Transform, Load

- **Extract**: pull data from source systems (databases, APIs, files, event streams)
- **Transform**: clean, reshape, validate, mask, or enrich data inside the pipeline before it reaches the destination
- **Load**: write the already-transformed, clean data into the target system
- When ETL is appropriate:
  - Target system has limited compute (legacy data warehouses, relational databases)
  - Data must be cleaned or masked before it is allowed into the target environment — e.g., PII scrubbing, compliance requirements
  - Source data is so messy that raw form is useless to store
  - Storage is expensive — only store what is needed after cleaning
- Trade-offs:
  - More control over what lands in the target
  - But if transformation logic changes, you often need to re-extract from source and redo everything
  - Pipeline is more complex — transformation bugs mean bad data silently enters the target

### ELT — Extract, Load, Transform

- **Extract**: pull data from source systems
- **Load**: write raw (or lightly processed) data directly into the target system first
- **Transform**: run transformation logic inside the target system after the data is already stored
- When ELT is appropriate:
  - Target is a cloud data warehouse with cheap, scalable compute (BigQuery, Snowflake, Redshift, Azure Synapse)
  - Team wants to preserve raw data for multiple downstream use cases, or for reprocessing when rules change
  - Different teams need different transformations of the same raw data
  - Schema is still evolving — preserve flexibility by landing raw first
  - Analysts want to explore raw data before standardizing transformation logic
- Trade-offs:
  - Raw data is preserved — transformation logic can change without re-extracting from source
  - But target system must be powerful enough to handle large-scale transformation queries
  - Raw layer can accumulate messy, untransformed data that becomes hard to manage over time

### ETL vs ELT — Side-by-Side Comparison

| Factor | ETL | ELT |
|--------|-----|-----|
| Transformation location | Before target (in pipeline) | Inside target (in DW/lake) |
| Target compute requirement | Low — target receives clean data | High — target does transformation work |
| PII / compliance | Clean before storage | Must control access to raw layer |
| Schema stability | Works well with fixed schema | Handles evolving schema better |
| Reprocessing when rules change | Must re-extract from source | Re-run transform on already-stored raw data |
| Storage cost | Lower — store only cleaned data | Higher — store raw AND transformed data |
| Flexibility | Less flexible post-ingest | More flexible — multiple transforms from same raw data |

### AEM Context

- AEM as a data platform likely uses **ELT pattern**:
  - Raw entity data from agency sources lands in the lake/warehouse layer first
  - Entity resolution, deduplication, and data quality transforms run inside the platform
  - Raw data is preserved for audit trail and reprocessing when dedup rules need to change
  - Multiple downstream consumers (reporting, operations, compliance) can apply different transforms to the same raw data

---

## 5. Data Pipeline Concepts

### What is a Data Pipeline?

- A data pipeline is the set of steps that moves data from where it originates (source) to where it is used (destination)
- Think of it like a factory assembly line — raw materials come in one end, go through several processing stations, and finished product comes out the other end
- Each stage transforms or filters the data in some way

### Simple Mental Model of a Pipeline

```
[Source Systems]
       ↓  (ingestion — pull or push data in)
[Ingestion Layer]
       ↓  (validate format, reject bad records)
[Standardisation & Validation]
       ↓  (normalize field values, handle nulls)
[Matching & Entity Resolution]
       ↓  (apply dedup rules, survivorship logic)
[Unified Storage / Golden Record Layer]
       ↓
[Consumption — dashboards, APIs, downstream tools]
```

- At each stage, a different design question appears:
  - **Ingestion**: how does data arrive? push (source sends it), pull (platform fetches it), or event stream?
  - **Validation**: what happens to malformed records? reject, flag, quarantine?
  - **Standardization**: what format transformations are needed before matching?
  - **Matching**: what rules decide when two records are the same entity?
  - **Storage**: what version of data is stored — raw only, transformed only, or both?

### Batch Processing

- **What it is**: data moves in **scheduled chunks** — hourly, daily, or nightly jobs collect and process data in bulk
- Analogy: collecting all the day's mail and sorting it once at the end of the day — not instant, but predictable and manageable
- When batch is appropriate:
  - Use case does not require real-time updates — e.g., daily inventory review, weekly compliance reports
  - A few hours of delay does not meaningfully affect decisions
  - Team needs predictable, easy-to-reason-about reconciliation and retry workflows
- Advantages:
  - Simpler to build and operate
  - Easier to handle failures — just re-run the batch
  - Reconciliation is straightforward — you know exactly what was processed in each run
- Disadvantages:
  - Data is never more current than the last batch run
  - Not suitable for real-time alerting or event-driven response

### Streaming Processing

- **What it is**: data is processed **continuously or near real-time** as events arrive — no waiting for a scheduled batch window
- Analogy: reading and sorting each letter the moment it arrives at the door, rather than waiting for end-of-day
- When streaming is appropriate:
  - Use case genuinely requires low-latency data — e.g., security alerts, real-time compliance events, automated response triggers
  - Delay of even a few minutes changes the value of the data
- Advantages:
  - Data is fresh — system reflects reality much faster
  - Supports event-driven architectures and automated triggers
- Disadvantages:
  - Significantly more complex to build and operate
  - Must handle: event ordering (events arriving out of sequence), deduplication (same event processed twice), schema drift (source changes format mid-stream), consumer reliability (what happens if downstream goes down)
  - **Rule of thumb**: only adopt streaming when there is a genuine operational requirement for low-latency data — do not adopt it by default because it sounds modern

### Kafka — Conceptual Level Only

- Kafka is a **distributed event-streaming platform** commonly used to move data between producers (sources) and consumers (destinations) in near real-time
- Analogy: a conveyor belt in a factory — multiple input stations drop things onto the belt, multiple output stations pick things off
- Key ideas to understand (not internals):
  - Kafka stores a log of events — data is not consumed and destroyed, it is retained for a configurable period
  - Multiple consumers can read the same events independently at their own pace
  - This decouples producers and consumers — the source system does not need to know about the destination system
- Why it appears in data platform discussions:
  - When sources push events continuously (e.g., endpoint status changes), Kafka acts as the ingestion buffer
  - Enables streaming ingestion without directly coupling sources to the processing layer
  - The key architectural implication: streaming with Kafka means **continuous ingestion + event handling**, not scheduled batch runs

---

## 6. Data Storage Concepts (Bonus Context)

### What is a Database?

- A database is a structured collection of data that supports querying, updating, and retrieval
- Most databases are **relational** — data is stored in tables with rows and columns, and tables relate to each other via keys
- Not typically the primary storage layer in a large data platform — too rigid for the scale and variety of data needed

### What is a Data Warehouse?

- A data warehouse is a system designed for **analytical queries** across large volumes of historical data
- Data is stored in a structured, pre-processed format optimised for reading (not frequent writes)
- Examples: BigQuery, Snowflake, Redshift, Azure Synapse
- Data warehouses are the natural home for ELT — they have the compute to run large-scale transformation queries cheaply

### What is a Data Lake?

- A data lake stores **raw, unprocessed data** at scale — structured, semi-structured, and unstructured
- Think of it as a giant file store that can hold anything: JSON files, CSVs, logs, binary files
- Very flexible — raw data is preserved; transformation happens later when needed
- Used as the landing zone in ELT pipelines — data lands in the lake first, then gets transformed into the warehouse

### What is a Lakehouse?

- A lakehouse combines a data lake (raw storage, flexible formats) with a data warehouse (structured querying, analytics)
- Increasingly common in modern data platforms — store raw data in the lake layer, query and transform it using warehouse-style SQL engines

---

## 7. How All Concepts Connect

| Concept | Practical question it answers |
|---------|-------------------------------|
| Entity definition | What real-world object is being tracked? |
| Field selection | Which properties are needed for matching, reporting, ownership? |
| Strong identifiers | Which fields are stable enough to use in dedup rules? |
| Normalization | Which value mappings are needed before records can be compared reliably? |
| Entity resolution | When do two records count as the same object? |
| Golden record | What is the single authoritative version of a record? |
| Survivorship logic | Which source or rule decides the final value when sources conflict? |
| Accuracy | Which source should be trusted for a given field? |
| Completeness | Which missing values should block use or trigger enrichment? |
| Consistency | Which fields need controlled vocabularies or canonical formats? |
| Timeliness | How fresh must data be for the use case to work? |
| Validity | What format rules must data pass before being accepted? |
| ETL vs ELT | Should transformation happen before or after data lands? |
| Batch vs streaming | How quickly must the platform reflect source changes? |
| Data lake | Where is raw data preserved before transformation? |
| Data warehouse | Where is transformed data stored for analytical querying? |

---

## 8. Worked Examples

### Example 1: Same Device, Three Different Records

- Source A (endpoint tool): `hostname=WKSTN-007, serial=ABC999, os=Windows 11, owner=alice@agency.gov.sg`
- Source B (network scanner): `hostname=wkstn-007, ip=10.0.0.45, os=win11`
- Source C (HR system): `serial=ABC999, owner=bob@agency.gov.sg, department=Finance`
- What needs to happen:
  - **Normalization**: `wkstn-007` and `WKSTN-007` → standardize casing to match
  - **Normalization**: `Windows 11` and `win11` → map to controlled vocabulary `Windows 11`
  - **Entity resolution**: Source A and Source B match on hostname (after normalization) → same device
  - **Entity resolution**: Source A and Source C match on exact serial number → same device
  - **Survivorship**: `owner` conflicts — Source A says Alice, Source C says Bob. Apply source trust rule: if HR system is more authoritative for ownership, Bob wins
  - **Output**: one golden record — `serial=ABC999, hostname=wkstn-007, os=Windows 11, owner=bob@agency.gov.sg, department=Finance, ip=10.0.0.45`

### Example 2: Missing Owner = Incomplete Record

- Record exists in the platform: device ID, serial number, OS version — all present
- Owner field: null
- What happens:
  - Record is technically ingested and passes validation
  - But it cannot be used for operations — no one to notify if device goes out of compliance
  - **Completeness problem**: owner is a required operational field
  - Right response: trigger enrichment workflow (attempt to pull owner from HR system), or flag for manual review, or lower the record's confidence score in downstream automation

### Example 3: Choosing Between Batch and Streaming

- Scenario A: daily compliance report showing which devices are running approved OS versions
  - Data needs to be accurate as of yesterday — one-day-old data is fine
  - **Batch** is appropriate — simpler, cheaper, sufficient

- Scenario B: real-time alerting when a device runs an unauthorised process
  - Delay of even a few minutes may mean a threat is not caught in time
  - **Streaming** is justified — operational requirement for low latency exists

- Scenario C: weekly executive dashboard of asset counts by department
  - Weekly cadence makes batch not just sufficient but ideal
  - **Batch** clearly wins

---

## 9. Recommended Learning Sequence (1 Day)

- **Block 1 — Entity resolution and deduplication (highest priority)**
  - Start here because this directly explains why the platform needs rules to decide whether two records are the same thing
  - Understanding this makes merge logic, golden records, and survivorship immediately meaningful
  - Core concepts: entity, field, strong identifier, blocking, matching, thresholding, survivorship, golden record

- **Block 2 — Data quality fundamentals**
  - Move here next because even perfect matching rules fail when identifiers are missing, stale, or inconsistently formatted
  - Core concepts: accuracy, completeness, consistency, timeliness, validity
  - Understand how each dimension causes entity resolution to fail when it is poor

- **Block 3 — ETL/ELT and pipeline patterns (conceptual only)**
  - Finish here at a conceptual level — understand trade-offs, not tool internals
  - Core concepts: ETL vs ELT, batch vs streaming, data lake vs warehouse, pipeline stages
  - Goal: be able to reason about why certain design decisions were made, not implement them

---

## 10. Questions to Take Back to the Team

- Which entity types matter most for the first phase of implementation?
- Which fields are considered strong identifiers vs. weak supporting attributes?
- For each high-value field, which source should be trusted as authoritative?
- What should happen when two sources disagree on the same field value?
- Which fields are mandatory for a record to be operationally useful?
- What is the acceptable staleness threshold for each use case — daily, hourly, or near real-time?
- Is raw source data preserved after transformation, or only the unified golden record?
- What happens to records that fall in the review queue — who reviews them and how?
- What confidence threshold is acceptable for auto-merge vs. human review?

---

*These questions convert foundational theory into implementation conversations without requiring deep vendor-specific study on day one.*
