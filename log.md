# Fix Bank Wiki — Log

Append-only. Never edit past entries. Format: `## [YYYY-MM-DD] operation | description`

---

## [2026-05-23] init | Wiki created from flat Fix Bank (entries #001–#047)

- Migrated 47 entries from flat Fix Bank format into 9 category files.
- Created schema.md, index.md, and this log.
- Categories: 01 data flow & wiring (8), 02 expressions & encoding (5), 03 node config & infrastructure (8), 04 cloud function & gcs (5), 05 data sources & config architecture (4), 06 llm prompt & schema contracts (7), 07 multi-llm pipelines & review panels (5), 08 debugging process & session discipline (8), 09 document generation (2).
- Next lint pass due after entry #057.

## [2026-05-23] ingest | Added entries #048–#051 from May 2026 error-reporting workflow build handover

- #048 → 03: native n8n node vs hand-built HTTP for instance execution reads
- #049 → 03: IF v2.2/Switch v3.2 conditions.options.version: 2 required by instance, not enforced by SDK
- #050 → 03: SDK workflow create drops maxTries/waitBetweenTries silently; post-create verification required
- #051 → 03: workflow-level executionTimeout silently capped by plan tier (300s on Starter)
- Updated index.md: total entries 47 → 51, category 03 entry count 8 → 12
- Cross-references added: #016 ↔ #051, #020 ↔ #050

## [2026-05-23] lint | Consolidation pass — entries #001–#051

- No merges required. All 51 entries are distinct.
- No category reassignments required. All entries correctly placed.
- No stale content found.
- 4 cross-references added: #019 → +#033; #043 → +#018; #028 → +#038; #049 → +#031
- Category patterns updated: cat-01 (named-node ref root pattern clarified), cat-05 (#038 link in patterns), cat-06 (#033 link in patterns)
- Consolidation record filed: consolidation-2026-05-23.md
- Stale files flagged for manual deletion (4 files — see consolidation doc for IDs)
- Next lint pass due: after entry #061

---

## Session — 27 May 2026 — TAAFT Prompt of the Day Tracker (Vps4llY2NUg2hnSZ)
- Debug session. Diagnosed junk-row output: Google Sheets node defineBelow mapping had literal column-name strings instead of expressions, writing header text into every row. Fixed mappings to expressions; switched append → appendOrUpdate matching on Notion Link for dedup; corrected a pre-existing IF-node lint error (notEmpty missing singleValue). All three live-verified via executions 928 + 929 (ran twice; 6 prompts → 6 rows, no duplication).
- New Fix Bank entry: #052 (category 01) — Google Sheets defineBelow literal-string mappings.
- Total entries 51 → 52.
