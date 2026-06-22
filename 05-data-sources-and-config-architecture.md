# 05 — Data Sources & Config Architecture

_Google Docs markup stripping, verifying node output from actual execution data, single source of truth for config, field naming normalisation at source boundaries._

**Entry count:** 5
**Last updated:** June 2026
**Related categories:** 06-llm-prompt-and-schema-contracts, 01-data-flow-and-wiring

---

## Entries

## #005 — Field name drift between form/source and downstream node

**Symptom:** Downstream node validation fails on required fields that "should be there." Fields visible in webhook payload but not picked up by the consuming node.
**Cause:** Source system uses one naming convention (e.g. camelCase from a Vercel form) and the consuming node expects another (e.g. snake_case). No automatic normalisation.
**Platform:** n8n, any
**Node types / context:** Any node consuming external input (form, API, webhook) with declared expected field names
**Fix:** Add a normaliser block at the top of the consuming node — explicit mapping from source convention to expected convention. Cover all known source field variants defensively.
**Spec rule:** When a workflow consumes input from an external source (form, API, third-party webhook), the spec must declare the source's exact field names alongside the workflow's internal field names, and specify the normalisation mapping. Do not assume conventions match.
**First seen:** May 2026, n8n
**Related:** #007
**Last updated:** May 2026

## History

---

## #027 — Google Docs node plain-text output strips structural markup

**Symptom:** A parser depending on tables, headings, or pipe-delimited rows works against locally-tested input but fails against the actual Google Docs node output, returning empty matches or wrong groupings. Markdown bold/italic markers and table pipes appear absent in runtime data.
**Cause:** n8n's `googleDocs` node `get` operation returns plain text by default. Tables become whitespace, `**bold**` becomes plain text, `###` headings become plain text. Structural markup is gone.
**Platform:** n8n + Google Docs API
**Node types / context:** `n8n-nodes-base.googleDocs` operation `get`, downstream Code nodes parsing the `content` field.
**Fix:** Two paths. (a) If structure is genuinely needed, switch to `n8n-nodes-base.googleDrive` `download` operation with `text/html` export — preserves tables, headings, lists. (b) Better: evaluate whether the data should live in a Google Sheet instead. Prose docs lose structure; spreadsheets are column-structured by design.
**Spec rule:** When a workflow needs to read structured data from a Google Doc, prefer a Google Sheet as the source. If a Doc is required, do not assume markdown survives the n8n round-trip — read the actual node output during spec design and validate the parser against it before committing to a structure-dependent approach.
**First seen:** May 2026, n8n
**Related:** #028, #029
**Last updated:** May 2026

## History

---

## #028 — Verify node output shape from execution data, not API documentation

**Symptom:** Multiple iterations of parser code before discovering the actual node output had a shape different from all assumptions. Time spent designing parsers against assumed structure that doesn't match reality.
**Cause:** Reasoning about node output from API documentation, intuition, or test fixtures rather than from real execution data on the live system.
**Platform:** any (most relevant for n8n, Cloud Functions, third-party API integrations)
**Node types / context:** Any node where Code/parser logic depends on the upstream node's output structure.
**Fix:** Before writing parser code, pull a recent successful execution where the upstream node ran (`n8n_executions` `get` with `mode: filtered, nodeNames: [target]` for n8n) and inspect the actual output shape. Confirm field names, value types, and structural assumptions. The cost is one MCP call; the saving is iterations of wrong parser code.
**Spec rule:** Parser-style code in any node spec must be designed against an example of the actual upstream output, not a notional schema. The spec should reference a captured execution sample as ground truth.
**First seen:** May 2026, n8n
**Related:** #027, #029, #038
**Last updated:** May 2026

## History

---

## #029 — Single source of truth beats two-source cross-check for config data

**Symptom:** Designing redundant-encoding cross-checks (parse two encodings of the same data, fail loudly on mismatch) when the architectural problem is that the data shouldn't have two encodings to begin with. Cross-check logic accumulates complexity to paper over the dual-source design.
**Cause:** Trying to make a human-readable document do double duty as machine-readable config. Cross-checks treat the symptom; the disease is one source serving two audiences with conflicting needs.
**Platform:** any (most relevant for config sources stored as docs/sheets/JSON)
**Node types / context:** Any workflow node consuming config that lives in a Google Doc / Notion page / similar prose-friendly format.
**Fix:** Move machine-readable config to a structured store: Google Sheet (column-driven), JSON file in a known location, or n8n static workflow data. Keep prose explanations where humans need them, separately.
**Spec rule:** When a config source is proposed as a Google Doc, ask explicitly: "is this a Doc because it requires prose explanation for humans, or is it a Doc because nobody thought about it?" If the latter, it should be a Sheet. Specs that designate a Doc as machine-config must justify the choice against the Sheet alternative.
**First seen:** May 2026, n8n + Google Docs/Sheets
**Related:** #027, #028
**Last updated:** May 2026

## History

---

## #053 — Controlled-vocabulary validators must match case-insensitively, and seed data must be reconciled to the canonical vocabulary as a data task

**Symptom:** A validator that checks free-text values (service names, categories, headings, roles) against a canonical list rejects valid records — e.g. 100% of submissions fail with "wrong heading," or an `update` on an existing record 400s using that record's *own* stored values (`Invalid services_delivered: AV / Production, …`). The canonical list is lowercase; the inbound/stored data is Title Case or uses legacy spellings.
**Cause:** Two compounding issues. (1) The comparison is case-sensitive (`list.includes(value)`) while the canonical list is lowercase and the data is not — and a shared `normalise` step does not canonicalise case. (2) Seed/reference data carries non-canonical vocabulary in several classes: case-only differences; separator/spacing differences (`Talent/Entertainment` vs `talent / entertainment`); and genuine mismatches (legacy short names like `Venue`, retired terms) — often far more widespread than assumed (e.g. the majority of rows, not "a couple").
**Platform:** any (n8n Code-node validators; applies to any controlled-vocabulary check)
**Node types / context:** Intake/validation Code nodes comparing inbound or stored values against a canonical allowlist for add/update actions. Also covers required-field validators backed by a downstream consumer.
**Fix:** (1) Compare case-insensitively — lowercase both sides (`VALID.includes(String(s).toLowerCase())`). Audit every validator: where multiple nodes check the same vocabulary, one is usually the outlier still comparing case-sensitively. (2) Treat residual divergence (separators, legacy/genuine mismatches) as a **one-time seed-data normalisation**, not a validator alias map — baking a legacy→canonical map into the validator violates the single-source rule (#035, #029). Measure the spread before migrating and surface unmappable tokens for an owner decision. (3) For a required field backed by a downstream consumer (e.g. a field that drives generation), reconcile empty seed data by **backfilling**, not by relaxing the validator.
**Spec rule:** Any controlled-vocabulary check against a canonical list is case-insensitive by spec. Seed/reference data must be reconciled to the canonical vocabulary as a data task before the validator ships — do not encode legacy aliases in the validator. When a required field is backed by a downstream consumer, the spec resolves empty seed data by backfilling, not by weakening the requirement. The web/client layer should already submit canonical (lowercase) values; the validator failures usually come from re-posting stored legacy data, so fix the store.
**First seen:** June 2026, n8n (proposal-drafting + content-intake workflows — services/category validation)
**Related:** #005, #029, #035
**Last updated:** June 2026

## History

(none — new entry; consolidates the proposal-side and content-side instances and the seed-data-vs-validator disposition)

---

## Category-level patterns

- **Test against real execution output, not assumed shape** (#028). One MCP call to pull execution data saves multiple wrong-parser iterations. This applies to any source boundary, not just Google Docs. See also #038 for the related pattern: even when you pull real execution data, what you inspect via the read transport may not be what the system actually stores — parse before inspecting.
- **Google Docs are for humans; Google Sheets are for machines** (#027, #029). Two entries from different angles reach the same conclusion: don't use Docs as config. If you must, verify what the n8n node actually returns before writing any parser against it.
- **Normalise at the source boundary, document the mapping** (#005). Field name conventions diverge silently. The spec must own the mapping, not leave it to the runtime.
- **Validators are case-insensitive; the data is the thing you fix** (#053). When a controlled-vocabulary check fails on valid records, lowercase both sides — then reconcile divergent seed data as a one-time data task rather than teaching the validator every legacy spelling. Aliases in the validator recreate the dual-source problem (#029, #035).
