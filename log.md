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

## Session — 27 May 2026 — Google Sheets tracker workflow
- Debug session. Diagnosed junk-row output: Google Sheets node defineBelow mapping had literal column-name strings instead of expressions, writing header text into every row. Fixed mappings to expressions; switched append → appendOrUpdate matching on Notion Link for dedup; corrected a pre-existing IF-node lint error (notEmpty missing singleValue). All three live-verified via executions 928 + 929 (ran twice; 6 prompts → 6 rows, no duplication).
- New Fix Bank entry: #052 (category 01) — Google Sheets defineBelow literal-string mappings.
- Total entries 51 → 52.

## [2026-06-22] ingest | Added entries #053–#073 from the June 2026 multi-component V3 build (n8n main + content-intake workflows, Cloud Function, Cloudflare Pages web app)

Triaged ~28 proposed entries from four `proposed-fix-bank.md` files (Testing / Content Management / Main Workflow / Web App build dirs). Folded duplicates, merged related proposals, scrubbed private identifiers (client/project names, GCP project + number, SA emails, credential/workflow/exec IDs, gids, repo names, instance URLs, MCP server hashes).

- #053 → 05: controlled-vocabulary validators match case-insensitively; seed data reconciled to canonical vocab as a data task (folds PFB-T01, T06, T06-AMENDED, both T09; do not bake legacy aliases into the validator)
- #054 → 04: Google APIs are enabled per-API per-project; Sheets enabled ≠ Docs enabled (PFB-T02)
- #055 → 03: Sheets tab selection by mode "name", not a "gid=" value (PFB-T03)
- #056 → 01: a Code node fed by two sources reads a non-deterministic items[0]; wire one source, named-ref the other (PFB-T04)
- #057 → 08: dedup/idempotency guard silently blocks fix-and-retest of the same fixture (PFB-T05)
- #058 → 07: LLM-derived numeric estimate driving a flag is biased on sparse input; fix the estimator, don't blind-widen the threshold (PFB-T07)
- #059 → 08: large executions truncate (~1MB) in full mode; pull filtered output-only, persist to disk (PFB-T10)
- #060 → 08: deterministic verifier/scorer must assert inputs present; uniform failures across all cases = harness bug (PFB-T11)
- #061 → 03: n8n-mcp validator false-positives (v4.5 Sheets writes, second-output branches, sub-nodes, Code-node heuristics); verify vs execution history / SDK validator / get_node_types (folds PFB-T12 + MW-C)
- #062 → 03: Code-node action routing — if/else-if chains make later branches for the same action unreachable (CM-001)
- #063 → 03: binary-buffer access in Code nodes (this.helpers.getBinaryDataBuffer) is version-sensitive — runtime-verify item (CM-002)
- #064 → 03: n8n public-API credential create — absent conditional toggles make schema if's vacuously true, forcing unrelated required fields (CM-003)
- #065 → 03: n8n Workflow SDK code is statically parsed; only SDK calls/operators, no Array/String methods (CM-004)
- #066 → 03: credential binding via SDK/MCP — auth mode ≠ binding; predefined/generic keys need updateNode; verify post-write (folds CM-005 + MW-D + MW-F)
- #067 → 01: Merge node caps at 10 inputs; combine-vs-append mode determines item shape (folds MW-A + MW-B)
- #068 → 08: spec "verbatim from live" code blocks can carry transcription typos; patch the live node with deltas (MW-E)
- #069 → 08: multiple MCP servers on one instance present divergent views; verify via the writing server / active graph (MW-G)
- #070 → 10: HMAC — sign over the exact bytes the verifier checks (base64url string), not the raw payload (WA-01)
- #071 → 10: crypto.subtle.timingSafeEqual is a Workers extension, not standard Web Crypto (WA-02)
- #072 → 10: Cloudflare deploy is auth-gated; Git-integration is the canonical hand-off — don't treat deploy as automatable (WA-03)
- #073 → 10: multipart pass-through — re-append every field, never reconstruct, don't set Content-Type manually (WA-04)

- Amendments (no new number): #047 (panel degrades gracefully = working; missing max_tokens → truncation) from PFB-T08; #050 (reconfirmed on create_workflow_from_code path) from CM-006.
- Dropped: PFB-T10 (WITHDRAWN — no defect; its "diff live vs spec" process-note is captured in #068/#069).
- New category created: 10 — Edge / Workers Web App (Cloudflare Pages Functions); 4 entries (threshold met). Added to schema.md and index.md.
- Index counts: 01 9→11, 03 12→19, 04 5→6, 05 4→5, 07 5→6, 08 8→13, 10 new=4. Total 52 → 73.
- LINT PASS NOW OVERDUE (baseline #061; bank at #073). Flagged in index.md. Recommended: split category 03 (19 entries) into an SDK/MCP-tooling sub-category; reconcile entry→category map (#033 lives in 01); privacy re-scan before next mirror publish.
- Public mirror NOT refreshed — user must run publish-wiki.sh.

## [2026-06-22] lint | Consolidation pass — entries #001–#073 (see consolidation-2026-06-22.md)

- Phase 0 privacy sweep: 0 findings across all files (June ingest scrubbed on the way in; prior consolidation doc already redacted).
- Phase 1 integrity fixes: removed a stray full copy of #033 from cat 08 (kept the #033-cross stub; #033 remains full in cat 01); corrected entry counts cat 01 11→10, cat 06 7→6, cat 08 13→11. Reconciles to 73 distinct IDs / 74 placements.
- Phase 2: no ID merges. Related pairs retained as cross-referenced separates (#059/#060, #069/#040-#042, #066/#026, #063/#002).
- Phase 3: split category 03 (had reached 19) → new **category 11 — n8n SDK / MCP Build-Path & Tooling**. Moved #048, #049, #050, #061, #064, #065, #066 (cat 03 → 11). IDs global, cross-refs intact. Cat 03 now 12, cat 11 now 7. Added 3 cross-refs: #049→+#061; #026→+#066; #046→+#058.
- Phase 4: no stale content.
- Phase 5: rewrote cat 03 and cat 11 category-level patterns; other categories' patterns confirmed current.
- Phase 6: updated index.md (counts, Categories table incl. new row 11, entry→category map, dual-filing note) and schema.md folder structure. No retired IDs (no merges). Lint counter reset.
- Next lint pass due: after entry #083.
- Public mirror NOT refreshed — user must run publish-wiki.sh.
