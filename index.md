# Fix Bank Wiki — Index

_Read this first at session start. Fetch only the category files relevant to your task._

**Total entries:** 73 (distinct IDs #001–#073, contiguous; #033 is dual-filed — full in cat 01, cross-ref stub in cat 08)
**Last updated:** June 2026
**Next lint pass due:** after entry #083 (10 entries from the current baseline of #073)

_Last consolidated 2026-06-22 (see `consolidation-2026-06-22.md`): ingested #053–#073; split category 03 → new category 11 (SDK/MCP build-path); deduped a stray full copy of #033 in cat 08; corrected cat 01/06/08 counts._

---

## Categories

| # | File | Description | Entries | Last Updated |
|---|------|-------------|---------|--------------|
| 01 | `01-data-flow-and-wiring.md` | Cross-branch refs, merge node behaviour (input cap + mode), field passthrough in loops, HTTP payload replacement, two-source items[0], multi-consumer field misses | 10 | June 2026 |
| 02 | `02-expressions-and-encoding.md` | JSON stringify, markdown fence stripping, array coercion, fieldPath syntax, escape-level misreads | 5 | May 2026 |
| 03 | `03-node-config-and-infrastructure.md` | Webhook registration, response modes, retry-on-fail, timeouts, credential rebinding (type swap), updateNode wipes, display name drift, dead feature flags, plan-tier timeout cap, Sheets tab selection, else-if dead branches, binary accessor | 12 | June 2026 |
| 04 | `04-cloud-function-and-gcs.md` | Binary payload sizing, signed URLs, IAM self-binding, Cloud Function auth, Python null handling, per-API GCP enablement | 6 | June 2026 |
| 05 | `05-data-sources-and-config-architecture.md` | Google Docs markup stripping, verifying output from execution data, single source of truth, field naming normalisation, controlled-vocab case-insensitivity + seed reconciliation | 5 | June 2026 |
| 06 | `06-llm-prompt-and-schema-contracts.md` | Output schema mismatches, read-only field exclusion, prompt staleness, hardcoded string drift, slug-key documentation, phantom fields | 6 | June 2026 |
| 07 | `07-multi-llm-pipelines-and-review-panels.md` | Rubric/reviewer as coordinated unit, half-shipped feature flags, single-reviewer-decides semantics, LLM numeric gates, sparse-input estimate bias, structured-output reliability | 6 | June 2026 |
| 08 | `08-debugging-process-and-session-discipline.md` | Hand-edit corruption, version-log pre/post state, transient-platform-issue discipline, MCP attribution + multi-server views, stopping before unverified deploy, dedup-guard retest, exec truncation, verifier-harness discipline, verbatim-block typos | 11 | June 2026 |
| 09 | `09-document-generation.md` | python-pptx slide deletion, template row cap truncation | 2 | May 2026 |
| 10 | `10-edge-workers-web-app.md` | HMAC sign-over-verifier-message, Workers-only Web Crypto, auth-gated deploy hand-off, multipart pass-through | 4 | June 2026 |
| 11 | `11-n8n-sdk-mcp-tooling.md` | Native node vs hand-built reads, SDK-vs-instance validation gaps, n8n-mcp validator false-positives, SDK create drops/rejects, credential-schema vacuous-if, SDK static parse, credential binding via SDK/MCP | 7 | June 2026 |

---

## Entry → Category map

| Entry | Category |
|-------|----------|
| #001 | 03 |
| #002 | 01 |
| #003 | 02 |
| #004 | 03 |
| #005 | 05 |
| #006 | 01 |
| #007 | 02 |
| #008 | 04 |
| #009 | 02 |
| #010 | 04 |
| #011 | 01 |
| #012 | 09 |
| #013 | 01 |
| #014 | 04 |
| #015 | 04 |
| #016 | 03 |
| #017 | 06 |
| #018 | 01 |
| #019 | 06 |
| #020 | 03 |
| #021 | 04 |
| #022 | 09 |
| #023 | 03 |
| #024 | 03 |
| #025 | 06 |
| #026 | 03 |
| #027 | 05 |
| #028 | 05 |
| #029 | 05 |
| #030 | 07 |
| #031 | 03 |
| #032 | 08 |
| #033 | 01 (+08 cross-ref) |
| #034 | 06 |
| #035 | 06 |
| #036 | 06 |
| #037 | 02 |
| #038 | 02 |
| #039 | 08 |
| #040 | 08 |
| #041 | 08 |
| #042 | 08 |
| #043 | 01 |
| #044 | 07 |
| #045 | 07 |
| #046 | 07 |
| #047 | 07 |
| #048 | 11 |
| #049 | 11 |
| #050 | 11 |
| #051 | 03 |
| #052 | 01 |
| #053 | 05 |
| #054 | 04 |
| #055 | 03 |
| #056 | 01 |
| #057 | 08 |
| #058 | 07 |
| #059 | 08 |
| #060 | 08 |
| #061 | 11 |
| #062 | 03 |
| #063 | 03 |
| #064 | 11 |
| #065 | 11 |
| #066 | 11 |
| #067 | 01 |
| #068 | 08 |
| #069 | 08 |
| #070 | 10 |
| #071 | 10 |
| #072 | 10 |
| #073 | 10 |

---

## Retired & dual-filed IDs

- **Retired (merged-away) IDs:** none. All IDs #001–#073 are live and contiguous.
- **Dual-filed:** #033 — full entry in cat 01 (data-flow), cross-reference stub in cat 08 (debugging discipline). Counts once toward the total, twice toward category placements (placements sum to 74).

---

_For schema and operations, see `schema.md`._
