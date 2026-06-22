# 01 — n8n Data Flow & Wiring

_Cross-branch references, merge node behaviour, field passthrough in loops, HTTP payload replacement at node boundaries, multi-consumer field misses._

**Entry count:** 10
**Last updated:** June 2026
**Related categories:** 02-expressions-and-encoding, 06-llm-prompt-and-schema-contracts

---

## Entries

## #002 — Webhook binary file uploaded under unexpected property name

**Symptom:** Downstream node (e.g. Read PDF, Extract from File) throws "binary property not found" or processes empty data, even though the webhook clearly received the file.
**Cause:** Webhook node stores uploaded files under `{fieldName}0`, `{fieldName}1`, etc. — appending an index even for a single-file upload. Downstream nodes referencing `{fieldName}` directly find nothing.
**Platform:** n8n
**Node types / context:** Webhook trigger (binary uploads), any downstream node consuming binary data
**Fix:** Set the downstream node's `binaryPropertyName` to `{fieldName}0` (e.g. `file0`, not `file`). For multi-file uploads, iterate over `{fieldName}0`, `{fieldName}1`, etc.
**Spec rule:** When specifying webhook → binary-consumer chains, the binary property name in the consumer must explicitly account for the indexed naming convention. Document the exact property name in the spec, not the form field name.
**First seen:** Jan 2026, n8n
**Related:** none
**Last updated:** May 2026

## History

---

## #006 — Branches assumed to run in parallel actually run sequentially

**Symptom:** A merge / aggregator node downstream of multiple parallel branches fires before all branches complete, processing partial input. `itemCount` at the merge is lower than the number of branches.
**Cause:** n8n is single-threaded. Even with execution order v0, branches do not run in parallel — they interleave. Any Code-based "merge" that assumes all branches have produced output by the time it runs is wrong.
**Platform:** n8n
**Node types / context:** Any workflow with multiple parallel branches feeding a downstream node
**Fix:** Replace the Code-based merge with n8n's native Merge node. The Merge node waits for all configured inputs to arrive before firing. Configure the number of inputs to match the number of branches.
**Spec rule:** Any spec showing parallel branches must explicitly use a Merge node where the branches converge. Never spec a Code node as the convergence point. The Merge node's input count must match the branch count exactly.
**First seen:** May 2026, n8n
**Related:** #013
**Last updated:** May 2026

## History

---

## #011 — Leaf nodes referenced cross-branch via $("NodeName") never execute

**Symptom:** Node referenced by another node via `$("NodeName")` returns undefined or empty data. Workflow execution path shows the referenced node never ran.
**Cause:** n8n only executes nodes whose output flows into the actively-needed downstream path. A leaf node not wired into the consuming branch's input chain will not run, even if another node references it by name.
**Platform:** n8n
**Node types / context:** Any cross-branch reference using `$("NodeName")` syntax
**Fix:** Wire the referenced node into the consuming branch's actual data path. If the data needs to be available cross-branch, route it through a Merge node or a shared upstream node.
**Spec rule:** Specs must show every data dependency as an actual wired connection. `$("NodeName")` references are valid syntax but the spec must verify the referenced node is on the executing path. Treat cross-branch references as a smell — prefer explicit wiring.
**First seen:** May 2026, n8n
**Related:** #006
**Last updated:** May 2026

## History

---

## #013 — Wiring verified but only one of N parallel inputs reaches the merge

**Symptom:** Merge node or aggregator runs but `itemCount` shows 1 (or fewer than N). Execution path appears to show only one upstream branch executing where N were expected.
**Cause:** Wiring confirmed visually doesn't guarantee data flow. Common causes: an upstream node returning empty output silently filters its branch out; the Merge node is configured for the wrong number of inputs; one branch errored without halting the workflow.
**Platform:** n8n
**Node types / context:** Merge node, Code-based aggregators
**Fix:** Pull the actual outputs of each upstream branch from the execution trace and verify each produced data. Verify Merge node input count matches branch count. Check for silent-error continuation on individual branches.
**Spec rule:** After any topology change involving merges or parallel branches, the verification step must inspect the actual data at the merge point, not just whether nodes turned green. "Wired" ≠ "ran" ≠ "produced data."
**First seen:** May 2026, n8n
**Related:** #006
**Last updated:** May 2026

## History

---

## #018 — HTTP Request nodes strip upstream JSON fields, breaking downstream $json.X refs

**Symptom:** A Code node downstream of an HTTP Request node reads `$json.X` (or `data.X`) and gets undefined/null/empty even though `X` was populated by a Code node further upstream. The intermediate HTTP Request response replaced the JSON payload.
**Cause:** HTTP Request nodes return their HTTP response body as the new `$json`. Anything in the upstream payload that's not in the response is dropped.
**Platform:** n8n
**Node types / context:** Any Code node downstream of an HTTP Request node, where the Code node needs fields populated by an earlier (pre-HTTP) Code node.
**Fix:** Resolve upstream fields via `$('Node Name').first().json.X` named-node references. Walk back to the last Code node before the HTTP boundary. Do not rely on `$json` of the immediate upstream.
**Spec rule:** For every Code node downstream of an HTTP Request, the spec must declare which fields are needed from before the HTTP boundary, and reference them via named-node refs. Inline `$json.X` for pre-HTTP fields is a spec-time error.
**First seen:** May 2026, n8n
**Related:** #011
**Last updated:** May 2026

## History

---

## #033 — Multi-consumer fields silently miss updates when only the planned consumer is documented

**Symptom:** A node's output shape is changed. The handover lists which downstream node consumes it, the change is implemented, the consumer is updated. A second, unmentioned consumer further downstream breaks at the next execution because nobody remembered it was reading the same source.
**Cause:** Cross-cutting fields (a rubric, a config object, a shared lookup) often have more than one reader, but handover notes typically name only the primary consumer. The secondary consumer's read is not surfaced during the change.
**Platform:** any (n8n with named-node references is particularly prone)
**Node types / context:** Any field referenced by multiple downstream nodes via named-node references (`$('Source').first().json.X` or `.all()`). Common with rubrics, classification maps, lookup tables, shared config payloads.
**Fix:** Before changing a node's output shape, do a workflow-wide search for references to the source node. In n8n, grep the workflow JSON for `$('<Source Node Name>')` — every match is a consumer. Update each one in the same change set. Document each consumer by name in the change log so the next handover doesn't re-lose them.
**Spec rule:** Specs must enumerate every consumer of every cross-node field, not just the canonical or primary one. When a shape change is planned, the spec rev must list every consumer to update — silent secondary readers are a structural error.
**First seen:** May 2026, n8n
**Related:** #019, #025
**Last updated:** May 2026

## History

---

## #043 — Loop-internal node refetches payload from pre-loop source, discarding loop-produced state

**Symptom:** A workflow with a feedback or revision loop appears to apply edits that never surface in downstream output. The loop runs another iteration, but the second iteration's reviewers/consumers see the original pre-loop data, not the edited version. Investigation reveals the edited content was present at the loop's output but is silently replaced inside the next iteration.
**Cause:** A node inside the loop references payload data from a fixed pre-loop node via `$('Node Outside Loop').first().json` and overwrites the incoming `data.X` with it. `.first()` resolves to the first (and only) execution of that pre-loop node — so on every iteration, the loop-internal node discards whatever the loop produced and re-injects the original state.
**Platform:** n8n
**Node types / context:** Code node (or any node-level reference) that reads from a fixed upstream source via `$('NodeName').first()` while operating inside a loop. Especially common when refactoring a non-looped workflow into a looped one.
**Fix:** Workflow-wide audit: in every node inside the loop, identify references to nodes outside the loop. Distinguish two kinds of read:
 - **Genuine pre-loop data** (rubric, config sheet that runs once) — `.first()` references are correct.
 - **Pipeline state that should pass through the loop** (content, payload, anything the loop can modify) — must come from the incoming data path (`items[0].json` or the prior node's output), not from a pre-loop refetch.
For pipeline state, replace the `$('NodeName').first()` reference with `data.X` from incoming items, or use iteration-aware logic (try the most recent loop-internal source, fall back to the pre-loop source on first iteration).
**Spec rule:** Specs for any workflow containing a loop must explicitly distinguish between (a) static pre-loop data and (b) pipeline state that the loop can modify. Loop-internal nodes must read static data via `.first()` and pipeline state via incoming data. The spec must call out each cross-node reference inside the loop and classify it as one of the two.
**First seen:** May 2026, n8n (proposal-drafting workflow with editor-revision loop)
**Related:** #011, #018, #033
**Last updated:** May 2026

## History

(none — new entry)

---

## #052 — Google Sheets defineBelow mapping with literal column-name strings writes header text into every row

**Symptom:** Every row appended by a Google Sheets node contains the literal column names ("Title", "Notion Link", "Description", …) as the cell values, instead of the intended data — on every row, every run, regardless of what the upstream node produced. The upstream execution trace shows correct data reaching the node; the data is discarded at the mapping step.
**Cause:** In the Google Sheets node's `columns` config with `mappingMode: defineBelow`, each entry in `columns.value` must be an n8n expression that resolves to the cell content (e.g. `={{ $json.Title }}`). If the value is a bare literal string equal to the column name (e.g. `"Title"`), n8n writes that literal string into the cell. The misconfiguration is silently accepted — no validation error — because a constant string is a legal mapping value.
**Platform:** n8n (Google Sheets node, append / appendOrUpdate operations)
**Node types / context:** n8n-nodes-base.googleSheets v4.x, operation append or appendOrUpdate, mappingMode defineBelow. Most likely after hand-editing a node or importing a template where the value side was left as the column name.
**Fix:** Set each `columns.value` entry to an expression: `={{ $json.Title }}`, and for keys containing spaces use bracket notation: `={{ $json["Notion Link"] }}`. Verify by reading back the node config and confirming each mapping value begins with `=`. A bare string value (no leading `=`) that happens to equal the column name is the tell.
**Spec rule:** Every `defineBelow` mapping value in a Sheets-writing node must be an expression. A bare string equal to the column name is a spec-time error. Specs for Sheets append/upsert nodes must show the mapping in expression form (`={{ $json.X }}`), never as the column name alone. Add a read-back verification step ("confirm values start with `=`") to the post-build check for any node using defineBelow.
**First seen:** May 2026, n8n (a Google Sheets tracker workflow — junk header-text rows)
**Related:** #038 (literal-vs-expression confusion in a different context — MCP read transport)
**Last updated:** May 2026

## History

(none — new entry)

---

## #056 — A Code node fed by two upstream sources reads the wrong items[0]

**Symptom:** A Code node that expects its primary payload at `items[0]` intermittently processes the wrong record — e.g. it treats a config/rubric row as the main payload, and downstream consumers receive `undefined`. The failure tracks execution timing, not input data.
**Cause:** Two upstream branches are both wired into the node's main input (input 0). n8n concatenates inputs in arrival order, so `items[0]` is whichever source completes first — non-deterministic. The node was only ever meant to read one source from its input and the other via a named reference.
**Platform:** n8n
**Node types / context:** Any Code node with a single logical "primary" payload that is nonetheless wired to receive a second dataset on the same input. Especially common where a spec already says one source is "named-ref'd, no downstream merge" but the wiring contradicts it.
**Fix:** Wire ONLY the primary source into the consumer's main input; read the secondary dataset via a named reference (`$('Source Node').all()` / `.first()`). `removeConnection` the secondary→consumer edge — the secondary node still executes via its own trigger path. Make the spec's "no downstream merge" note an actual wiring constraint, not just prose.
**Spec rule:** A Code node's main input must carry exactly one logical dataset. Any additional dataset it needs must be declared as a named-reference read, and the spec's wiring diagram must not draw a second edge into that input. Treat "two edges into input 0" as a structural error wherever `items[0]` is read positionally.
**First seen:** June 2026, n8n (multi-LLM proposal-drafting workflow — reviewer input node)
**Related:** #011, #018, #043
**Last updated:** June 2026

## History

(none — new entry)

---

## #067 — n8n Merge node: max 10 inputs, and combine-vs-append mode determines item shape

**Symptom:** Two distinct failures from the same node. (a) A Merge configured for >10 inputs saves without error but `validate_workflow` rejects it ("numberInputs must be one of 2…10") and inputs 11+ are silently orphaned at runtime. (b) A Merge in `combine / combineByPosition` mode feeding a consumer that iterates `items[0..n-1]` produces ONE item with most fields overwritten — silent content loss that passes validation.
**Cause:** (a) The native Merge node accepts `numberInputs` 2–10 only; the save-time API does not enforce the cap that the validator/runtime does. (b) `combine + combineByPosition` merges items at the same position across inputs into a single item (last-writer-wins on field collisions); `append` (the default, no `mode`) concatenates all inputs' items in input order. Choosing the wrong mode mismatches what the consumer expects.
**Platform:** n8n
**Node types / context:** `n8n-nodes-base.merge`, and any aggregator/Code node downstream that reads merged output either as merged fields off one item or as N separate items.
**Fix:** (a) For >10 convergent inputs, build a multi-stage merge tree (e.g. two 6-input merges → a 2-input merge), preserving append order so positional `items[0..N]` still holds; or replace the merge with a Code-node aggregator that reads sources by named reference (order-independent). (b) Use `append` when the consumer iterates items as N separate records; use `combine/combineByPosition` only when the consumer reads merged fields off a single item. Confirm `numberInputs` is set explicitly — n8n does not infer it from wired connections.
**Spec rule:** Specs must (a) never call for a single Merge with >10 inputs — specify the merge tree or a named-ref aggregator; and (b) state the Merge mode explicitly alongside what the consumer reads ("N separate items" → append; "one merged item" → combineByPosition). A merge mode left unstated is a spec-time error.
**First seen:** June 2026, n8n (proposal-drafting workflow — 12 content-doc readers converging)
**Related:** #006, #013
**Last updated:** June 2026

## History

(none — new entry)

---

## Category-level patterns

- **"Wired" does not mean "ran" does not mean "produced data."** Three distinct failure points; verify each independently at the merge/aggregation point (#006, #013).
- **Named-node refs (`$('X').first()`) are load-bearing.** They silently fix the wrong thing when used inside loops (#043) or across HTTP boundaries (#018). Always audit what they resolve to in context. Both are instances of the same root pattern: `.first()` in the wrong context returns stale or misscoped data.
- **Multi-consumer fields are a persistent blind spot.** The spec names the primary consumer; secondary consumers accumulate silently and break on shape changes (#033, #011). See also #019 for the related "field has consumers but no producer at all" variant.
- **Mapping values must be expressions, not constants.** A Sheets defineBelow value left as the bare column name writes header text into every row (#052). Read back the node config and confirm mapping values begin with `=`.
- **A node's main input must carry one logical dataset; the Merge node has hard limits.** Two edges into input 0 make `items[0]` timing-dependent (#056). The Merge node caps at 10 inputs and its mode (append vs combineByPosition) silently changes item shape (#067). Both are caught only by inspecting actual data at the convergence point, not by "is it wired."
