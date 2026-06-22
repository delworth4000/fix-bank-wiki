# 08 — Debugging Process & Session Discipline

_Hand-edit corruption of long string payloads, multi-consumer field misses during shape changes, n8n version-log pre/post state semantics, transient-platform-issue discipline, MCP change attribution, stopping before deploying unverified fixes._

**Entry count:** 11
**Last updated:** June 2026
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

## #057 — A dedup/idempotency guard silently blocks fix-and-retest of the same fixture

**Symptom:** Re-running the same test after applying a fix "succeeds" in ~1–2 seconds without doing any real work — the run terminates at a duplicate-notify/short-circuit node with an `is_duplicate: true` marker, and none of the build steps execute. The fix appears untested because nothing ran.
**Cause:** The workflow has a duplicate-submission guard (keyed on, e.g., `client + "__" + event` within a time window). Repeated test submissions with identical key fields are correctly classified as duplicates and short-circuited — which is right in production but defeats iterative testing.
**Platform:** any workflow with an idempotency/dedup guard
**Node types / context:** Intake workflows with a duplicate-detection branch and a fast "duplicate notify" terminal path.
**Fix:** When re-testing after a fix, vary the dedup key (e.g. change the event name) or wait out the window. Recognise the signature — a sub-2-second run whose last executed node is the duplicate path and whose output carries `is_duplicate: true` — so a guard-blocked retest is not mistaken for a passing run. Document the dedup key in the test plan.
**Spec rule:** Test plans for workflows with idempotency guards must specify how to obtain a fresh key per test run (vary the key, or a test-mode bypass). A debug session must check the last-executed node and any `is_duplicate` marker before concluding a re-run exercised the fix.
**First seen:** June 2026, n8n (proposal-drafting workflow — duplicate-submission guard)
**Related:** none
**Last updated:** June 2026

## History

(none — new entry)

---

## #059 — Large n8n executions truncate (~1MB) in full mode; pull filtered output-only and persist to disk

**Symptom:** Reading a large execution with `n8n_executions get mode=full` returns truncated, invalid JSON — the tail nodes (email/editor/aggregator) are cut off (~999k chars, JSON invalid at the end). Any analysis over the full dump silently loses the last nodes.
**Cause:** The full-mode execution payload exceeds the ~1MB transport limit and is truncated mid-document. The bulk of the size is usually giant input payloads (content blobs, reviewer inputs) that the analysis does not even need.
**Platform:** n8n MCP (`n8n_executions get`)
**Node types / context:** Any debugging/scoring task over executions of a content-heavy or multi-LLM workflow.
**Fix:** Pull `mode=filtered` restricted to the decision/output node set with `includeInputData: false` — this drops the large input payloads, stays within limit, and stays valid. Persist the filtered dump to a file (keeps it out of context) and run summarisers/scorers against the file, not the inline response.
**Spec rule:** Verification/scoring harnesses for content-heavy workflows must read from a filtered, output-only execution dump persisted to disk, not a full inline pull. Assume full-mode pulls of large executions are truncated and unusable.
**First seen:** June 2026, n8n MCP (proposal-drafting workflow — scoring over large executions)
**Related:** #060
**Last updated:** June 2026

## History

(none — new entry)

---

## #060 — A deterministic verifier/scorer must assert its inputs are present; identical failures across every case fingerprint a harness bug, not a pipeline failure

**Symptom:** A scoring/verification harness reports the same set of failures across *every* test case (e.g. "reviewers=[]", "present 0/N", "n_scores=0" on all 9 tests). A tell-tale self-contradiction appears: one check names the reviewers for a case while another reports zero reviewers for the *same* execution — impossible if both read the same pipeline output.
**Cause:** The harness sources its data from the wrong place — e.g. a summary artefact that omits the inner content/reviews blob — so a lookup returns `{}`/`[]` and every input-derived check false-fails, while checks fed from a different (present) source pass vacuously. The pipeline is fine; the harness's data pathing is wrong.
**Platform:** any deterministic verification/scoring harness over pipeline output
**Node types / context:** Offline scorers/validators reading execution dumps, summaries, and fixtures.
**Fix:** Build each input from where it actually lives (the filtered exec dump per #059, fixtures, reconstructed matrices), not from a convenience summary that may omit fields. Make the harness **assert** each required input is present and error loudly on absence, rather than treating missing data as a failing check. When a presence-check fails identically across all cases, suspect the harness first and cross-check against an artefact the harness does *not* depend on (e.g. a value surfaced in an unrelated check). Validate recomputed figures against an independent observation record.
**Spec rule:** Deterministic verifiers must assert input presence and distinguish "input missing" (harness/plumbing error) from "check failed" (real defect). Uniform failures across all cases are a harness smell, not a verdict. Always provide an independent cross-check the harness's data path cannot also break.
**First seen:** June 2026 (proposal-drafting workflow — Round-2 scoring harness)
**Related:** #059
**Last updated:** June 2026

## History

(none — new entry)

---

## #068 — Spec "verbatim from live" code blocks can carry transcription typos; patch the live node with deltas instead of pasting the block

**Symptom:** A spec marks a Code node "verbatim from live — update named-refs only," but the spec's code block contains a syntax error absent from the live node (e.g. a dropped `const` declaration that merges two statements). Pasting the block verbatim would break the node (here, email composition).
**Cause:** "Verbatim from live" blocks are transcriptions and can lose characters in the copy into the spec. The live node — not the spec transcription — is the source of truth for the portion that is meant to be unchanged.
**Platform:** any (acute with n8n Code/HTTP nodes carrying long code/prompt blocks)
**Node types / context:** Any node a spec marks "carried verbatim from live, update only X," where the spec embeds the full prior code.
**Fix:** For "unchanged except deltas" nodes, patch the *live* node with only the intended changes (the V-next additions + named-ref renames) rather than replacing it with the spec's transcribed block. Diff the spec block against the live node before trusting either; let the live node win for the unchanged remainder. Mirrors the spec-wins rule in reverse: the spec wins on *intent*, the live node wins on the *bytes it said not to change*.
**Spec rule:** When a spec says "carry verbatim from live," it must instruct the builder to apply deltas to the live node, not paste the embedded block — and flag the embedded block as a transcription that may contain typos. Treat long embedded "verbatim" code as illustrative, not authoritative.
**First seen:** June 2026, n8n (proposal-drafting workflow — email-composition Code node)
**Related:** #032
**Last updated:** June 2026

## History

(none — new entry)

---

## #069 — Multiple MCP servers on one instance present divergent views; verify a write via the writing server (or the active graph), not the other's draft view

**Symptom:** A write reports success and is live, but a second MCP server's read of the same workflow still shows the pre-write state (e.g. `credentials: null` after a binding write), nearly triggering an unnecessary re-fix.
**Cause:** When two MCP servers manage one instance — one REST, one SDK with a draft/publish model — they can present different views. A draft/publish server's `get` may show the draft (pre-write) state while the running graph already reflects the write.
**Platform:** n8n (multiple MCP servers on one instance), generalises to any multi-tool write path with caching/draft layers.
**Node types / context:** Any change applied via one MCP server and read back via another.
**Fix:** Confirm a write through the server that made it, reading the *active/running* graph (e.g. `n8n_get_workflow mode=active`), which is authoritative for what executes. Treat the other server's draft view as non-authoritative; do not "re-fix" based on it. Combine with the attribution discipline of #042 — record which server made the change.
**Spec rule:** When more than one MCP/automation server can write a workflow, the verification step must specify reading back via the writing server's active-graph view. A stale read from a different server is not evidence the write failed.
**First seen:** June 2026, n8n (proposal-drafting workflow — two MCP servers, credential write)
**Related:** #040, #042
**Last updated:** June 2026

## History

(none — new entry)

---

## Category-level patterns

- **The version log is pre-state, not post-state** (#040). This is the single most common misread in n8n version-log inspection. The snapshot shows what the operations were applied to, not what they produced.
- **"Transient platform issue" is a hypothesis of last resort, not first resort** (#041). Check the version log for inter-session changes before concluding. The hypothesis is particularly dangerous because it is unfalsifiable and terminates diagnostic work prematurely.
- **Attribution is a first-class concern, not bookkeeping** (#042). Unattributed changes in the version log are active liabilities for future sessions. Write the change log entry before the next operation, not at session end.
- **Unverified changes cost more than they save** (#039). Shipping a fix the user can't verify doesn't accelerate progress — it entrenches a guess and creates cleanup work for the next session.
- **Long-string deploys need marker-count verification, not just visual inspection** (#032). Copy-paste duplication is silent and JSON-valid. Verify deployed state against expected structure, not just source.
- **Confirm a re-run actually ran** (#057). Idempotency guards short-circuit identical test inputs; a sub-2-second "success" at a duplicate path means the fix was never exercised. Check the last-executed node before trusting a green re-run.
- **The verification harness fails before the pipeline does** (#059, #060). Truncated full-mode pulls and wrong-source data plumbing produce uniform false failures. Pull filtered/persisted dumps, assert inputs are present, and cross-check against an artefact the harness doesn't depend on. Identical failures across every case = harness bug.
- **"Verbatim from live" is a transcription, not a source of truth** (#068). Apply deltas to the live node; the embedded block may carry typos the live node doesn't.
- **One instance, many tools, many views** (#069). After a write, read back via the server that made it and its active graph — a stale view from another MCP server is not evidence the write failed. Pairs with attribution (#042).
