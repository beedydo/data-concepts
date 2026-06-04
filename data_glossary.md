# Data Terminology Glossary — Visual Reference

---

## Part 0 — Entity, Fields, Attributes, Records — Annotated

> These four terms are used constantly. This visual shows exactly what each one means.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║              ENTITY / FIELD / ATTRIBUTE / RECORD — ANNOTATED                    ║
╚══════════════════════════════════════════════════════════════════════════════════╝

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                          devices                                        │
  │  ▲                                                                      │
  │  │                                                                      │
  │  └── ENTITY TYPE                                                        │
  │      The category of real-world thing being tracked.                    │
  │      "Device" is the entity type. One specific laptop is one instance. │
  │      Other entity types: User, Application, Cloud Workload, Dept.      │
  ├───────────────┬────────────────┬──────────────┬────────────────────────┤
  │  device_id    │ serial_number  │  hostname    │  os_version            │
  │  ▲            │  ▲             │  ▲           │  ▲                     │
  │  │            │  │             │  │           │  │                     │
  │  └─ FIELD ────┘  └─ FIELD ────┘  └─ FIELD ───┘  └─ FIELD             │
  │                                                                         │
  │  A FIELD = one property that describes the entity.                     │
  │  Also called: COLUMN, ATTRIBUTE (all three mean the same thing).       │
  │  Each field stores one type of data per record.                        │
  ├───────────────┼────────────────┼──────────────┼────────────────────────┤
  │  d-001        │  ABC999        │  wkstn-01    │  Windows 11            │◄─┐
  ├───────────────┼────────────────┼──────────────┼────────────────────────┤  │
  │  d-002        │  XYZ123        │  wkstn-02    │  macOS 14              │◄─┤
  ├───────────────┼────────────────┼──────────────┼────────────────────────┤  │
  │  d-003        │  DEF456        │  wkstn-03    │  Windows 11            │◄─┤
  └───────────────┴────────────────┴──────────────┴────────────────────────┘  │
                                                                               │
    Each horizontal row = one RECORD (also called: ROW, TUPLE, INSTANCE) ─────┘
    One record = one specific real-world thing (one specific laptop)


  ─────────────────────────────────────────────────────────────────────────────
  SUMMARY OF SYNONYMS:

  Entity type  ≈  Table name      (the category: "devices", "users")
  Entity       ≈  Record / Row    (one specific thing: laptop d-001)
  Field        ≈  Column          (one property: "hostname")
  Attribute    ≈  Field / Column  (same thing, different word)
  Value        =  the actual data in one cell (e.g. "wkstn-01")

  ┌──────────────┬─────────────────────────────────────────────────────────┐
  │ Term         │ Analogy                                                  │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │ Entity type  │ A form template (e.g. the blank "Device Registration"   │
  │              │ form — defines what info to collect)                    │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │ Entity       │ One filled-in form (this specific laptop's details)     │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │ Field /      │ One question on the form ("What is the hostname?")      │
  │ Attribute    │                                                          │
  ├──────────────┼─────────────────────────────────────────────────────────┤
  │ Value        │ The answer written in that field ("wkstn-01")           │
  └──────────────┴─────────────────────────────────────────────────────────┘
```

---

## Part 0b — How AEM Works with a Single Data Source (End-to-End Example)

> Walkthrough: one source (CrowdStrike EDR via DEEP/SEED) flowing into AEM, from raw data to golden record.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║          AEM + SINGLE DATA SOURCE — END TO END (CrowdStrike EDR)                ║
╚══════════════════════════════════════════════════════════════════════════════════╝

  ─────────────────────────────────────────────────────────────────────────────
  STEP 1: SOURCE EMITS DATA
  ─────────────────────────────────────────────────────────────────────────────

  CrowdStrike EDR (via SEED/DEEP) detects and reports on managed endpoints.
  It sends a record like this (raw JSON from the source):

  {
    "aid":          "cs-9f3a21b",          ← CrowdStrike's own device ID
    "hostname":     "WKSTN-007",           ← not normalised (uppercase)
    "platform":     "Windows",             ← not specific enough
    "os_version":   "10.0.19045",          ← raw build number, not human-readable
    "last_seen":    "2026-06-04T09:12:00Z",
    "agent_status": "online",
    "policy":       "Corp-Standard"
  }

  ┌─────────────┐
  │ CrowdStrike │ ──── pushes record ────▶  AEM ingestion endpoint
  │    EDR      │
  └─────────────┘


  ─────────────────────────────────────────────────────────────────────────────
  STEP 2: INGESTION — AEM RECEIVES THE RAW RECORD
  ─────────────────────────────────────────────────────────────────────────────

  • AEM receives the raw JSON and stores it in the raw layer (data lake)
  • Nothing is changed yet — raw data is preserved exactly as sent
  • A source tag is added: { "source": "crowdstrike_edr", "ingested_at": "..." }

  Raw layer now contains:
  /raw/crowdstrike_edr/2026-06-04/record_cs-9f3a21b.json


  ─────────────────────────────────────────────────────────────────────────────
  STEP 3: VALIDATION — IS THE RECORD USABLE?
  ─────────────────────────────────────────────────────────────────────────────

  AEM checks validity rules:

  ✅  "aid" present and non-null          → passes (strong identifier exists)
  ✅  "hostname" present                  → passes
  ✅  "last_seen" is valid datetime       → passes
  ✅  "agent_status" is known value       → passes ("online" in allowed list)

  Record passes validation → moves to normalisation.
  If it had failed (e.g. missing "aid"), it would be quarantined for review.


  ─────────────────────────────────────────────────────────────────────────────
  STEP 4: NORMALISATION — STANDARDISE FIELD VALUES
  ─────────────────────────────────────────────────────────────────────────────

  Raw value              →  Normalised value         Reason
  ─────────────────────────────────────────────────────────────────────────
  "WKSTN-007"            →  "wkstn-007"              casing standard: lowercase
  "Windows"              →  (kept, enriched later)   too vague on its own
  "10.0.19045"           →  "Windows 10 (22H2)"      build number mapped to
                                                       human-readable OS label
  "2026-06-04T09:12:00Z" →  kept as-is               already ISO 8601 standard

  Normalised record now looks like:
  {
    "source_id":    "cs-9f3a21b",
    "hostname":     "wkstn-007",
    "os_version":   "Windows 10 (22H2)",
    "last_seen":    "2026-06-04T09:12:00Z",
    "agent_status": "online",
    "source":       "crowdstrike_edr"
  }


  ─────────────────────────────────────────────────────────────────────────────
  STEP 5: ENTITY RESOLUTION — DOES THIS DEVICE ALREADY EXIST IN AEM?
  ─────────────────────────────────────────────────────────────────────────────

  AEM checks its existing golden records to see if this device is already known.

  BLOCKING: look for existing records with similar hostname or same source_id
  MATCHING: compare fields

  Scenario A — First time this device is seen:
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  No match found → CREATE new golden record                              │
  │  golden_record_id: gr-00142                                             │
  │  hostname:         wkstn-007                                            │
  │  os_version:       Windows 10 (22H2)                                   │
  │  last_seen:        2026-06-04T09:12:00Z                                 │
  │  sources:          [ crowdstrike_edr ]                                  │
  │  owner:            null  (not in this source — to be enriched later)   │
  └─────────────────────────────────────────────────────────────────────────┘

  Scenario B — Device already exists from a previous sync:
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  Match found (same hostname + same source_id) → MERGE / UPDATE         │
  │  Survivorship: "last_seen" → most recent wins → update to 09:12        │
  │  Survivorship: "os_version" → CrowdStrike trusted for OS → update      │
  │  Other fields not in this source → keep existing values                │
  └─────────────────────────────────────────────────────────────────────────┘


  ─────────────────────────────────────────────────────────────────────────────
  STEP 6: GOLDEN RECORD — UNIFIED OUTPUT
  ─────────────────────────────────────────────────────────────────────────────

  ╔═════════════════════════════════════════════════════════════════════════╗
  ║  GOLDEN RECORD: gr-00142                                                ║
  ║  ───────────────────────────────────────────────────────────────────── ║
  ║  hostname:       wkstn-007                                              ║
  ║  os_version:     Windows 10 (22H2)       ← from CrowdStrike            ║
  ║  agent_status:   online                  ← from CrowdStrike            ║
  ║  last_seen:      2026-06-04T09:12:00Z    ← from CrowdStrike            ║
  ║  owner:          null                    ← not yet enriched            ║
  ║  serial_number:  null                    ← not in EDR data             ║
  ║  department:     null                    ← not in EDR data             ║
  ║                                                                         ║
  ║  sources: [ crowdstrike_edr ]                                           ║
  ║  completeness: PARTIAL — awaiting MDM + HR enrichment                  ║
  ╚═════════════════════════════════════════════════════════════════════════╝

  This is what AEM exposes downstream for reporting, alerting, compliance.
  When MDM and HR data are later ingested, they will enrich the null fields
  and the golden record becomes more complete.


  ─────────────────────────────────────────────────────────────────────────────
  WHAT EACH CONCEPT LOOKS LIKE IN THIS EXAMPLE:
  ─────────────────────────────────────────────────────────────────────────────

  Entity type    →  Device (the category of thing AEM is tracking)
  Entity         →  wkstn-007 (one specific laptop)
  Fields         →  hostname, os_version, last_seen, owner, serial_number
  Record         →  the row in AEM's golden record table for wkstn-007
  Schema         →  the definition of all fields the Device entity can have
  Strong ID      →  source_id "cs-9f3a21b" (stable, unique, won't be reused)
  Weak ID        →  hostname "wkstn-007" (useful, but could be recycled)
  Normalisation  →  "WKSTN-007" → "wkstn-007", build number → OS label
  Validation     →  checking "aid" is present, "agent_status" is a known value
  Survivorship   →  CrowdStrike trusted for os_version, last_seen
  Completeness   →  owner, serial_number, department still null — partial record
  Data contract  →  agreement with CrowdStrike team that "aid" is always present
                    and "last_seen" is always ISO 8601 format
```

---

## Part 1 — Annotated Relational Database

> The most common type of database. Data stored in tables. Tables linked by keys.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                     ANNOTATED RELATIONAL DATABASE                               ║
╚══════════════════════════════════════════════════════════════════════════════════╝

  ◄─────────────────── TABLE (also called: Relation, Entity) ──────────────────►

  ┌──────────────────────────────────────────────────────────┐
  │                     devices                              │ ◄── TABLE NAME
  ├─────────────┬────────────────┬──────────┬───────────────┤
  │ device_id   │ serial_number  │ hostname │ owner_id      │ ◄── COLUMNS
  │ (PK)        │                │          │ (FK)          │     (also: Fields,
  ├─────────────┼────────────────┼──────────┼───────────────┤      Attributes)
  │ d-001       │ ABC999         │ wkstn-01 │ u-042         │ ◄── ROW
  │ d-002       │ XYZ123         │ wkstn-02 │ u-017         │     (also: Record,
  │ d-003       │ DEF456         │ wkstn-03 │ u-042         │      Tuple, Instance)
  └──────────────┴───────────────┴──────────┴───────────────┘
        │                                         │
        │                                         │
        ▼                                         ▼
   PRIMARY KEY (PK)                         FOREIGN KEY (FK)
   ─────────────────                        ─────────────────
   Unique identifier for each row           References the PK of another table
   No two rows can share this value         Creates a LINK between tables
   Cannot be null                           "owner_id here = user_id over there"


  ┌─────────────────────────────────────┐
  │               users                 │ ◄── SECOND TABLE
  ├──────────┬──────────────┬───────────┤
  │ user_id  │ name         │ dept      │
  │ (PK)     │              │           │
  ├──────────┼──────────────┼───────────┤
  │ u-017    │ Alice Tan    │ Finance   │
  │ u-042    │ Bob Lim      │ IT        │
  └──────────┴──────────────┴───────────┘
        ▲
        │
        └──── owner_id in devices TABLE points HERE
              This is a RELATIONSHIP (one user owns many devices)


  CARDINALITY — how many of one thing relate to another:
  ──────────────────────────────────────────────────────
  One-to-One   (1:1)   │  One device has one certificate
  One-to-Many  (1:N)   │  One user owns many devices        ◄── this example above
  Many-to-Many (M:N)   │  Many users access many apps


  SCHEMA — the full blueprint of all tables, columns, types, and relationships
  ──────────────────────────────────────────────────────────────────────────────
  ┌────────────────────────────────────────────────────────┐
  │  SCHEMA: security_platform                             │
  │  ├── TABLE: devices (device_id PK, serial, owner_id FK)│
  │  ├── TABLE: users   (user_id PK, name, dept)           │
  │  ├── TABLE: apps    (app_id PK, name, device_id FK)    │
  │  └── TABLE: vulns   (vuln_id PK, device_id FK, score)  │
  └────────────────────────────────────────────────────────┘
```

---

## Part 2 — Annotated ERD (Entity Relationship Diagram)

> ERD = visual map of entities, their fields, and how they connect. Used in data modelling.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                    ENTITY RELATIONSHIP DIAGRAM (ERD)                            ║
╚══════════════════════════════════════════════════════════════════════════════════╝

  ┌──────────────────┐                           ┌──────────────────┐
  │     DEVICE       │                           │      USER        │
  │ ─────────────── │                           │ ──────────────── │
  │ *device_id  (PK)│ ────────────────────────▶ │ *user_id    (PK) │
  │  serial_number  │   owns                    │  full_name       │
  │  hostname       │   (many devices           │  email           │
  │  os_version     │    to one user)           │  department      │
  │  owner_id  (FK) │                           │  status          │
  └────────┬────────┘                           └──────────────────┘
           │                                             ▲
           │ has                                         │ belongs to
           │ (one device has                             │
           │  many vulns)                               │
           ▼                                    ┌────────┴────────┐
  ┌──────────────────┐                          │  DEPARTMENT     │
  │  VULNERABILITY   │                          │ ─────────────── │
  │ ─────────────── │                          │ *dept_id   (PK) │
  │ *vuln_id    (PK) │                          │  dept_name      │
  │  cve_id          │                          │  agency         │
  │  severity        │                          └─────────────────┘
  │  status          │
  │  device_id  (FK) │
  └──────────────────┘

  LEGEND:
  * = Primary Key (PK)    FK = Foreign Key     ──▶ = Relationship direction
  (1:N) label on line = cardinality           Entity box = one table in the DB
```

---

## Part 3 — Annotated Data Platform Stack

> How data flows from raw sources to useful outputs across different storage layers.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                       DATA PLATFORM STORAGE LAYERS                              ║
╚══════════════════════════════════════════════════════════════════════════════════╝

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │  SOURCE SYSTEMS                                                               │
  │  MDM │ EDR │ Vuln Scanner │ CNAPP │ Cloud Tools │ CASB                       │
  └─────────────────────────────────┬────────────────────────────────────────────┘
                                    │  ingest (pull or push)
                                    ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │  DATA LAKE                                                                    │ ◄── Raw, unprocessed data
  │                                                                               │     Any format: JSON, CSV,
  │  /raw/mdm/2026-06-04/devices.json                                            │     logs, binary
  │  /raw/edr/2026-06-04/events.json                                             │
  │  /raw/vuln/2026-06-04/findings.json                                          │     Cheap storage
  │                                                                               │     Nothing is thrown away
  └─────────────────────────────────┬────────────────────────────────────────────┘
                                    │  transform (ELT — transform inside platform)
                                    ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │  DATA WAREHOUSE                                                               │ ◄── Cleaned, structured,
  │                                                                               │     analytics-ready data
  │  TABLE: unified_devices   (golden records, post entity resolution)            │
  │  TABLE: unified_users     (golden records)                                   │     Fast to query
  │  TABLE: vulnerability_findings  (normalised, deduplicated)                   │     Expensive compute
  │                                                                               │     Optimised for reads
  └─────────────────────────────────┬────────────────────────────────────────────┘
                                    │
                                    ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │  DATA MARTS                                                                   │ ◄── Subset of warehouse
  │                                                                               │     for a specific team
  │  compliance_mart │ security_operations_mart │ executive_reporting_mart        │     or use case
  └──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                             Dashboards / APIs / Tools


  LAKEHOUSE = Data Lake + Data Warehouse combined in one system
              (store raw data like a lake, query it like a warehouse)
              Examples: Databricks, Delta Lake, Apache Iceberg
```

---

## Part 4 — Annotated SQL Query

> SQL (Structured Query Language) = how you ask questions of a relational database.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                         ANATOMY OF A SQL QUERY                                  ║
╚══════════════════════════════════════════════════════════════════════════════════╝

  SELECT   d.hostname, d.os_version, u.name, u.department
  ──────   ──────────────────────────────────────────────
  │        │
  │        └── COLUMNS to return (from both tables)
  │
  └── Keyword: "give me these columns"

  FROM     devices d
  ────     ─────────
  │        │
  │        └── TABLE to query from, aliased as "d" (shorthand)
  │
  └── Keyword: "from this table"

  JOIN     users u  ON  d.owner_id = u.user_id
  ────     ──────────────────────────────────────
  │        │               │
  │        │               └── CONDITION: link rows where FK matches PK
  │        └── second TABLE to join, aliased as "u"
  └── Keyword: "combine with this other table"

  WHERE    u.department = 'Finance'
  ─────    ─────────────────────────
  │        │
  │        └── FILTER condition — only rows matching this
  └── Keyword: "only include rows where..."

  ORDER BY d.hostname ASC;
  ────────
  └── Sort results (ASC = A→Z, DESC = Z→A)


  RESULT: all Finance department devices with their owner names and OS versions

  ─────────────────────────────────────────────────────────────────────────────
  OTHER COMMON SQL KEYWORDS:
  GROUP BY  — aggregate rows by a column (e.g. count devices per department)
  HAVING    — filter after GROUP BY (like WHERE but for aggregates)
  LIMIT     — return only first N rows
  DISTINCT  — return only unique values
  COUNT()   — count rows
  SUM()     — add up values
  AVG()     — average of values
```

---

## Part 5 — Annotated Kafka / Streaming Pipeline

> Kafka is a message bus — connects systems that produce data to systems that consume it.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                    KAFKA STREAMING — ANNOTATED                                  ║
╚══════════════════════════════════════════════════════════════════════════════════╝

  PRODUCER                     KAFKA                        CONSUMER
  ─────────                    ─────                        ────────
  System that                  The message bus              System that reads
  generates events             in the middle                and processes events

  ┌──────────┐                 ┌─────────────────────────┐
  │  EDR     │ ──── event ──▶ │  TOPIC: edr-events       │ ──▶ ┌──────────────┐
  │  tool    │                 │                          │     │  AEM         │
  └──────────┘                 │  [event1][event2][event3]│     │  platform    │
                               │   ───────────────────── │     └──────────────┘
  ┌──────────┐ ──── event ──▶ │       LOG (append-only)  │
  │  MDM     │                 │                          │ ──▶ ┌──────────────┐
  │  tool    │                 └─────────────────────────┘     │  SIEM /      │
  └──────────┘                          │                       │  alerting    │
                                        │                       └──────────────┘
                               ▲        │
                               │        └── Multiple consumers read
                          TOPIC          the SAME events independently
                          ─────          at their own pace
                          Named channel
                          for a category
                          of events

  KEY CONCEPTS:
  ───────────────────────────────────────────────────────────────────────────────
  Producer   — any system that sends events to Kafka
  Consumer   — any system that reads events from Kafka
  Topic      — named channel grouping related events (e.g. "device-status-changes")
  Partition  — topic split into shards for parallelism and scale
  Offset     — position of a consumer in the log (how far they've read)
  Retention  — how long Kafka keeps events before deleting them
  Log        — append-only ordered sequence of events — nothing is overwritten
```

---

## Part 6 — Full Glossary A–Z

### A

- **Aggregate** — combining multiple rows into a single result (e.g. COUNT, SUM, AVG)
- **API (Application Programming Interface)** — a defined way for two systems to talk to each other; data platforms often expose APIs for consumers to query the data
- **Attribute** — another word for field/column — a property that describes an entity

### B

- **Batch processing** — moving or processing data in scheduled chunks (hourly, daily) rather than continuously
- **Blocking** — in entity resolution: narrowing down candidate pairs before matching to avoid comparing every record against every other
- **Boolean** — data type that holds only `true` or `false`

### C

- **Candidate pair** — two records identified as potentially referring to the same entity and selected for matching comparison
- **Cardinality** — how many of one entity relate to another (one-to-one, one-to-many, many-to-many)
- **CASB (Cloud Access Security Broker)** — tool that monitors and governs SaaS application usage
- **CDC (Change Data Capture)** — technique that captures only the rows that changed in a source since the last sync, instead of re-copying the whole table
- **Column** — a field in a table — defines one type of data stored per row (e.g. `hostname`)
- **Conceptual data model** — high-level map of entities and relationships; no field-level detail; technology-agnostic
- **Confidence score** — numerical measure (0–1) of how likely two records refer to the same entity
- **Consumer** — in streaming: a system that reads events from a message bus (e.g. Kafka)
- **Controlled vocabulary** — a fixed list of allowed values for a field (e.g. environment must be one of: dev, staging, prod)

### D

- **Data contract** — formal agreement between a data producer and consumer defining schema, quality SLAs, delivery frequency, and versioning rules
- **Data lake** — storage layer that holds raw, unprocessed data in any format at scale; nothing is cleaned before landing
- **Data lakehouse** — combines data lake (raw flexible storage) with data warehouse (structured querying); single system for both
- **Data mart** — a subset of a data warehouse scoped to a specific team or use case
- **Data model** — the blueprint defining what entities exist, what fields they have, and how they relate
- **Data pipeline** — the set of steps that moves and transforms data from source systems to a destination
- **Data profiling** — examining a dataset to understand its structure, completeness, quality, and content before using it
- **Data type** — the kind of value a field holds: string (text), integer (whole number), float (decimal), boolean (true/false), datetime (date + time)
- **Data warehouse** — storage layer holding cleaned, structured, analytics-ready data; optimised for fast querying
- **Deduplication** — identifying multiple records that refer to the same entity and collapsing them into one
- **Derived field** — a field whose value is calculated from other fields (e.g. `days_since_last_seen` derived from `last_seen_at`)

### E

- **ELT (Extract, Load, Transform)** — pipeline pattern where raw data lands in destination first; transformation runs inside destination
- **Entity** — a real-world thing being tracked in a data model (device, user, application, etc.)
- **Entity resolution** — process of determining whether records from different sources refer to the same real-world entity
- **ERD (Entity Relationship Diagram)** — visual map of entities, their fields, and the relationships between them
- **ETL (Extract, Transform, Load)** — pipeline pattern where data is cleaned/transformed before landing in destination
- **Event** — a record of something that happened at a point in time (e.g. "device d-001 connected at 14:32")
- **Event-driven architecture** — system design where components react to events as they happen rather than being called directly

### F

- **False negative** — in entity resolution: same entity incorrectly left as two separate records (missed match)
- **False positive** — in entity resolution: two different entities incorrectly merged into one
- **Field** — a property that describes an entity (synonym: column, attribute)
- **Foreign key (FK)** — a field in one table that references the primary key of another table, creating a link between them

### G

- **Golden record** — the single authoritative, unified record for an entity after deduplication and survivorship; the "best known truth"
- **Graph database** — database that stores entities as nodes and relationships as edges; optimised for traversing connections (e.g. Neo4j)

### I

- **Index** — a data structure that speeds up lookups on a column; like a book index — lets the database find rows fast without scanning everything
- **Ingestion** — the process of pulling or receiving data from source systems into a platform
- **Instance** — one row / one record in a table — one specific example of an entity

### J

- **JOIN** — SQL operation that combines rows from two tables based on a matching condition (usually FK = PK)
- **JSON (JavaScript Object Notation)** — common text format for structured data; uses key-value pairs and nesting; widely used in APIs

### K

- **Kafka** — distributed event-streaming platform; connects producers and consumers via named topics; events are stored in an append-only log
- **Key** — a field (or combination of fields) used to uniquely identify or link records

### L

- **Latency** — delay between when data is generated at the source and when it is available in the destination
- **Logical data model** — data model that defines fields and relationships in detail but is not tied to a specific database technology

### M

- **Master data** — the core reference data about key entities (devices, users, locations) that is shared across systems; should be consistent and authoritative
- **MDM (Master Data Management)** — practice + tooling for creating and maintaining a single authoritative version of master data across the organisation
- **Metadata** — data about data — describes the properties of a dataset (e.g. when it was created, who owns it, what fields it contains)

### N

- **Normalisation** — in data modelling: organising tables to reduce redundancy. In data platforms: standardising field values to a consistent format
- **NoSQL** — category of databases that don't use fixed table schemas; includes document stores (MongoDB), key-value stores (Redis), graph DBs (Neo4j)
- **Null** — absence of a value in a field; not the same as zero or empty string — means the value is unknown or missing

### O

- **OLAP (Online Analytical Processing)** — database optimised for complex analytical queries across large datasets; data warehouses use OLAP
- **OLTP (Online Transaction Processing)** — database optimised for fast, frequent reads and writes; operational systems (CRMs, ticketing tools) use OLTP
- **Offset** — in Kafka: a consumer's current position in the event log; tracks how far through the stream they have read

### P

- **Partition** — dividing a large table or topic into smaller pieces for performance and scalability
- **Physical data model** — the actual database implementation: tables, columns, data types, indexes, constraints
- **Pipeline** — the sequence of steps that moves and transforms data from source to destination
- **Primary key (PK)** — unique identifier for a row in a table; no duplicates, no nulls
- **Producer** — in streaming: a system that sends events to a message bus
- **Projection** — selecting only specific columns from a query result (the SELECT part of SQL)

### Q

- **Query** — a request for data from a database; written in SQL for relational databases

### R

- **Record** — one row in a table; one instance of an entity
- **Relational database** — database that stores data in tables linked by keys; supports SQL; examples: PostgreSQL, MySQL, SQL Server
- **Retention** — how long data is stored before being deleted or archived; applies to data lakes, Kafka topics, warehouses
- **Row** — a single record in a table (synonym: record, tuple, instance)

### S

- **Schema** — the full definition of a database's structure: all tables, columns, data types, constraints, and relationships
- **Schema drift** — when a source system changes its data structure (adds/removes/renames fields) without notice, breaking downstream pipelines
- **Streaming** — processing data continuously as events arrive, rather than in scheduled batches
- **Survivorship** — rules that decide which field value becomes authoritative in a golden record when multiple sources conflict
- **Surrogate key** — a generated unique identifier (e.g. UUID, auto-increment integer) with no business meaning; used as PK when no natural key exists

### T

- **Table** — the basic storage unit in a relational database; has rows (records) and columns (fields)
- **Thresholding** — in entity resolution: deciding what to do with a confidence score (auto-merge, human review, or keep separate)
- **Timestamp** — a datetime value recording when something happened
- **Topic** — in Kafka: a named channel for a category of events; producers write to topics, consumers read from them
- **Tuple** — another word for a row/record in a relational database

### U

- **Upsert** — database operation meaning: update if the record exists, insert if it doesn't (merge of UPDATE + INSERT)

### V

- **View** — a virtual table defined by a query; doesn't store data itself, just a saved query you can treat like a table
- **Void / Null handling** — how a system deals with missing values: reject, default, flag, or pass through

### W

- **Webhook** — a push mechanism where a source system sends data to a destination automatically when an event occurs (vs. polling/pulling)

---

*For concept explanations with examples, see `aem-foundation-data-concepts.md`. For visual diagrams of data flow, see `aem-foundation-data-concepts-visuals.md`.*
