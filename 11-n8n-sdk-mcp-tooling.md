# 11 — n8n SDK / MCP Build-Path & Tooling

_How the build tooling behaves, as distinct from how a node is configured at runtime (cat 03): native-node vs hand-built reads, SDK-vs-instance validation gaps, heuristic validator false-positives, silent drops on SDK create, conditional-schema credential-create rejections, SDK static-parse limits, and credential binding via SDK/MCP. Split out of category 03 in the June 2026 consolidation._

**Entry count:** 7
**Last updated:** June 2026
**Related categories:** 03-node-config-and-infrastructure, 08-debugging-process-and-session-discipline, 02-expressions-and-encoding

---

## Entries

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
**Related:** #031, #061
**Last updated:** June 2026

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
**Last updated:** June 2026

## History

- June 2026: reconfirmed on the `create_workflow_from_code` path during the content-intake build — `retryOnFail: true` survived but `maxTries`/`waitBetweenTries` were dropped on all 10 retry nodes; restored via partial-update `updateNode` and re-pulled to confirm. No change to the rule.

---

## #061 — n8n-mcp validator raises non-actionable false positives; verify against execution history, the SDK validator, and get_node_types

**Symptom:** The n8n-mcp REST/validator server reports "errors" on nodes that are valid and have run successfully. Recurring false positives include: `"Range is required"` / `"Values are required"` on Google Sheets v4.5 `update`/`append` nodes using `mappingMode: defineBelow` + `matchingColumns` (the validator checks pre-v4 params); `"responseNode mode requires onError"`; `"error output in main[1] missing continueErrorOutput"` on IF/Switch nodes whose `main[1]` is a legitimate false/second branch; "Cannot return primitive values directly" on Code nodes returning `[{ json: {...} }]`; "Invalid $ usage" on valid `$('Node Name')` refs; "File system/process access not available" on nodes using neither; "URL appears to be missing http://" on `={{ }}` expressions that resolve to https; "Node is not reachable from any trigger" on `ai_languageModel` sub-nodes.
**Cause:** The validator's heuristics (especially its `runtime` profile) lag node versions and misread legitimate structures (second outputs, sub-node attachment, expression-resolved URLs). They are advisory, not authoritative.
**Platform:** n8n-mcp (validator / `n8n_validate_workflow` runtime profile)
**Node types / context:** Google Sheets v4.5 write nodes, IF/Switch with a real false branch, Code nodes, webhook/responseNode, `ai_languageModel` sub-nodes.
**Fix:** Treat these as noise. Confirm against (a) the node's successful execution history, (b) the official SDK `validate_workflow` (which passes these clean), and (c) `get_node_types` for the actual current parameter shape. Trust "0 invalid connections" + a full pull of the real node config over the heuristic verdict.
**Spec rule:** Build checklists must not gate on the n8n-mcp validator's heuristic "errors" for v4.5 Sheets writes, second-output branches, or sub-nodes. Cross-check any such error against execution history or the SDK validator before acting. Complements #049 — the SDK validator and the instance/heuristic validators each miss what the other catches; run both and reconcile.
**First seen:** June 2026, n8n (proposal-drafting workflow build — post-patch validation noise)
**Related:** #049
**Last updated:** June 2026

## History

(none — new entry)

---

## #064 — n8n public-API credential create rejects valid data because absent conditional toggles make schema `if`s vacuously true

**Symptom:** Creating a credential via the n8n public API (e.g. `googleApi`) is rejected demanding unrelated fields (`delegatedEmail`, `httpWarning`, `scopes`, `allowedDomains`) even when the core fields (`email`, `privateKey`) are correct and complete.
**Cause:** The credential JSON schema gates `then`-required fields behind `if: {properties: {toggle: {enum: [...]}}}` conditions. When the toggle property is omitted entirely, the `if` passes vacuously, so its `then`-required fields are enforced — producing demands for fields that only matter under a configuration you didn't select.
**Platform:** n8n (public API credential create, e.g. via an MCP credentials tool)
**Node types / context:** Programmatic credential creation for any credential type whose schema has conditional required-field blocks (notably `googleApi`).
**Fix:** Explicitly set the conditional toggles to non-triggering values so their `if`s evaluate false. Working `googleApi` shape for an HTTP-Request-node GCS-write use: `{ email, privateKey, inpersonate: false, httpNode: true, httpWarning: "", scopes: "<scope>", allowedHttpRequestDomains: "all" }` — `inpersonate:false` and `allowedHttpRequestDomains:"all"` make those `if`s false (no `delegatedEmail`/`allowedDomains` needed); `httpNode:true` enables HTTP-Request-node use and pulls in `scopes` + `httpWarning`.
**Spec rule:** Specs that create credentials programmatically must enumerate the conditional toggle fields and set each to an explicit non-triggering value, not rely on omission. "Send only the fields we use" is wrong for schemas with vacuous-`if` conditional blocks.
**First seen:** June 2026, n8n (content-intake workflow — GCS image-upload credential)
**Related:** #066
**Last updated:** June 2026

## History

(none — new entry)

---

## #065 — n8n Workflow SDK code is statically parsed: only SDK calls and operators allowed, no Array/String methods

**Symptom:** `validate_workflow` / `create_workflow_from_code` fails with "Security violation: Method '<name>' is not an allowed SDK method" (e.g. `join`) on code that is otherwise valid JavaScript.
**Cause:** The SDK code path runs in a restricted static parser that permits only recognised SDK calls and operators — not arbitrary runtime methods like `Array.prototype.join` or `String` methods. This applies to the SDK-authoring code itself, not to a Code node's eventual runtime.
**Platform:** n8n Workflow SDK (`validate_workflow`, `create_workflow_from_code`)
**Node types / context:** Building any node's string fields (e.g. a Code node's `jsCode`) inside SDK-authored workflow code.
**Fix:** Assemble multi-line strings with `+` concatenation (`"line\n" + "line\n" + ...`), not `[...].join("\n")`. Note: regex backslashes embedded in double-quoted source lines must be doubled (`\\d`) so the literal survives to the generated code.
**Spec rule:** When a build is specified via the SDK code path, the spec must note that node string content is assembled with operators only (no helper methods) and that embedded regex/escapes are double-escaped. Provide long `jsCode` blocks in concatenation-ready form.
**First seen:** June 2026, n8n (content-intake workflow — SDK build)
**Related:** none
**Last updated:** June 2026

## History

(none — new entry)

---

## #066 — Credential binding via SDK/MCP: setting auth mode does not bind a credential, and predefined/generic keys need updateNode

**Symptom:** After a build/patch that "set up authentication", a node is effectively unauthenticated: a webhook with `authentication: basicAuth` but `credentials: null` accepts anonymous calls; an HTTP Request node with `genericCredentialType`/`httpHeaderAuth` but `credentials: null` sends no auth header and 401s. Separately, the SDK `setNodeCredential` op rejects a valid predefined-credential key ("node type does not accept credential 'googleApi'"), and repointing a Google node from OAuth2 to a service account leaves it using the wrong auth mode.
**Cause:** `authentication` / `genericAuthType` and the `credentials` object are separate fields — writing the former does not write the latter. And `setNodeCredential` validates against a node's *static* credential allow-list, so it rejects dynamic predefined keys (e.g. `googleApi` on httpRequest, whose key is `nodeCredentialType`) and OAuth-vs-SA mode mismatches.
**Platform:** n8n (SDK / MCP partial-update)
**Node types / context:** Webhook (basicAuth), HTTP Request (genericCredentialType / predefinedCredentialType), Google Sheets/Docs nodes switching OAuth2 → service account.
**Fix:** (1) Bind predefined/generic credential keys via the partial-update `updateNode` op with `updates: {"credentials.<key>": {id, name}}` — not `setNodeCredential`. (2) Switching a Google node OAuth2 → service account requires three coordinated changes in one `updateNode`: set `authentication: "serviceAccount"`, null the old OAuth key (`credentials.googleSheetsOAuth2Api: null` etc.), and set `credentials.googleApi = {id, name}`. (3) After any auth wiring, re-pull the node and confirm the `credentials` object is present with the correct id — make this a mandatory post-wire step (a cross-component audit is where null bindings usually surface).
**Spec rule:** Specs that wire authentication must list credential binding as an explicit step distinct from setting the auth mode, specify `updateNode credentials.<key>` for predefined/generic types, enumerate all three changes for an OAuth→SA repoint, and require a post-wire `credentials`-presence verification. "Repoint the credential" is under-specified. See #026 for credential rebinding on a node-type swap.
**First seen:** June 2026, n8n (proposal-drafting + content-intake workflows — auth wiring audit)
**Related:** #026
**Last updated:** June 2026

## History

(none — new entry)

---

## Category-level patterns

- **The SDK validator is not the instance validator is not the heuristic validator** (#049, #061). Each misses what the others catch: the SDK validator skips instance-required filter metadata (#049); the n8n-mcp heuristic validator false-positives on v4.5 Sheets writes, second-output branches, and sub-nodes (#061). Run more than one and reconcile against execution history and `get_node_types`.
- **SDK/MCP write paths silently drop or reject what the UI handles** (#050 retry timings dropped on create; #064 vacuous-`if` credential-create rejection; #065 static-parse method ban). Treat every SDK create/patch as "write, then re-pull and verify," never "write and trust."
- **Credential binding is its own step, separate from auth mode** (#066). Setting `authentication` does not write `credentials`; predefined/generic keys need `updateNode credentials.<key>`, not `setNodeCredential`; verify with a post-wire re-pull. Pairs with the runtime-side rebinding rule #026 (cat 03).
- **Prefer the first-class node over a hand-built API call** (#048). n8n ships a native node for instance-internal reads; reach for HTTP only when the native node's SSL limitation forces it, and document that fallback.
