# 07 — Multi-LLM Pipelines & Review Panels

_Rubric and reviewer prompt as a coordinated deployment unit, half-shipped feature flags at the CF boundary, single-reviewer-decides-everything semantics, LLM-derived numeric gates triggering unrepayable loops, structured-output reliability by model._

**Entry count:** 6
**Last updated:** June 2026
**Related categories:** 06-llm-prompt-and-schema-contracts, 03-node-config-and-infrastructure

---

## Entries

## #030 — Reviewer prompts and rubric are a coordinated deployment unit

**Symptom:** After a rubric update (added/removed/split criteria), reviewer LLMs send `criterion_id` values that the rubric no longer recognises. Aggregator nodes default-classify the unknown IDs (typically conservatively to blocking) and produce noisy `unknown_criteria` arrays.
**Cause:** The reviewer's output schema and the rubric's criterion list are tightly coupled — same IDs, same names. Updating one without the other creates a silent contract mismatch that only surfaces at the aggregator.
**Platform:** any LLM-judging pipeline with an external rubric
**Node types / context:** Multi-reviewer LLM panels (e.g. three OpenRouter HTTP nodes scoring against a shared rubric), feedback aggregator nodes parsing scores against rubric classifications.
**Fix:** Treat rubric updates and reviewer-prompt updates as a single deployment unit. Plan changes so both ship together. If they cannot ship together (e.g. cross-session work), document the mismatch period explicitly in the handover and avoid testing during it.
**Spec rule:** Any change that adds, removes, renumbers, or reclassifies a rubric criterion must list reviewer-prompt updates as a coordinated dependency. The spec must call this out — both the rubric section and the reviewer-prompt section must reference each other's version.
**First seen:** May 2026, n8n + multi-LLM review pipeline
**Related:** none
**Last updated:** May 2026

## History

---

## #044 — Boolean-flagged Cloud Function feature with caller never sending the gating data

**Symptom:** A Cloud Function exposes a behaviour gated behind `is_revision: true` (or any similar feature flag). Calls with the flag set return success but with "did nothing" markers — `patches_applied: 0`, empty result maps, output identical to input. The feature branch is intact; the flag fires it; the branch reads the gating data; the gating data is always null; the branch silently no-ops.
**Cause:** Half-shipped feature. The receiver code was written and deployed. The caller-side data production was never built. Nothing errors because "no patches" is a valid state — but it's never been a state any real call has produced any other value of.
**Platform:** Cloud Function (any), n8n caller (or similar workflow caller)
**Node types / context:** HTTP Request node body construction in n8n; any caller-side payload assembly for a Cloud Function feature with conditional behaviour.
**Fix:** When investigating a feature-flagged Cloud Function branch that appears to no-op, grep the caller-side code for the gating field name. If nothing writes it, the feature was never wired end-to-end. Two options:
  - Build the missing producer (caller-side diff generation).
  - Collapse the feature flag entirely — if the branch was never functional, deleting it is usually cleaner than completing it.
**Spec rule:** Cloud Function specs that include feature-flagged branches must list both (a) the receiver-side branch and (b) the caller-side producer of the gating data as paired deliverables. Shipping the receiver without a producer is a structural error and produces silent no-op behaviour that escapes basic testing. Either deliver both or ship neither.
**First seen:** May 2026, n8n + Cloud Function (proposal-drafting revision API)
**Related:** #019, #024
**Last updated:** May 2026

## History

---

## #045 — Multi-reviewer LLM panel with single-reviewer-decides-everything semantics

**Symptom:** A workflow runs 3 (or N) LLM reviewers in parallel. In practice, every blocking decision across multiple executions comes from the same single reviewer (typically the most capable model). The other reviewers either fail to parse, return empty, or unanimously disagree — none of which affects the outcome. The panel's three-reviewer cost is paid; the three-reviewer value (diversity, consensus-checking) is not realised.
**Cause:** The aggregator's pass/fail logic treats any single reviewer's blocking-class flag (score = 1) as a workflow-failing event. The panel structure is decorative — it looks like consensus, behaves like primary-with-rubber-stamps.
**Platform:** any LLM-judging pipeline
**Node types / context:** Multi-reviewer LLM panels, Code node aggregator that derives `pass` from reviewer scores.
**Fix:** Decide which semantics the panel actually wants:
  - **Consensus mode:** require N-of-M reviewers to agree on a blocking flag. Single-reviewer flags are recorded as informational issues but do not gate. In the aggregator, replace `pass = !blockingFlagged` with `pass = !blockingConsensus` where `blockingConsensus` requires `consensus_type` is `majority` or `unanimous`.
  - **Primary-only mode:** drop the panel, run one reliable reviewer. Same blocking decisions at one third of the cost.
**Spec rule:** Multi-reviewer panel specs must declare upfront whether the panel is consensus-based (and at what threshold — majority or unanimous) or primary-with-validation (and whether the validators have any decision power). The aggregator's pass logic must match the declared mode. "Three reviewers, one of them blocks" is a structural inconsistency between cost and behaviour and must be designed out.
**First seen:** May 2026, n8n + multi-LLM review pipeline
**Related:** #030
**Last updated:** May 2026

## History

---

## #046 — Workflow gates on LLM-derived numeric estimates, looping on issues the LLM cannot fix

**Symptom:** A workflow's pass/fail logic includes a hard threshold on an LLM-derived value (a cost estimate, a confidence score, a count). When the threshold is breached, the workflow triggers a remediation loop. The remediation LLM correctly recognises the issue cannot be fixed at its level and emits a passthrough/note. The workflow loops anyway, burns compute, and ultimately exits at iteration limit with the same issue unresolved.
**Cause:** Two compounding mistakes: (1) treating an LLM estimate as a deterministic input to control flow; (2) routing all gate failures through the same remediation path even when the issue is structurally out of the remediator's reach.
**Platform:** any LLM-in-the-loop pipeline with quantitative gates
**Node types / context:** IF nodes or Code-node gates that branch on LLM-derived numeric values. Remediation loops with no escape valve for unrepairable issues.
**Fix:** Two-step:
  - **Reclassify the gate.** Move LLM-derived numeric thresholds from hard gates to advisory surfacing. The value still flows through to the user in delivery output; it just doesn't auto-trigger the workflow.
  - **Add issue-routing logic** in the loop decision step. If the only blocking issue is one the remediator cannot fix, route directly to human review instead of remediation.
**Spec rule:** Workflow gates must distinguish deterministic inputs (form fields, schema validation results, deterministic calculations) from LLM-derived inputs (estimates, scores, judgments). Deterministic inputs may gate; LLM-derived inputs surface as advisory. Every remediation loop must declare which issue classes it can repair — issues outside that class must route past the remediation, not through it.
**First seen:** May 2026, n8n + LLM review-and-edit pipeline
**Related:** #045, #058
**Last updated:** June 2026

## History

---

## #047 — LLM reviewer with poor structured-output reliability silently drops out of panel

**Symptom:** A multi-reviewer panel runs N models in parallel. Across multiple executions, one reviewer slot consistently produces parse failures — malformed JSON, empty response envelopes, or duplicated/garbled keys — at a 20–40%+ failure rate. The aggregator silently drops the failed reviewer's scores. The cost of calling that model is paid on every execution; its vote counts on none of the ones where it fails.
**Cause:** A cost-optimised model (Flash variants, Lite models, smaller frontier-adjacent options) was selected for a reviewer slot without reliability testing on the actual prompt and schema. Structured-output reliability varies significantly by model even for simple JSON schemas.
**Platform:** any LLM-judging pipeline using OpenRouter or similar multi-model gateway
**Node types / context:** HTTP Request nodes calling reviewer LLMs (OpenRouter /chat/completions), Code node aggregator that parses reviewer responses.
**Fix:** Two steps:
  - **Replace the model.** For reviewer slots, reliability of structured output on the actual prompt+schema takes priority over reasoning quality or cost. Test the candidate model over 10+ runs with the live prompt before committing. A model producing valid JSON 95%+ of the time is worth more than a marginally smarter model producing it 70% of the time.
  - **Consider adding a hard failure alert.** If `parse_failures.length > 0`, surface this in the delivery email or a monitoring channel — silent drops are worse than noisy ones because they mask degradation.
**Spec rule:** When selecting models for a multi-reviewer panel, prioritise reliability of structured output over quality of reasoning. Test JSON-output reliability over 10+ runs on the live prompt and schema before committing to a model for a reviewer slot. A cheap, unreliable reviewer contributes less than no reviewer (it adds cost and noise without adding signal). Reviewer model changes require a reliability test before deploy, not just a qualitative assessment.
**First seen:** May 2026, n8n + multi-LLM review pipeline (Gemini Flash Lite and GLM-5 both replaced after consistent parse failures)
**Related:** #017, #045
**Last updated:** June 2026

## History

- June 2026: reconfirmed across a 9-test regression. Two refinements. (1) **A well-built panel degrades gracefully** — with quorum ≥2 maintained, parse-failures/anomalies tracked, and `pass` still computed, a *single* reviewer dropout is intermittent, not a pipeline regression; do not treat one dropout as a defect. (2) A frequent concrete cause of one reviewer's parse failures is a **missing `max_tokens`** on that reviewer's HTTP node → truncated JSON; set an explicit `max_tokens` (e.g. ~8000) and lower temperature (~0.1) on the unreliable slot before concluding the model itself must be replaced. Recurring duplicate-`criterion_id` output usually traces to ambiguous rubric wording for that criterion (a rubric/sheet edit, not a workflow edit) — see #030.

---

## #058 — An LLM-derived numeric estimate driving a flag is biased when its input data is sparse; don't blind-widen the threshold

**Symptom:** A flag computed from an LLM's bottom-up numeric estimate over-fires when the source material is thin. Example: a fee-variance flag compares a target figure to an LLM-estimated cost; when the brief omits the detail the estimate needs (e.g. per-item day counts), the estimator lowballs (~40% of target) and the flag reads "significant" on genuinely-aligned cases. The same flag is correct when the input states the missing detail, and correctly fires on true outliers.
**Cause:** The estimate's error is correlated with input completeness, not with the thing being flagged. Treating the estimate as if it were uniformly reliable makes the flag a proxy for "was the input detailed" rather than "is the value off." Widening or suppressing the threshold to quiet the false positives also masks the true positives the flag exists to catch.
**Platform:** any LLM-in-the-loop pipeline with a numeric flag/gate derived from an LLM estimate
**Node types / context:** Code-node or IF-node flags branching on an LLM-produced number (cost/effort/confidence estimate). Especially advisory flags whose noise is reference-only but erodes trust.
**Fix:** Fix the estimator, not the threshold. Strengthen the estimation prompt to scale sensibly from available context (e.g. guest count / scope) when specific inputs are absent, and to signal low confidence rather than guess low. Only then recalibrate thresholds, and verify against known true-positive cases so the recalibration doesn't blind the flag. If the flag is internal/advisory, rank its noise accordingly (low priority) but still treat the root cause as estimator bias.
**Spec rule:** When a flag or gate is driven by an LLM-derived number, the spec must state how the estimator behaves when its inputs are incomplete (scale-from-context + low-confidence signal), and require threshold changes to be validated against true-positive cases. "Widen the threshold until it stops firing" is a spec-time anti-pattern — it masks real signal. Complements #046: LLM-derived numbers are advisory; here, even the advisory value must be made robust to sparse input.
**First seen:** June 2026, n8n (proposal-drafting workflow — investment-summary fee-variance flag)
**Related:** #046
**Last updated:** June 2026

## History

(none — new entry)

---

## Category-level patterns

- **Panel semantics must be declared, not assumed** (#045). "Three reviewers" means nothing without specifying whether it's consensus-based or primary-with-validation. The cost structure implies one; the code may implement the other.
- **LLM-derived numbers are advisory, not deterministic** (#046). Hard-gating on LLM estimates invites unrepairable loops. Reclassify as surfaced values and add explicit issue-class routing. And even as an advisory value, an LLM estimate is biased by input completeness (#058) — fix the estimator before touching the threshold; widening it to silence false positives masks the true positives.
- **A panel that degrades gracefully is working, not broken** (#047 history). Quorum-with-tracking means a single reviewer dropout is intermittent noise, not a regression — but a persistently failing slot is usually a missing `max_tokens` (truncation) or an under-reliable model, both fixable before blaming "flakiness."
- **Reliability testing is a deployment prerequisite for reviewer model selection** (#047). Cost-optimised models fail at structured-output at meaningful rates. Test on the actual prompt, not in general.
- **Both sides of a feature flag must ship together** (#044). Receiver without producer = silent no-op. Rubric without matching reviewer prompts = silent noise (#030). These are the same pattern at different scales.
