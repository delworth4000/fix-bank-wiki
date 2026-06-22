# 03 — n8n Node Configuration & Infrastructure

_Webhook registration and response modes, retry-on-fail, execution timeouts, credential rebinding on node type swap, updateNode parameter wipe, display name drift, dead feature flags, native n8n node vs hand-built HTTP, IF/Switch metadata gaps, SDK retry timing drop, plan-tier execution timeout cap._

**Entry count:** 12
**Last updated:** May 2026
**Related categories:** 01-data-flow-and-wiring, 02-expressions-and-encoding

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
**Spec rule:** Every external-API node must have retry-on-fail configured. The spec must specify maxTries and waitBetweenTries per node category, calibrated to expected call duration. Default-no-retry is a spec-time error. See also #050 — SDK create drops retry timing fields silently; post-create verification is required.
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
**Related:** none
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

## #048 — Reading the instance's own executions: use the native n8n node, not a hand-built HTTP call

**Symptom:** A workflow that needs to read its own n8n instance's executions (e.g. a sweeper, error handler, or audit flow) reaches for $helpers.httpRequest to /api/v1/executions with an inline API key in a Code node. This fails with an auth or connection error on instances where credentials are managed externally.
**Cause:** Treating the instance's own public API as an arbitrary external HTTP target, when n8n ships a first-class node for it (n8n-nodes-base.n8n) with its own credential type (n8nApi).
**Platform:** n8n
**Node types / context:** Any workflow reading instance executions or workflow state from within n8n; sweepers, error handlers, audit flows.
**Fix:** Use the native n8n node (n8n-nodes-base.n8n, resource: execution, operations Get Many / Get) with an n8nApi credential. List the n8nApi credential as a deployment prerequisite. Important caveat: the native n8n node has a documented limitation on SSL — it may fail with a connection error on HTTPS Cloud instances. If this occurs, fall back to an HTTP Request node calling the same /api/v1/executions endpoint with the same API key credential. Test the native node first; if it errors on SSL, switch the node type, not the credential.
**Spec rule:** When a workflow must read its own instance's executions or workflows, the spec must specify the native n8n node + n8nApi credential, list the credential as a deployment prerequisite, and note the SSL fallback to HTTP Request. Do not spec a hand-built HTTP call with an inline API key.
**First seen:** May 2026, n8n (timeout-sweeper + error-handler build)
**Related:** #018, #028
**Last updated:** May 2026

## History

(none — new entry)

---

## #049 — IF v2.2 / Switch v3.2 require conditions.options.version 2; SDK validator does not enforce it; instance save rejects it

**Symptom:** An IF node (typeVersion 2.2+) or Switch node (typeVersion 3.2+) passes validate_workflow in the SDK with no errors, but saving or updating it on the live instance is rejected with "Missing required field conditions.options.version. Expected value: 2". The workflow is not saved.
**Cause:** The filter-node conditions.options block requires version: 2 on the n8n instance, alongside caseSensitive, leftValue, and typeValidation. The SDK pre-deploy validator does not check for this field, so the gap only surfaces at instance save time.
**Platform:** n8n
**Node types / context:** IF v2.2+, Switch v3.2+ nodes built or edited via the SDK / code path (not via the UI editor, which injects the field automatically).
**Fix:** Always include conditions.options.version: 2 when constructing filter nodes via code. The full required conditions.options block is: { "version": 2, "caseSensitive": true, "leftValue": "", "typeValidation": "strict" }. If a save is rejected for this reason, add the missing field via patchNodeField and re-save.
**Spec rule:** Specs that include IF v2.2+/Switch v3.2+ nodes must specify the full conditions.options block including version: 2. Do not rely on SDK validation alone to catch filter-node metadata gaps — the validator does not enforce it.
**First seen:** May 2026, n8n (error-handler build — E3 save rejection)
**Related:** none
**Last updated:** May 2026

## History

(none — new entry)

---

## #050 — Retry timings (maxTries/waitBetweenTries) dropped silently on SDK workflow create

**Symptom:** Nodes configured with retryOnFail: true, maxTries: 3, waitBetweenTries: 3000 via the SDK create_workflow_from_code are created with retryOnFail: true but without the explicit timing fields. maxTries and waitBetweenTries are absent from the created node, leaving n8n to apply its defaults (3 tries / 1000ms wait). The create call does not error. The discrepancy is only visible on a post-create re-pull.
**Cause:** The SDK create path does not carry retry timing fields through to the created node. Only the retryOnFail boolean survives; the accompanying timing values are silently dropped.
**Platform:** n8n
**Node types / context:** Any node with custom retry configuration created via the SDK code path (create_workflow_from_code or equivalent).
**Fix:** After SDK-based workflow creation, re-pull the created workflow and verify retry fields on all nodes that require specific timings. Restore maxTries and waitBetweenTries via n8n_update_partial_workflow (updateNode) for each affected node.
**Spec rule:** Where a spec mandates specific retry timings (#020 family), the build checklist must include a post-create verification and patch step for retry fields. The SDK create call alone does not guarantee custom retry timings are persisted.
**First seen:** May 2026, n8n (timeout-sweeper build — retry timings dropped on initial create)
**Related:** #020
**Last updated:** May 2026

## History

(none — new entry)

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

## Category-level patterns

- **Deactivate/reactivate is a deployment step, not a recovery step** (#001). It belongs in the checklist.
- **Defaults are wrong for production LLM workflows.** Timeout (#016, #051) and retry-on-fail (#020, #050) both default to values appropriate for trivial workflows and wrong for LLM chains. Both must be explicitly set in the spec — and verified post-create, since the SDK drops retry timing fields silently (#050).
- **updateNode is a full replace, patchNodeField is a partial** (#031). Mixing these up causes silent parameter wipes. Default to patchNodeField.
- **Model swaps have two parts: the body and the label** (#023). Doing only one creates a silent lie in the topology.
- **The SDK validator is not the instance validator** (#049). conditions.options.version: 2 is required by the instance but not enforced by the SDK. Always run a save-and-verify step after SDK-based creates, not just a validate step.
- **Plan tier caps are silent** (#051). A workflow setting that appears correct may be overridden at the instance level with no warning. Verify the instance cap during spec design, not after the first timeout.
- **Use native nodes first; build HTTP calls only when native fails** (#048). The n8n node for instance-internal reads exists and should be preferred. Document the SSL fallback pattern as a deployment prerequisite.
