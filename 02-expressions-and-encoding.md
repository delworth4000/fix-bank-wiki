# 02 — n8n Expressions & Encoding

_JSON stringify patterns, markdown fence stripping, array coercion, fieldPath syntax for MCP operations, escape-level misreads when inspecting workflow state._

**Entry count:** 5
**Last updated:** May 2026
**Related categories:** 01-data-flow-and-wiring, 08-debugging-process-and-session-discipline

---

## Entries

## #003 — JSON body fails with "JSON parameter needs to be valid JSON"

**Symptom:** HTTP Request node throws `JSON parameter needs to be valid JSON` when the body contains text with newlines, quotes, or other special characters interpolated via `{{ $json.field }}`.
**Cause:** n8n expression interpolation inserts the raw string into the JSON body. Newlines and quotes inside the value break the surrounding JSON syntax.
**Platform:** n8n
**Node types / context:** HTTP Request node with JSON body, any expression that injects user-supplied text into a JSON template
**Fix:** Wrap the expression in `JSON.stringify` and slice off the surrounding quotes: `{{ JSON.stringify($json.field).slice(1, -1) }}`. Or build the entire body with `JSON.stringify({...})` in a preceding Code node and reference it as a single expression.
**Spec rule:** Any HTTP Request body that injects free-text fields must use the `JSON.stringify` wrapper pattern, not raw expression interpolation. Spec the body structure in a Code node where it's safer.
**First seen:** Jan 2026, n8n
**Related:** #007
**Last updated:** May 2026

## History

---

## #007 — Arrays render as [object Object] or comma-joined in LLM prompt templates

**Symptom:** LLM receives `[object Object]`, `,,,`, or unexpectedly comma-joined content where structured data should appear in the prompt.
**Cause:** n8n expression `{{ $json.field }}` calls `.toString()` on arrays/objects, which loses structure. Comma-join is the default for arrays of strings, which breaks if a string contains a comma.
**Platform:** n8n
**Node types / context:** Any LLM node with a prompt template (Anthropic, OpenAI, chainLlm, HTTP Request to LLM API)
**Fix:** Wrap object/array references in `JSON.stringify`: `{{ JSON.stringify($json.field) }}`. For arrays of strings where structured serialisation is preferred over comma-join, use `JSON.stringify` rather than relying on default coercion.
**Spec rule:** The spec must declare the type and shape of every field consumed by every prompt template. Prompt templates referencing arrays or objects must use `JSON.stringify` explicitly. Defensive stringify across the board is a smell — fix the spec, not just the symptom.
**First seen:** May 2026, n8n
**Related:** #003, #005
**Last updated:** May 2026

## History

---

## #009 — chainLlm response wrapped in markdown fences breaks JSON parsers

**Symptom:** Downstream Code node throws `Unexpected token` or "missing field" when parsing an LLM response. Response text starts with ` ```json ` and ends with ` ``` `.
**Cause:** LLMs called via n8n's chainLlm node (or via instruction prompts asking for JSON) often return JSON wrapped in markdown code fences. JSON parsers reject the fences.
**Platform:** n8n, any LLM
**Node types / context:** chainLlm node, or any LLM prompt instructed to return JSON
**Fix:** Strip fences before parsing: `text.replace(/```json|```/g, '').trim()`. Apply in the Code node that consumes the LLM output, or in an intermediate Set/Code node.
**Spec rule:** Any spec node that consumes LLM JSON output must include a fence-stripping step before parsing. Do not assume the LLM will obey "return raw JSON only" instructions consistently.
**First seen:** May 2026, n8n
**Related:** #017
**Last updated:** May 2026

## History

---

## #037 — n8n_update_partial_workflow fieldPath uses dotted indices, not bracket notation

**Symptom:** `n8n_update_partial_workflow` with a `patchNodeField` operation returns `Cannot apply patchNodeField to "<path>": property does not exist on node "<name>"` when the path includes an array index in bracket notation (e.g. `parameters.messages.messageValues[0].message`). Switching to dotted index notation (`parameters.messages.messageValues.0.message`) resolves it.
**Cause:** n8n's diff operations parse `fieldPath` as a dot-separated chain of property keys. Bracket notation is not parsed — the literal string `messageValues[0]` is treated as a single property name, which doesn't exist.
**Platform:** n8n (n8n_update_partial_workflow tool)
**Node types / context:** Any operation targeting a field nested inside an array, e.g. `messages.messageValues[N].message`, `headerParameters.parameters[N].value`.
**Fix:** Replace `[N]` with `.N` in the `fieldPath`. Always run `validateOnly: true` first to catch syntax issues without changing live state.
**Spec rule:** Workflow modification scripts that target nested array elements must use dotted-index syntax in the `fieldPath`. Code that constructs paths programmatically should normalise array indices to dots, never brackets. Every `n8n_update_partial_workflow` call should be preceded by a `validateOnly: true` dry run.
**First seen:** May 2026, n8n MCP
**Related:** #031
**Last updated:** May 2026

## History

---

## #038 — n8n_get_workflow returns JSON-encoded field values; raw-text inspection produces phantom escape diagnoses

**Symptom:** When inspecting field values from `n8n_get_workflow` output, the apparent escaping depends on how the response is parsed. Getting this wrong leads to phantom over-escape diagnoses — every embedded `\"` appearing as `\\"`, triggering a wrong hypothesis that produces a fix the validateOnly probe rejects.

**Important caveat (added S13):** A correctly-parsed read tells you what the wire content is *right now*, not what it was at the time of any historical execution. Always check the version log for changes between the failure timestamp and the read timestamp before drawing conclusions.

**Cause:** The n8n MCP returns workflow data as a JSON document. String field values inside that document are JSON-encoded — every `\"` in the wire content becomes `\\"` in the JSON representation. A debugger that pipes the raw output to a file and analyses bytes treats the JSON-encoded form as ground truth.
**Platform:** n8n MCP (any read operation that returns workflow JSON)
**Node types / context:** Any debugging session investigating expression-engine errors on long string fields. Most acute when the field contains an n8n expression with embedded JS string literals with escaped quotes.
**Fix:** To inspect actual wire content, use a JSON-parsed view of the response:
```python
data = json.loads(tool_response_text)
body = data['data']['nodes'][i]['parameters']['jsonBody']
# `body` is wire-level. Inspect characters directly with body[i] / ord(body[i]).
```
To verify ground-truth wire content independent of read transport, use the validateOnly probe technique: attempt a `patchNodeField` with `validateOnly: True` and the candidate find string. Cost is zero state changes per probe. Iterate at most three escape levels (zero, one, two backslashes).
**Spec rule:** When debugging an n8n field value, distinguish between (a) raw response text (JSON-encoded), (b) parsed value (wire-level, current state), and (c) wire content at the time of failure. Backslash counts from (a) are not reliable. For character-level escape count diagnosis, confirm via a `validateOnly: true` patchNodeField probe.
**First seen:** May 2026, n8n MCP (Node 16a expression render failure misdiagnosis in S11; corrected and refined across S12 and S13)
**Related:** #031, #032, #040
**Last updated:** May 2026

## History

(none — new entry; consolidates S11's original proposal, S12's amendment, and S13's refinement)

---

## Category-level patterns

- **Stringify defensively, always.** Free-text fields in HTTP bodies (#003) and arrays/objects in prompt templates (#007) both require explicit `JSON.stringify`. Raw expression interpolation is a recurring source of silent corruption.
- **LLM JSON output requires fence-stripping by default** (#009). Treat "return raw JSON" as aspirational, not guaranteed.
- **Read-transport encoding ≠ wire content** (#038). Inspect parsed values, not raw response text. Use validateOnly probes for ground truth.
