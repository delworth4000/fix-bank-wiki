# 06 — LLM Prompt & Schema Contracts

_Output schema mismatches between LLM and downstream validators, read-only field exclusion from LLM context, pre-build prompt staleness, hardcoded string drift, slug-key documentation, phantom fields (produced by no node)._

**Entry count:** 7
**Last updated:** May 2026
**Related categories:** 07-multi-llm-pipelines-and-review-panels, 05-data-sources-and-config-architecture

---

## Entries

## #017 — Output schema mismatch between LLM and downstream validator

**Symptom:** Validator Code node throws "missing field" on a field the LLM was instructed to produce. LLM output looks correct on inspection but key naming differs from validator's expectation.
**Cause:** LLM prompt instructs one schema, validator expects another, and the spec didn't pin down which is canonical. Common variants: `revised_content` vs `revisedContent` vs `content`; nested vs flat structure.
**Platform:** n8n, any LLM
**Node types / context:** LLM node followed by a Code-based validator or schema check
**Fix:** Inspect the actual LLM output and the validator's expected schema. Reconcile to a single canonical schema documented in the spec. Update either the LLM prompt or the validator to match — pick the schema that is most consistent with the rest of the workflow.
**Spec rule:** Every LLM node's output schema must be declared explicitly in the spec, with field names, types, and shapes. Downstream validators must reference the same schema definition. No reliance on the LLM "figuring out" what the downstream wants from prompt context alone.
**First seen:** May 2026, n8n
**Related:** #009
**Last updated:** May 2026

## History

---

## #019 — Phantom field consumed by multiple nodes, written by none

**Symptom:** A Code node, IF gate, or other consumer reads `$json.X` (or `data.X`). The path is wired correctly, nodes turn green, but `X` is always null/empty/missing. Downstream behaviour silently fails (e.g. an IF routes to the wrong branch, an array-of-rows produces zero rows).
**Cause:** No node anywhere in the workflow writes `X`. The field was specified in the design but the writer was never implemented or was removed in a refactor.
**Platform:** n8n, any
**Node types / context:** Any field referenced across multiple nodes.
**Fix:** Workflow-wide grep on the field name. Confirm at least one node writes it, not just reads. If nothing writes it, either (a) add the writer, (b) remove the readers, or (c) eliminate the field from the contract entirely.
**Spec rule:** The spec must explicitly identify the writer of every cross-node field. A field with consumers but no producer is a structural error. Verify before any test run.
**First seen:** May 2026, n8n
**Related:** #013, #017, #033
**Last updated:** May 2026

## History

---

## #025 — Cross-node field required by schema but read-only to the writer is a latent drift surface

**Symptom:** An LLM agent (editor, transformer, summariser) is told "do not change field X" in its system prompt, but the JSON schema for its return shape requires X to be present. The agent reproduces X in every response. Occasionally — non-deterministically — X arrives slightly altered, and the alteration propagates.
**Cause:** The contract is internally inconsistent. By making X mandatory in the schema, the writer must produce X. By telling the writer not to change X, the writer is asked to reproduce it byte-for-byte. LLMs reproduce structured data with high but imperfect fidelity. Over many runs, drift is statistically certain.
**Platform:** n8n, any LLM-in-the-loop architecture
**Node types / context:** Editor / reviser / transformer LLM nodes that operate on multi-field structured payloads.
**Fix:** Strip the read-only field from the contract entirely. Remove it from the user-message context sent to the LLM. Remove it from the return schema. Downstream nodes that need the original value resolve it from the original payload via named-node references (`$('Pre-Editor Node').first().json.field`). The LLM never sees the field, never returns it, cannot drift it.
**Spec rule:** When designing an LLM-in-the-loop transformation, audit every field in the I/O contract: which fields the LLM is *allowed* to change, and which it is *not*. Read-only fields must be excluded from the LLM's context and return schema entirely. If a downstream node needs the original value of a read-only field, resolve it via named-node ref to the pre-LLM source, not via the LLM's response.
**First seen:** May 2026, n8n (editor loop in revision pipeline)
**Related:** #017, #019
**Last updated:** May 2026

## History

---

## #034 — Pre-build prompt drafts go stale once the build introduces real contracts

**Symptom:** A prompt draft written during spec design is queued for installation. By the time the build reaches that node, the surrounding contracts have evolved. Installing the draft verbatim introduces silent contradictions: the prompt asks for a structure the downstream code does not consume, references fields that no longer exist, or hardcodes values that have since been parameterised.
**Cause:** Prompt drafts written before the build cannot anticipate the data shapes and contracts that emerge during implementation. The draft is internally coherent against its design-time assumptions but stale against live reality.
**Platform:** any LLM-in-the-loop architecture
**Node types / context:** Any LLM node where the prompt was drafted ahead of the build and queued for later installation.
**Fix:** Before installing a pre-build prompt draft, audit it against the live build's contracts: (1) what does the upstream node actually send? (2) what does the downstream node actually consume? (3) what hardcoded values does the draft reference, and are they still right? If the contracts have drifted enough that more than minor updates are needed, treat the prompt as a planned spec exercise — open a dedicated session, lay the contracts on the table, and revise. Do not install a draft on a debug session.
**Spec rule:** Prompt drafts referenced in a spec must be flagged with the build state at draft time. When the build evolves past that state, the spec rev must mark the draft as "needs revision against current contracts" rather than "ready to install." Installation requires a contracts review, not just a copy-paste.
**First seen:** May 2026, n8n + multi-LLM pipeline
**Related:** #017, #030
**Last updated:** May 2026

## History

---

## #035 — Hardcoded reference strings in prompts drift from canonical input source

**Symptom:** A prompt enforces "match input exactly" against an external input (e.g. a SCOPE list) but also contains hardcoded mappings or lists referencing the same strings. The hardcoded versions drift away from the input format — em-dashes become hyphens, slashes gain or lose surrounding spaces — typically during human edits that prioritise readability over fidelity. The LLM then faces contradictory instructions and drifts under load; downstream validation rejects the output.
**Cause:** Prompt text often duplicates strings that should be canonical-by-reference. Once duplicated in prose, they accumulate independent edits that diverge over time. Prompt edits are typically made by humans reading for sense, not byte-comparing for fidelity.
**Platform:** any LLM prompt that references strings also enforced as exact matches against runtime input.
**Node types / context:** Any prompt template containing a mapping table, allowlist, or schema example whose strings overlap with downstream-validated runtime input.
**Fix:** Remove the duplication. The runtime input is the source of truth — anchor the prompt to it. Replace prose like "use this mapping table" with rules like "every string must be character-for-character identical to a string in INPUT." If a mapping IS needed, source the strings on the input side of the mapping from the same canonical store as the runtime input — Google Sheet, JSON file, or workflow static data. Do not paste them into prose.
**Spec rule:** Specs must not duplicate strings that are also runtime-validated. Where a mapping is unavoidable, the input-side strings must be loaded from the canonical store at runtime (e.g. injected via expression), not hardcoded in prompt prose. Reviewers should grep specs for hardcoded string lists and ask: "is this duplicating something the form/Sheet/database also defines?" If yes, replace with a runtime injection.
**First seen:** May 2026, n8n + multi-LLM pipeline (Node 7 mapping table)
**Related:** #005, #029, #033, #034
**Last updated:** May 2026

## History

(none — new entry)

---

## #036 — Slug-keyed lookup objects in prompts must document the slug rule

**Symptom:** A prompt instructs the LLM to look up a value by name in a provided object but the object is actually keyed by a slugified version of the name. The LLM either fails to find the key, infers the slug pattern from inspecting the object, or fabricates content. The downstream validator only checks output shape — not whether the lookup succeeded. Result: silent under-use of the lookup data, drop in output quality, no error.
**Cause:** Prompt-side documentation lags code-side transformations. A Code node introduces a slug transformation for a sensible reason, but the prompt that consumes the resulting object continues to talk about the original names.
**Platform:** any LLM-in-the-loop architecture where a prompt receives a lookup object whose keys are transformed from a more natural form.
**Node types / context:** Any LLM node receiving a key-value map as input where the keys are normalised, slugified, lowercased, or otherwise transformed from human-readable strings.
**Fix:** Document the transformation in the prompt explicitly. Include the rule (e.g. "lowercase, replace spaces and slashes with underscores") and at least three worked examples covering edge cases. Anchor the lookup instruction to the slug ("compute the slug for the service from SCOPE and look up GENERIC[slug]") rather than the original name.
**Spec rule:** When a lookup object passed into a prompt uses transformed keys, the prompt must document the transformation explicitly and provide worked examples. Prompts that say "look up X in Y" without documenting Y's key format are silent contracts. The spec must enumerate every prompt-passed object and either (a) state that keys match the original names verbatim, or (b) document the transformation rule.
**First seen:** May 2026, n8n + multi-LLM pipeline (Node 11 generic_descriptions slug)
**Related:** #005, #034, #035
**Last updated:** May 2026

## History

(none — new entry)

---

## Category-level patterns

- **Every LLM node needs an explicit I/O contract in the spec** (#017, #019, #025). Field names, types, shapes — declared, not assumed. A field with no declared writer is a structural error before a line of code runs. See also #033 for the related "secondary consumers not enumerated" variant.
- **Read-only fields must be excluded entirely from the LLM's context and schema** (#025). "Don't change X" is not enforceable; "never see X" is.
- **Prompt staleness is a deployment risk, not an authoring risk** (#034). Drafts go stale during the build. Installation requires a live contracts review.
- **Hardcoded strings in prompts drift** (#035, #036). Any string that is also runtime-validated should never be pasted into prompt prose — inject it from the canonical source or document the transformation rule explicitly.
