# AEM Data Concepts — Visual Reference

> Companion to `aem-foundation-data-concepts.md`. Read the notes first, then use these diagrams to see how the pieces connect.

---

## Visual 1 — The Big Picture: What AEM Is Doing

```
╔══════════════════════════════════════════════════════════════════════════════╗
║              AEM DATA PLATFORM — END TO END FLOW                            ║
╚══════════════════════════════════════════════════════════════════════════════╝

 SOURCE SYSTEMS            PIPELINE STAGES                      OUTPUT
 ───────────────           ───────────────────────────────      ──────────────

 ┌───────────┐
 │  Tool A   │ ──┐
 │ (endpoint │   │
 │  mgmt)    │   │    ┌──────────────┐     ┌──────────────┐
 └───────────┘   │    │              │     │ STANDARDISE  │
                 ├──▶ │  INGESTION   │ ──▶ │     &        │
 ┌───────────┐   │    │ (pull/push)  │     │  VALIDATE    │
 │  Tool B   │ ──┤    │              │     │              │
 │ (network  │   │    └──────────────┘     └──────┬───────┘
 │  scanner) │   │                                │
 └───────────┘   │                                ▼
                 │                        ┌──────────────┐
 ┌───────────┐   │                        │    ENTITY    │
 │  Tool C   │ ──┘                        │  RESOLUTION  │
 │   (HR /   │                            │  & DEDUP     │
 │  identity)│                            └──────┬───────┘
 └───────────┘                                   │
                                                 ▼
                                        ┌──────────────┐
                                        │    GOLDEN    │  ← one trusted record
                                        │    RECORD    │    per real-world entity
                                        │    LAYER     │
                                        └──────┬───────┘
                                               │
                             ┌─────────────────┼─────────────────┐
                             ▼                 ▼                 ▼
                       ┌──────────┐     ┌──────────┐     ┌──────────┐
                       │Dashboards│     │   APIs   │     │Downstream│
                       │& Reports │     │& Queries │     │  Tools   │
                       └──────────┘     └──────────┘     └──────────┘

KEY INSIGHT: Each source has partial, imperfect data about the same real-world
things. The platform's job is to combine them into one complete, trusted record.
```

---

## Visual 2 — Pipeline Stages: What Happens at Each Step

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    PIPELINE STAGES — WHAT EACH STEP DOES                    ║
╚══════════════════════════════════════════════════════════════════════════════╝

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  STAGE 1: INGESTION                                                      │
  │  ─────────────────                                                       │
  │  • Pull data from source systems (APIs, files, databases)                │
  │  • Or receive pushed data (webhooks, event streams)                      │
  │  • Question: how does data arrive? push or pull?                         │
  └──────────────────────────────┬───────────────────────────────────────────┘
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  STAGE 2: VALIDATION                                                     │
  │  ──────────────────                                                      │
  │  • Check format rules (is this a valid IP? valid date? non-empty ID?)    │
  │  • Reject or quarantine records that fail                                │
  │  • Question: what happens to malformed records?                          │
  └──────────────────────────────┬───────────────────────────────────────────┘
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  STAGE 3: STANDARDISATION (NORMALISATION)                                │
  │  ────────────────────────────────────────                                │
  │  • Casing: WKSTN-007 → wkstn-007                                        │
  │  • OS names: win11, Windows 11, Win 11 → "Windows 11"                   │
  │  • Dates: DD/MM/YYYY → YYYY-MM-DD                                        │
  │  • Nulls: "", "N/A", "unknown" → null                                   │
  │  • Question: which fields need controlled vocabularies?                  │
  └──────────────────────────────┬───────────────────────────────────────────┘
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  STAGE 4: ENTITY RESOLUTION & DEDUPLICATION                              │
  │  ──────────────────────────────────────────                              │
  │  • Match records across sources using identifiers                        │
  │  • Score confidence, apply thresholds                                    │
  │  • Merge matched records, apply survivorship rules                       │
  │  • Question: when do two records count as the same thing?                │
  └──────────────────────────────┬───────────────────────────────────────────┘
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  STAGE 5: GOLDEN RECORD STORAGE                                          │
  │  ───────────────────────────────                                         │
  │  • Store the unified, trusted record                                     │
  │  • Optionally also preserve raw source data (for reprocessing/audit)     │
  │  • Question: what version of the data is stored?                         │
  └──────────────────────────────────────────────────────────────────────────┘
```

---

## Visual 3 — Entity Resolution: Step by Step

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                  ENTITY RESOLUTION — HOW IT WORKS                           ║
╚══════════════════════════════════════════════════════════════════════════════╝

  ALL INCOMING RECORDS (potentially thousands)
          │
          ▼
  ┌───────────────┐
  │   STEP 1:     │  Problem: comparing every record vs. every other = O(n²)
  │   BLOCKING    │  Solution: group into candidate pairs that MIGHT match
  │               │  Example: only compare records that share an email domain
  └───────┬───────┘            or same first 4 chars of serial number
          │
          │  smaller set of candidate pairs
          ▼
  ┌───────────────┐
  │   STEP 2:     │  Compare each candidate pair, assign a confidence score
  │   MATCHING    │
  │               │  serial_number exact match     → score: 1.0  (STRONG)
  │               │  hostname match + same dept     → score: 0.85 (MEDIUM)
  │               │  fuzzy name only                → score: 0.5  (WEAK)
  └───────┬───────┘
          │
          │  confidence scores for each pair
          ▼
  ┌───────────────────────────────────────────────────┐
  │   STEP 3: THRESHOLDING — what to do with scores   │
  └───────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼──────────────┐
        │             │              │
        ▼             ▼              ▼
  score ≥ 0.9    0.6–0.89       score < 0.6
        │             │              │
        ▼             ▼              ▼
  AUTO-MERGE    HUMAN REVIEW    KEEP SEPARATE
  (confident)    QUEUE           (distinct entities)
        │        (borderline)
        └────────────┐
                     ▼
           ┌─────────────────┐
           │    STEP 4:      │  Which value wins when two sources disagree?
           │  SURVIVORSHIP   │
           │                 │  Most recent?   ─── owner last updated
           │                 │  Source ranking ─── HR trusted over endpoint tool
           │                 │  Non-null first ─── prefer any value over blank
           └────────┬────────┘
                    │
                    ▼
           ╔═════════════════╗
           ║  GOLDEN RECORD  ║  ← single authoritative record
           ║                 ║    for this real-world entity
           ╚═════════════════╝
```

---

## Visual 4 — Worked Example: 3 Sources → 1 Golden Record

```
╔══════════════════════════════════════════════════════════════════════════════╗
║         WORKED EXAMPLE: THREE SOURCES → ONE GOLDEN RECORD                   ║
╚══════════════════════════════════════════════════════════════════════════════╝

  SOURCE A (endpoint tool)       SOURCE B (network scanner)    SOURCE C (HR)
  ─────────────────────────      ─────────────────────────     ─────────────
  hostname: WKSTN-007            hostname: wkstn-007            serial: ABC999
  serial:   ABC999               ip:       10.0.0.45            owner:  bob@agency
  os:       Windows 11           os:       win11                dept:   Finance
  owner:    alice@agency

        │                               │                            │
        └───────────────────────────────┼────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  NORMALISATION                                                           │
  │  ─────────────                                                           │
  │  WKSTN-007 → wkstn-007   (casing standardised — now A matches B)        │
  │  win11     → Windows 11  (OS name standardised — now A matches B)       │
  └─────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  MATCHING                                                                │
  │  ────────                                                                │
  │  A + B: hostname = wkstn-007 (exact match after normalise) → MATCH ✓   │
  │  A + C: serial   = ABC999    (exact match)                  → MATCH ✓   │
  │  ∴ A, B, C all refer to the same device                                 │
  └─────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  SURVIVORSHIP — resolving conflicts                                      │
  │  ──────────────────────────────────                                      │
  │  owner:  A says alice@, C says bob@                                      │
  │          Rule: HR system is authoritative for ownership → bob@ wins      │
  │  ip:     only source B has it → use it (non-null preference)             │
  │  dept:   only source C has it → use it (non-null preference)             │
  └─────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
  ╔═════════════════════════════════════════════════════════════════════════╗
  ║  GOLDEN RECORD                                                          ║
  ║  ─────────────                                                          ║
  ║  hostname: wkstn-007          (standardised)                            ║
  ║  serial:   ABC999             (strong identifier — matched all sources) ║
  ║  os:       Windows 11         (standardised from win11)                 ║
  ║  owner:    bob@agency.gov.sg  (HR wins survivorship)                    ║
  ║  dept:     Finance            (enriched from source C)                  ║
  ║  ip:       10.0.0.45          (enriched from source B)                  ║
  ╚═════════════════════════════════════════════════════════════════════════╝
```

---

## Visual 5 — Data Quality: What Each Dimension Affects

```
╔══════════════════════════════════════════════════════════════════════════════╗
║           DATA QUALITY DIMENSIONS — WHAT BREAKS WHEN THEY'RE POOR           ║
╚══════════════════════════════════════════════════════════════════════════════╝

  DIMENSION      WHAT IT MEANS              WHAT BREAKS WHEN IT'S POOR
  ─────────────  ─────────────────────────  ──────────────────────────────────
  ACCURACY       Value reflects reality     Survivorship picks wrong value.
                                            Golden record is wrong even though
                                            it was built correctly.

  COMPLETENESS   Required fields are        Matching fails — can't match on
                 present / non-null         a field that doesn't exist.
                                            Record exists but is operationally
                                            useless (no owner, no ID).

  CONSISTENCY    Same thing represented     Normalisation can't fix what it
                 same way across sources    doesn't know about. Same device
                                            looks like two different devices.
                                            Dedup fails silently.

  TIMELINESS     Data is current enough     Accurate values from 6 months ago.
                 for the use case           Owner was reassigned. Patch was
                                            applied. Record looks fine but is
                                            operationally stale.

  VALIDITY       Values conform to          Malformed records pass ingestion.
                 expected format/range      IP fields with garbage, dates in
                                            year 2099, cause silent failures
                                            downstream.

  ─────────────────────────────────────────────────────────────────────────────

  WHICH DIMENSION CAUSES WHICH TYPE OF FAILURE:

  ┌──────────────────────────────────────────┬──────────────┬──────────────────┐
  │ Failure type                             │ Dimension(s) │ Symptom          │
  ├──────────────────────────────────────────┼──────────────┼──────────────────┤
  │ Same device exists as two records        │ Consistency  │ Inflated counts  │
  │ (false negative — missed dedup)          │ Completeness │                  │
  ├──────────────────────────────────────────┼──────────────┼──────────────────┤
  │ Two different devices merged into one    │ Accuracy     │ Wrong ownership  │
  │ (false positive — wrong dedup)           │ Validity     │ alerts           │
  ├──────────────────────────────────────────┼──────────────┼──────────────────┤
  │ Golden record has wrong field value      │ Accuracy     │ Decisions made   │
  │                                          │ Timeliness   │ on false info    │
  ├──────────────────────────────────────────┼──────────────┼──────────────────┤
  │ Record can't be matched to anything      │ Completeness │ Orphan records   │
  │ (no strong identifiers present)          │              │ never unified    │
  └──────────────────────────────────────────┴──────────────┴──────────────────┘
```

---

## Visual 6 — Field Strength: Identifier Spectrum

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    IDENTIFIER STRENGTH SPECTRUM                              ║
╚══════════════════════════════════════════════════════════════════════════════╝

  WEAK                                                               STRONG
  ◄──────────────────────────────────────────────────────────────────────►
  Use with caution                                        Prefer for matching

  Display      IP          Hostname      Email         Serial      Device
  name         address     (with         address       number      certificate /
                           casing fix)                             cloud ID

  Why weak:                              Why strong:
  ─────────                              ───────────
  • Can be shared                        • Globally unique
  • Often reused (recycled hostnames)    • Stable over time
  • Formatted differently per source    • Not reused after decommission
  • Frequently missing                   • Rarely formatted inconsistently
  • Typos common (display names)         • Hard to accidentally duplicate

  ─────────────────────────────────────────────────────────────────────────
  DESIGN RULE:
  • Use strong identifiers as PRIMARY match keys
  • Use weak identifiers as SECONDARY signals (to boost or lower confidence)
  • Never auto-merge on a weak identifier alone
```

---

## Visual 7 — ETL vs ELT: Where Transformation Happens

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                         ETL  vs  ELT                                        ║
╚══════════════════════════════════════════════════════════════════════════════╝

  ETL — Transform BEFORE loading
  ───────────────────────────────

  [Source] ──▶ [Extract] ──▶ [TRANSFORM] ──▶ [Load] ──▶ [Target/DW]
                              ▲
                              transformation happens HERE
                              (pipeline cleans data before it lands)

  Target receives: clean, structured, ready-to-use data
  Raw data:        NOT preserved (unless you store it separately)

  Good when:
  ✓ Target has limited compute (legacy DW, relational DB)
  ✓ PII must be masked before it touches the target environment
  ✓ Source data is too messy to store in raw form
  ✓ Storage is expensive — only store what's needed


  ELT — Transform AFTER loading
  ───────────────────────────────

  [Source] ──▶ [Extract] ──▶ [Load] ──▶ [TRANSFORM] ──▶ [Transformed layer]
                                          ▲
                                          transformation happens HERE
                                          (inside the target, after raw lands)

  Target receives: raw data first, then transforms run on top
  Raw data:        PRESERVED — can be reprocessed if rules change

  Good when:
  ✓ Target is a powerful cloud DW (BigQuery, Snowflake, Redshift)
  ✓ Team wants to keep raw data for reprocessing / audit
  ✓ Multiple consumers need different transforms from same raw data
  ✓ Schema is evolving — don't want to re-extract when rules change


  ─────────────────────────────────────────────────────────────────────────
  SIDE BY SIDE:

  Question                         ETL              ELT
  ───────────────────────────────  ───────────────  ────────────────────
  Where does transform happen?     In the pipeline  Inside the target
  What lands in the target?        Clean data        Raw data (first)
  Is raw data preserved?           No (usually)      Yes
  If rules change, what happens?   Re-extract        Re-run transform
  Target needs heavy compute?      No                Yes
  Good for PII/compliance?         Yes               Requires access controls
  ─────────────────────────────────────────────────────────────────────────

  AEM likely uses ELT: raw agency data lands first → transforms + entity
  resolution run inside the platform → golden records produced.
  Raw data kept for audit trail and rule reprocessing.
```

---

## Visual 8 — Batch vs Streaming: When Data Moves

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                      BATCH  vs  STREAMING                                   ║
╚══════════════════════════════════════════════════════════════════════════════╝

  BATCH — Data moves in scheduled chunks
  ────────────────────────────────────────

  Source           Pipeline             Target
  ──────           ────────             ──────
  [events          [collects            [target updated
   accumulate]      everything]          in one go]
      │                │                    │
  ────┼────────────────┼────────────────────┼────  TIME ──▶
      │                │                    │
   09:00            23:00               23:05
   (events         (batch job           (target now
    happen)         runs)                reflects day)

  Data age: up to 24 hours old (daily batch)
  Complexity: LOW — easier to build, operate, retry
  Use when: daily reports, weekly compliance, inventory reviews


  STREAMING — Data moves continuously
  ─────────────────────────────────────

  Source           Pipeline             Target
  ──────           ────────             ──────
  [event happens] ──▶ [processed] ──▶ [target updated]   ← seconds later
  [event happens] ──▶ [processed] ──▶ [target updated]
  [event happens] ──▶ [processed] ──▶ [target updated]

  Data age: seconds to minutes old
  Complexity: HIGH — must handle ordering, retries, duplicates, schema drift
  Use when: real-time alerts, security events, automated triggers

  ─────────────────────────────────────────────────────────────────────────
  DECISION GUIDE:

         Does the use case break if data is
         more than a few minutes old?
                     │
            ┌────────┴────────┐
            │                 │
           YES                NO
            │                 │
            ▼                 ▼
        STREAMING           BATCH
        (justify with     (simpler, cheaper,
         real need)        easier to operate)

  Rule of thumb: default to batch. Upgrade to streaming only when there
  is a genuine operational requirement for low-latency data.
```

---

## Visual 9 — How Everything Connects

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    THE FULL PICTURE — HOW IT ALL CONNECTS                   ║
╚══════════════════════════════════════════════════════════════════════════════╝

                          ┌───────────────────────────────────┐
                          │         SOURCE SYSTEMS            │
                          │  (multiple, partial, imperfect)   │
                          └─────────────────┬─────────────────┘
                                            │
                          PIPELINE CHOICE:  │  ETL or ELT?
                          Batch or stream?  │  (where transform happens)
                                            │
                          ┌─────────────────▼─────────────────┐
                          │           INGESTION               │
                          │      (pull / push / stream)       │
                          └─────────────────┬─────────────────┘
                                            │
                          ┌─────────────────▼─────────────────┐
                          │    VALIDATION + NORMALISATION      │
                          │                                    │
                          │  Data Quality gates:               │
                          │  • Validity  — format checks       │
                          │  • Consistency — standardise       │
                          │  • Completeness — flag nulls       │
                          └─────────────────┬─────────────────┘
                                            │
                          ┌─────────────────▼─────────────────┐
                          │       ENTITY RESOLUTION           │
                          │                                    │
                          │  1. Blocking (narrow candidates)   │
                          │  2. Matching (score similarity)    │
                          │     — needs strong identifiers     │
                          │     — needs consistency to work    │
                          │     — needs completeness to work   │
                          │  3. Thresholding (auto / review)   │
                          │  4. Survivorship (which wins?)     │
                          │     — needs accuracy to be useful  │
                          │     — needs timeliness to be valid │
                          └─────────────────┬─────────────────┘
                                            │
                          ┌─────────────────▼─────────────────┐
                          │          GOLDEN RECORD            │
                          │   (single trusted representation  │
                          │    of each real-world entity)      │
                          └─────────────────┬─────────────────┘
                                            │
                     ┌──────────────────────┼──────────────────────┐
                     ▼                      ▼                      ▼
             ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
             │  Dashboards  │      │  Operations  │      │  Compliance  │
             │  & Reports   │      │  & Alerting  │      │  & Audit     │
             └──────────────┘      └──────────────┘      └──────────────┘

  ─────────────────────────────────────────────────────────────────────────
  WHY DATA QUALITY IS THE FOUNDATION:

  Poor CONSISTENCY  →  matching fails  →  same entity stays as 2 records
  Poor COMPLETENESS →  matching fails  →  orphan records never unified
  Poor ACCURACY     →  survivorship picks wrong value  →  golden record is wrong
  Poor TIMELINESS   →  golden record is stale          →  correct but outdated
  Poor VALIDITY     →  bad data passes ingestion        →  silent failures later
```

---

*Each diagram maps to a section in `aem-foundation-data-concepts.md`. Use them together.*
