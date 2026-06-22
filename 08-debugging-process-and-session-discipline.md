# 08 — Debugging Process & Session Discipline

_Hand-edit corruption of long string payloads, multi-consumer field misses during shape changes, n8n version-log pre/post state semantics, transient-platform-issue discipline, MCP change attribution, stopping before deploying unverified fixes._

**Entry count:** 8
**Last updated:** May 2026
**Related categories:** 02-expressions-and-encoding, 01-data-flow-and-wiring

---

## Entries

## #032 — Hand-edited tool-call JSON corrupts long string payloads via copy-paste duplication

**Symptom:** A long string payload (e.g. an LLM prompt body, a Code node's `jsCode`, a JSON body template) is built correctly in a script or workspace file, but the deployed value is incorrect — typically with a duplicated paragraph or block — only on one of N nodes touched by the same multi-operation deploy. Source-of-truth file is clean; deployed state is corrupt.
**Cause:** When constructing the inline argument to a multi-operation tool call by hand-copying or hand-editing JSON containing heavily-escaped strings, it is easy to accidentally duplicate a block during selection or paste. The duplication does not break JSON validity, so the tool call succeeds. Verification by length or by structural marker count is the only way to catch it.
**Platform:** any (most acute on n8n MCP `n8n_update_partial_workflow`, but applies to any tool call accepting long escaped-string payloads)
**Node types / context:** Long-string field updates (Code node `jsCode`, HTTP Request node `jsonBody`, prompt bodies in any LLM-call node).
**Fix:** Build the operations payload end-to-end in a script. Write the payload to a file. Re-load and verify it before sending — count occurrences of unique markers (e.g. "Return a single valid JSON object" should appear exactly once per node), check expected length per node. After deploy, re-pull the workflow and re-run the same marker checks against deployed state.
**Spec rule:** Tool-call automation scripts that update long string fields must include a pre-send verification step (marker count, expected length) and a post-deploy re-pull verification step. "Wrote it correctly in source" is not the same as "deployed it correctly."
**First seen:** May 2026, n8n MCP
**Related:** #031
**Last updated:** May 2026

## History

---

## #033 — Multi-consumer fields silently miss updates when only the planned consumer is documented

**Symptom:** A node's output shape is changed. The handover lists which downstream node consumes it, the change is implemented, the consumer is updated. A second, unmentioned consumer further downstream breaks at the next execution.
**Cause:** Cross-cutting fields (a rubric, a config object, a shared lookup) often have more than one reader, but handover notes typically name only the primary consumer.
**Platform:** any (n8n with named-node references is particularly prone)
**Node types / context:** Any field referenced by multiple downstream nodes via named-node references.
**Fix:** Before changing a node's output shape, do a workflow-wide search for references to the source node. In n8n, grep the workflow JSON for `$('<Source Node Name>')` — every match is a consumer. Update each one in the same change set.
**Spec rule:** Specs must enumerate every consumer of every cross-node field, not just the canonical or primary one. When a shape change is planned, the spec rev must list every consumer to update.
**First seen:** May 2026, n8n
**Related:** #019, #025
**Last updated:** May 2026

## History

---

## #039 — Stopping a debug session before deploying an unverified fix is the correct move when the user is unavailable

**Symptom:** A debug session reaches a point where (a) the previous session's diagnosis has been disconfirmed, (b) a fix or probe is the obvious next action, but (c) the verification step requires a live execution that the user must trigger or supervise. Pressure (often from "do as much as you can without me") creates a temptation to ship the fix unverified, which entrenches whichever interpretation the fix encodes — even when that interpretation is wrong.
**Cause:** Verification is the part of debugging that distinguishes "applied a change" from "applied a fix." Pushing past the natural stop point feels like productivity but compounds risk. An unverified fix forces the next session to (a) recognise it's wrong, (b) un-apply it, (c) re-derive the diagnostic state.
**Platform:** any debugging session, any platform.
**Node types / context:** Any change that requires a live execution (or live external interaction) to verify, where the user is unavailable.
**Fix:** When the verification step is gated on the user, stop. Write the three artefacts (handover, change log, Fix Bank entries) with full diagnostic state captured. Surface the candidate fix in the handover under "next-session pickup" with the validation criterion attached. Note explicitly which hypotheses have been disconfirmed so the next session does not re-run them.

The user's "do as much as you can without me" instruction is satisfied by completing all the work that does not require them. Shipping an unverified change is *not* "as much as you can" — it is more than that.
**Spec rule:** Sessions running unsupervised must stop at the first action whose verification requires the user. The handover must: identify the candidate fix or probe explicitly; state the verification criterion; list which hypotheses have been disconfirmed and the evidence; recommend the cheapest probe that distinguishes the remaining hypotheses.
**First seen:** May 2026, n8n debug session 12 (Node 16a invalid-syntax diagnosis)
**Related:** #034, #042
**Last updated:** May 2026

## History

---

## #040 — n8n version-log snapshots are pre-operation state, not post-operation

**Symptom:** Reading a workflow version's `workflowSnapshot` field and treating it as the result of that version's operations leads to contradictions when the operations are non-trivial. For example, a version recording a `replaceAll` patch on a string field, where the snapshot still shows the pre-patch content.
**Cause:** n8n's version log is structured around operations as the unit of change. Each version record contains `operations` (what was done) and `workflowSnapshot` (the workflow as it was BEFORE the operations applied). To compute the post-state of a version, you must apply the operations to the snapshot yourself.
**Platform:** n8n MCP (`n8n_workflow_versions get` mode)
**Node types / context:** Any debugging session that uses the version log to investigate "what state was the workflow in when failure X occurred."
**Fix:** When reading a version's snapshot, treat it as the *input* to the operations, not the *output*. To get the post-operation state, apply the operations to the snapshot in code, or read the next version's snapshot (which IS the post-operation state of the previous version, by induction). For the most recent version, use `n8n_get_workflow` to read live state.

To verify operations actually applied: compare the snapshot of the version *after* the version of interest with the snapshot of the version of interest. The difference is what the version's operations did.
**Spec rule:** When investigating workflow history via the version log, build the state model explicitly: for each version, snapshot = pre-state, operations = transition, next-version's snapshot = post-state. Do not infer that "version N's snapshot reflects what version N's operations produced" — it does not.
**First seen:** May 2026, n8n debug session 13 (investigation of v3278 unescape patch attribution)
**Related:** #038, #041
**Last updated:** May 2026

## History

---

## #041 — Apparent "transient platform issue" outcomes often reflect undocumented changes by another actor

**Symptom:** A debug session reproduces a failure at time T1, applies a probe at time T2, observes success, and concludes "the platform must have been transiently broken between T1 and T2." This conclusion masks a different cause: an unrecorded change to the workflow between T1 and T2 that the current session is unaware of. The probe's success is not informative about the original failure — it tested a different state.
**Cause:** When multiple actors have MCP write access to a workflow, changes can land in the version log without any single session's records explaining them. The "transient platform issue" hypothesis is particularly seductive because it is unfalsifiable from outside, explains the observed pattern, and requires no further diagnostic work. It should be the conclusion of last resort, not the first.
**Platform:** any platform with multi-actor write access and a version log that is not surfaced by default in debugging tools.
**Node types / context:** Any debugging session investigating a failure that was reproduced at one time and is no longer reproducing.
**Fix:** Before concluding "transient platform issue," check the platform's version log for any changes between the failure timestamp and the current read timestamp. If changes exist: identify each change; determine whether it could have affected the failure mode; cross-reference against the current session's own change log and prior sessions'. If a change is not attributable to a known actor, note it explicitly as unattributed and do not assume the platform was at fault.
**Spec rule:** When a probe succeeds against a state that previously failed, the diagnostic question is not "was the platform broken?" but "what is different between the failure state and the probe state?" The version log is the first place to look. Any debugging narrative that concludes "transient platform issue" must explicitly document that the version log was checked and found clean.
**First seen:** May 2026, n8n debug session 13 (probe-success conclusion was based on testing a state that had been silently corrected via v3278)
**Related:** #040, #042
**Last updated:** May 2026

## History

---

## #042 — MCP-driven changes must be recorded in the session's change log to maintain attribution

**Symptom:** A future debugging session reads the platform's version log, finds an MCP-driven change with no recorded attribution, and cannot determine which actor applied it. Diagnostic state degrades because the session must investigate a change with no context for why it was made.
**Cause:** The platform's version log records what changed but not why. Session-specific change logs record both — but only if the session writes them. This failure mode is particularly common when: a session applies a fix mid-investigation without separating "probe" from "deploy"; a session's context limit terminates work before artefacts are written; a session's narrative mentions "I'll try this" without a corresponding change log entry.
**Platform:** any platform where multiple actors can write changes via API/MCP.
**Node types / context:** Any change applied via `n8n_update_partial_workflow` (or equivalent) that modifies live workflow state.
**Fix:** Every MCP-driven change must produce a corresponding change log entry in the session's records, written *before* the next operation begins. The entry must capture: timestamp; platform-side identifier (e.g. n8n version id); the exact operation (find/replace strings, target node, target field); the reason for the change; the verification status.

If a session is at risk of running out of context before completing artefacts, the change log must be written *as the change is applied*, not at session end.
**Spec rule:** A session's change log is the canonical record of what that session did, parallel to the platform's version log. Every entry in the platform version log attributable to a session must have a matching session change log entry. If a session ends without all its operations recorded, the next session must treat the gap as a flagged investigation item.

A weaker form of this rule: any debugging session that begins should perform an "attribution audit" early — read the platform version log for the period since the last known clean handover, and cross-reference against all session change logs available.
**First seen:** May 2026, n8n debug session 13 (v3278 unescape patch applied during S12's window with no S12 change log entry)
**Related:** #034, #039, #040, #041
**Last updated:** May 2026

## History

---

## #033-cross — Multi-consumer fields (cross-reference)

_See also entry #033 in category 01-data-flow-and-wiring. Filed here because multi-consumer misses are equally a debugging-discipline failure (not enumerating all consumers before a shape change) as a wiring failure._

---

## Category-level patterns

- **The version log is pre-state, not post-state** (#040). This is the single most common misread in n8n version-log inspection. The snapshot shows what the operations were applied to, not what they produced.
- **"Transient platform issue" is a hypothesis of last resort, not first resort** (#041). Check the version log for inter-session changes before concluding. The hypothesis is particularly dangerous because it is unfalsifiable and terminates diagnostic work prematurely.
- **Attribution is a first-class concern, not bookkeeping** (#042). Unattributed changes in the version log are active liabilities for future sessions. Write the change log entry before the next operation, not at session end.
- **Unverified changes cost more than they save** (#039). Shipping a fix the user can't verify doesn't accelerate progress — it entrenches a guess and creates cleanup work for the next session.
- **Long-string deploys need marker-count verification, not just visual inspection** (#032). Copy-paste duplication is silent and JSON-valid. Verify deployed state against expected structure, not just source.
