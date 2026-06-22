# 03 — n8n Node Configuration & Infrastructure

_Webhook registration and response modes, retry-on-fail, execution timeouts, credential rebinding on node type swap, updateNode parameter wipe, display name drift, dead feature flags, plan-tier execution timeout cap, Sheets tab selection, Code-node action routing, binary accessor version-sensitivity. (SDK / MCP build-path tooling moved to category 11.)_

**Entry count:** 12
**Last updated:** June 2026
**Related categories:** 01-data-flow-and-wiring, 02-expressions-and-encoding, 11-n8n-sdk-mcp-tooling

---

## Entries

## #001 — Webhook returns "POST not registered" or 404 after configuration change

**Symptom:** Webhook URL returns 404 or "POST not registered" error after the webhook node's configuration has been edited and saved.
**Cause:** n8n does not always re-register the webhook with its internal routing table after a config change, even after save and publish.
**Platform:** n8n
**Node types / context:** Webhook trigger node
**Fix:** Deactivate the workflow, wait a few seconds, reactivate. The webhook re-registers on activation.
**Spec rule:** Any change to a Webhook node's configuration must be followed by deactivate/reactivate as part of the deployment checklist, not assumed to take effect on save.
**First seen:** Jan 2026, n8n
**Related:** none
**Last updated:** May 2026

## History

---

## #004 — Webhook responseMode "lastNode" causes validation errors in branched workflows

**Symptom:** Workflow validation fails or behaves unpredictably in workflows with branching, error handling, or multiple terminal nodes when webhook is set to respond from "last node."
**Cause:** "Last node" responseMode is ambiguous in workflows where multiple nodes can be the terminal node depending on path taken.
**Platform:** n8n
**Node types / context:** Webhook trigger with branched downstream topology
**Fix:** Set webhook responseMode to responseNode and add an explicit Respond to Webhook node at each terminal point.
**Spec rule:** Any workflow with branching or error paths must use explicit Respond to Webhook nodes, not "last node" mode. Spec must show which terminal node responds in which scenario.
**First seen:** Jan 2026, n8n
**Related:** none
**Last updated:** May 2026

## History

---

## #016 — Workflow timeout too short for long LLM chains

**Symptom:** Workflow execution truncated mid-flow with "execution timed out" after a chain of LLM calls. Individual node responses are within their own timeouts.
**Cause:** Workflow-level timeout is shorter than the cumulative time of the LLM chain. Default timeouts (often 5 minutes / 300s) are insufficient for multi-stage LLM workflows.
**Platform:** n8n
**Node types / context:** Any workflow with sequential LLM calls totalling >5 minutes
**Fix:** Set workflow executionTimeout explicitly. 720 seconds (12 minutes) is a reasonable starting point for multi-LLM workflows; tune upward if needed.
**Spec rule:** Specs must declare the workflow timeout based on a worst-case latency estimate (sum of slowest expected response per node). Do not rely on defaults. See also #051 — the instance plan tier may cap this value silently.
**First seen:** May 2026, n8n
**Related:** #051
**Last updated:** May 2026

## History

---

## #020 — All external-API nodes need retry-on-fail; one-shot calls die on transient 5xx

**Symptom:** Workflow execution dies with a 502/503/504 from an external API. Single transient failure kills the whole run; user must resubmit.
**Cause:** External-API nodes default to no retry. Transient 5xx errors are common at Google APIs, OpenRouter, GCS, Cloud Functions cold-start.
**Platform:** n8n
**Node types / context:** Every external-API node — googleDocs, googleSheets, gmail, httpRequest, chainLlm and friends.
**Fix:** Set retryOnFail: true, maxTries: 3, waitBetweenTries: 3000-5000ms on every external-API node. For slow calls (CF with 30-60s response time), use maxTries: 2 with waitBetweenTries: 10000ms.
**Spec rule:** Every external-API node must have retry-on-fail configured. The spec must specify maxTries and waitBetweenTries per node category, calibrated to expected call duration. Default-no-retry is a spec-time error. See also #050 (cat 11) — SDK create drops retry timing fields silently; post-create verification is required.
**First seen:** May 2026, n8n
**Related:** #050
**Last updated:** May 2026

## History

---

## #023 — Display name encoding model identity drifts when model is swapped

**Symptom:** A node display name describes the model in use (e.g. "Reviewer B (Qwen)") but the node's parameters call a different model. Downstream code using the display name as a label produces misleading output.
**Cause:** During an earlier model swap, the underlying model ID was changed but the display name was not.
**Platform:** n8n, any
**Node types / context:** HTTP Request nodes calling LLM APIs by model ID; multi-reviewer / multi-model patterns where each model has its own node.
**Fix:** When swapping a model, update the display name in the same operation. Audit: workflow-wide grep for old model identifiers in node names.
**Spec rule:** Display names that encode model identity must change when the underlying model changes. Consider keeping model identity out of display names entirely — use a stable role-based name (e.g. "Reviewer B") and document the model assignment in a separate models-in-use table.
**First seen:** May 2026, n8n
**Related:** none
**Last updated:** May 2026

## History

---

## #024 — Dead chainLlm config flag with wired-but-unused receiver

**Symptom:** A chainLlm node has a feature flag set (e.g. parameters.needsFallback: true) with a separate model node wired into the second ai_languageModel input slot. The wiring exists, the flag is set, but the fallback never fires.
**Cause:** chainLlm node implementations on n8n Cloud accept feature-flag parameters that the underlying code does not honour. The runtime ignores the parameter; the second wired input model is dead-but-present.
**Platform:** n8n
**Node types / context:** chainLlm and related LangChain-pattern LLM agent nodes.
**Fix:** Workflow-wide audit: any chainLlm node with needsFallback: true (or analogous) is suspect. Verify against current n8n source/docs which parameters the runtime honours. For confirmed dead config: remove the flag, remove the secondary wired model node, remove the connection.
**Spec rule:** Specs must assert which LLM-node parameters are runtime-honoured vs. accepted-but-ignored. When the spec calls for fallback behaviour on a chainLlm-class node, mark the requirement with a verification note: "chainLlm fallback is not honoured in the current n8n version — implement via Error Trigger workflow or manual If/Switch routing instead."
**First seen:** May 2026, n8n (Cloud Starter)
**Related:** #019
**Last updated:** May 2026

## History

---

## #026 — n8n node type swap requires explicit credential rebinding

**Symptom:** Missing required credential after updateNode operation that changes the node's type field. Workflow rolls back automatically with rollbackPerformed: true.
**Cause:** When updateNode changes the node type, the credentials field is wiped if not included in the updates. n8n requires the new credential type explicitly bound.
**Platform:** n8n
**Node types / context:** Any node type swap via updateNode operation, especially across Google services.
**Fix:** Before deploying a node type swap, survey existing credential bindings on other nodes of the target type using the workflow JSON. Find an applicable credential ID. Include it in updates.credentials alongside the type change.
**Spec rule:** Workflow specs that require migrating a node from one Google service to another must list the credential rebinding as an explicit deployment step, not assume it carries over from the prior node configuration.
**First seen:** May 2026, n8n
**Related:** #066
**Last updated:** May 2026

## History

---

## #031 — n8n updateNode wipes parameters fields not present in the partial update

**Symptom:** n8n_update_partial_workflow with an updateNode operation supplying a partial parameters object returns a validation error like "Missing or invalid required parameters: url" for the target node, even though url was correctly set before the operation. Workflow auto-rolls back.
**Cause:** updateNode with updates.parameters replaces the node's entire parameters object with the supplied value. Fields not present in the partial are wiped, not preserved.
**Platform:** n8n
**Node types / context:** Any updateNode operation that intends to change one field within parameters while leaving others alone. Most likely on httpRequest, code, googleSheets, googleDocs nodes.
**Fix:** Use patchNodeField with fieldPath "parameters.<field>" and a find/replace patch instead. patchNodeField targets a single field and leaves the rest of parameters intact. If many fields need to change atomically, use updateNode with the full parameters object (pull current state first, modify, send the complete object).
**Spec rule:** Workflow modification scripts must distinguish between full-object replacement (full parameters provided) and field-level patching (use patchNodeField). Default to patchNodeField for single-field changes.
**First seen:** May 2026, n8n
**Related:** #026
**Last updated:** May 2026

## History

---

## #051 — Workflow-level executionTimeout is silently capped by the n8n plan tier

**Symptom:** A workflow is configured with settings.executionTimeout: 600 (or higher). Executions are consistently cancelled at exactly 300 seconds regardless of the workflow setting. The executionTimeout value is saved correctly in the workflow. The 300-second cut-off is clean and repeatable, not a transient error.
**Cause:** On n8n Cloud (and self-hosted instances with EXECUTIONS_TIMEOUT_MAX set), the workflow-level executionTimeout cannot exceed the instance-level hard cap. The Starter plan has a 300-second cap enforced at the instance level. A workflow setting of 600 is silently capped at 300 with no warning on save.
**Platform:** n8n Cloud (plan-tier dependent); also affects self-hosted instances with EXECUTIONS_TIMEOUT_MAX set below the workflow's setting.
**Node types / context:** Any long-running workflow. Most common symptom: LLM pipeline with multi-iteration review loop that occasionally exceeds the cap.
**Fix:** Two options: (1) Upgrade the n8n plan — Pro tier and above have higher or configurable caps. (2) Reduce workflow runtime — profile the slowest nodes and optimise. On self-hosted: set EXECUTIONS_TIMEOUT_MAX in the instance environment.
**Spec rule:** Specs for workflows expected to run longer than 3 minutes must declare the target runtime, the expected execution timeout setting, and confirm the deployment platform's plan tier supports that timeout. Do not assume executionTimeout on the workflow takes effect — verify the instance-level cap matches or exceeds it.
**First seen:** May 2026, n8n Cloud Starter tier (proposal-drafting workflow, long-running edge case with two editor loop iterations)
**Related:** #016
**Last updated:** May 2026

## History

(none — new entry)

---

## #055 — n8n Google Sheets tab selection: use mode "name", not a "gid=" value

**Symptom:** A Google Sheets node fails with "Sheet with ID gid=NNN not found" even though the gid is correct and the service account has access. Other Sheets nodes in the same workflow that select the tab by name work fine.
**Cause:** The `sheetName` resourceLocator was set with `mode: "list"` and a `value` of `"gid=NNN"`. n8n does not resolve a `gid=`-prefixed value here; the prefixed string is the bug, independent of access or the gid being valid.
**Platform:** n8n
**Node types / context:** `n8n-nodes-base.googleSheets`, any operation, `sheetName` resourceLocator.
**Fix:** Select the tab by `mode: "name"` with the exact tab title. Diagnostic for "is it access or is it the selector": mint an SA token from the key and call `GET /v4/spreadsheets/{id}?fields=sheets.properties(sheetId,title)` to read the real tab titles/gids as the node's identity sees them — if those resolve, the `gid=` selector is the fault.
**Spec rule:** Specs for Sheets nodes must specify tab selection by name (`mode: "name"`, exact title), not by a `gid=` list value. If a gid must be used, store the bare numeric id, not the `gid=`-prefixed form.
**First seen:** June 2026, n8n (proposal-drafting workflow — Sheets read/write nodes)
**Related:** #052
**Last updated:** June 2026

## History

(none — new entry)

---

## #062 — Code-node action routing: `if / else if` chains make later branches for the same action unreachable

**Symptom:** An action-routing Code node (e.g. a validator switching on `action`) populates some fields but silently leaves others unset for certain actions — e.g. an update path runs with `entity_id = undefined` and matches no row, so the "update" no-ops or hits the wrong record. No error is thrown.
**Cause:** The branch logic uses `if (A || B) {…} else if (B || C) {…}`. An action value matched by the first branch can never reach a later `else if` that also lists it — the later branch is dead code. The author's intent (often stated in a comment like "fields already populated above") is that both run, which `else if` cannot deliver.
**Platform:** n8n (Code node), but a generic control-flow hazard
**Node types / context:** Any Code node that routes per-`action` with chained `if/else if` and assigns different field sets in different branches.
**Fix:** Trace each action value to exactly one branch. Either assign every field that action needs inside its first matching branch, or convert the dependent branches to standalone `if` blocks after the chain (e.g. a separate `if (update_x || update_y) { entity_id = body.entity_id; validate; }`) so update actions get both field population and id validation.
**Spec rule:** When a spec presents action-routing branch logic, it must map each action value to exactly one branch and confirm every field that action requires is set there. Reviewers should flag any action value appearing in both an `if` and a later `else if` of the same chain as dead code.
**First seen:** June 2026, n8n (content-intake workflow — action-aware validation node)
**Related:** none
**Last updated:** June 2026

## History

(none — new entry)

---

## #063 — Binary-buffer access in a Code node (`this.helpers.getBinaryDataBuffer`) is version-sensitive — treat as a runtime-verify item

**Symptom:** A Code node that reads an uploaded file via `this.helpers.getBinaryDataBuffer(index, prop)` throws on every execution of the binary path if the accessor is undefined in the running n8n version — even though the code is spec-mandated and validates clean. The n8n-mcp validator may also warn "use `$helpers` not `helpers`".
**Cause:** The binary-buffer accessor name varies by n8n version (`this.helpers` vs `$helpers`). The accessor is not exercised until a real binary payload runs the node, so a wrong accessor passes build and validation and only fails at runtime on the primary content path.
**Platform:** n8n (Code node, binary handling)
**Node types / context:** Any Code node that reads binary data (image/file upload) on a primary path that must "never throw."
**Fix:** Keep the spec-mandated accessor (`this.helpers.getBinaryDataBuffer(0, 'file0')` is the canonical Code-node form), but mark every binary-reading Code node as a must-runtime-test item — exercise the actual add-with-file path before sign-off. If it throws, switch to `$helpers.getBinaryDataBuffer(...)` and record the swap as a spec correction.
**Spec rule:** Specs must flag any Code node performing binary-buffer access as runtime-verify-required on the primary path, and name the fallback accessor. Validation passing is not evidence the accessor resolves in the deployed version. See #002 for the related binary-property-name hazard.
**First seen:** June 2026, n8n (content-intake workflow — add-with-image path)
**Related:** #002
**Last updated:** June 2026

## History

(none — new entry)

---

## Category-level patterns

- **Deactivate/reactivate is a deployment step, not a recovery step** (#001). It belongs in the checklist.
- **Defaults are wrong for production LLM workflows.** Timeout (#016, #051) and retry-on-fail (#020) both default to values appropriate for trivial workflows and wrong for LLM chains. Set both explicitly in the spec — and note the SDK-side retry-timing drop (#050, cat 11) means "set" is not "persisted" until you re-pull.
- **updateNode is a full replace, patchNodeField is a partial** (#031). Mixing these up causes silent parameter wipes. Default to patchNodeField.
- **Model swaps have two parts: the body and the label** (#023). Doing only one creates a silent lie in the topology.
- **Plan tier caps are silent** (#051). A workflow setting that appears correct may be overridden at the instance level with no warning. Verify the instance cap during spec design, not after the first timeout.
- **Credential rebinding is an explicit step on any node-type swap** (#026). Changing the type wipes `credentials`; bind the new one in the same operation. The SDK/MCP binding mechanics live in cat 11 (#066).
- **Resource-locator and accessor details are version- and value-sensitive** (#055 Sheets tab by name not `gid=`; #063 binary accessor). Both pass build/validation and fail only at runtime — exercise the real path.
- **Code-node branch logic must route each value once** (#062). `if/else if` chains silently strand later branches for the same value; trace every action to exactly one branch.
