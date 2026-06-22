# Consolidation Pass — 22 June 2026

## Scope

Entries #001–#073. First pass since the May 2026 baseline (#001–#051). Triggered by the June 2026 V3-build ingest (#053–#073, 21 new entries) which took the bank well past the "after #061" threshold in a single pass. Full pass (Phases 0–6) plus the SDK/MCP category split.

---

## Phase 0 — Privacy sweep

Scanned every file (index, log, schema, all category files, prior consolidation doc) for private information per `schema.md`. **0 findings.** The June ingest was scrubbed on the way in (client/project names, GCP project + number, SA emails, credential/workflow/exec IDs, gids, repo names, instance URLs, MCP server hashes, `N0x` node labels). The prior consolidation doc's Drive IDs were already redacted. Confirmed clean before this record was written.

---

## Phase 1 — Count & integrity reconciliation

Found and fixed pre-existing integrity drift that the May pass missed:

| Issue | Before | After |
|-------|--------|-------|
| `#033` triple-filed | Full entry in cat 01 **and** a stray full copy in cat 08, **plus** the intended `#033-cross` stub in cat 08 | Full entry in cat 01; `#033-cross` stub in cat 08. Stray full copy removed. |
| cat 01 entry count | header 11, actual 10 | 10 |
| cat 06 entry count | header 7, actual 6 (drift predating May pass) | 6 |
| cat 08 entry count | header 13, actual 12 (→ 11 after #033 dedup) | 11 |

Reconciliation after fixes: **73 distinct IDs** (#001–#073, contiguous), **74 placements** (the single legitimate #033 dual-file). Category counts sum to 74. `index.md` Total = 73. Consistent.

---

## Phase 2 — Merge candidates

**No ID merges.** The 21 June entries were deduped during ingest; remaining related pairs were examined and retained as cross-referenced separates:

- **#059 and #060** — complementary, not duplicate. #059 = the tooling limit (full-mode exec truncation); #060 = the discipline (verifier must assert inputs). Cross-referenced.
- **#069 and #040/#042** — distinct. #069 = divergent views across two MCP servers; #040/#042 = version-log pre-state and change attribution. Cross-referenced.
- **#066 and #026** — distinct. #066 = credential **binding** mechanics via SDK/MCP; #026 = credential **rebinding** on a node-type swap at runtime. Cross-referenced across the cat 11 / cat 03 boundary.
- **#063 and #002** — distinct. #063 = binary **accessor** version-sensitivity; #002 = binary **property name** indexing. Cross-referenced.
- **#053** folded four June proposals (case-only, separator, and genuine-mismatch vocabulary classes + the required-field seed-gap disposition) into one entry at ingest — recorded here, not re-split.

---

## Phase 3 — Placement & cross-references

**Category split (the one reassignment):** category 03 had grown to 19 entries — the "fetch only the relevant file" model breaks down at that size, and the SDK/MCP build-path entries are a distinct concern (how the tooling behaves at build time) from runtime node config. Created **category 11 — n8n SDK / MCP Build-Path & Tooling** and moved 7 entries:

| Moved | Was | Now |
|-------|-----|-----|
| #048 native node vs hand-built reads | 03 | 11 |
| #049 SDK validator ≠ instance validator | 03 | 11 |
| #050 SDK create drops retry timings | 03 | 11 |
| #061 n8n-mcp validator false-positives | 03 | 11 |
| #064 credential-create vacuous-`if` | 03 | 11 |
| #065 SDK static-parse method ban | 03 | 11 |
| #066 credential binding via SDK/MCP | 03 | 11 |

IDs are global, so all inbound cross-references survived the move untouched. Cat 03 retains 12 runtime/config entries; cat 11 holds 7. Both files have fresh category-level patterns.

**Cross-references added (3):**

| Entry | Was | Now | Rationale |
|-------|-----|-----|-----------|
| #049 (cat 11) | #031 | #031, #061 | Both are "one validator passes, another rejects" — the SDK/instance/heuristic validation gap. |
| #026 (cat 03) | none | #066 | Credential rebinding (type swap) ↔ credential binding (SDK/MCP mechanics). Same concern, two layers. |
| #046 (cat 07) | #045 | #045, #058 | Both concern LLM-derived numbers driving control flow — #046 hard gates/loops, #058 sparse-input estimate bias. |

---

## Phase 4 — Stale content

None. All entries current; no superseded claims or fixed-platform-bug entries to archive.

---

## Phase 5 — Category-level patterns

- **Cat 03** patterns rewritten to reference only retained entries; SDK-side notes now point to cat 11.
- **Cat 11** patterns written fresh: validators disagree (#049, #061); SDK/MCP write paths silently drop/reject (#050, #064, #065); binding is its own step (#066); prefer the native node (#048).
- **Cats 01, 04, 05, 07, 08, 10** pattern sections were refreshed during the June ingest and confirmed current here.

---

## Phase 6 — Write-back

- Updated all category headers (counts, Last updated) and `index.md` (Total, Categories table, entry→category map, dual-filing note).
- Added category 11 to `index.md` and `schema.md` folder structure.
- No Retired list needed (no merges); dual-filing of #033 documented in `index.md`.
- Reset lint counter (see below) and appended a generic session block to `log.md`.

---

## Next pass due: after entry #083 (10 entries from current baseline of #073)
