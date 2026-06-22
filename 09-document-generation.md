# 09 — Document Generation

_python-pptx slide deletion leaving orphaned parts, template-driven row cap silently truncating at render._

**Entry count:** 2
**Last updated:** June 2026
**Related categories:** 04-cloud-function-and-gcs

---

## Entries

## #012 — python-pptx slide deletion leaves orphaned parts and media

**Symptom:** Generated .pptx is larger than expected, contains unreferenced media, or fails validation in PowerPoint. Removing slides via `sldIdLst` only does not fully delete them.
**Cause:** Removing a slide ID from `sldIdLst` deregisters the slide but leaves the slide part and its relationships in the package. Orphaned media accumulates.
**Platform:** Cloud Function, python-pptx
**Node types / context:** Any python-pptx code that deletes slides
**Fix:** Drop the relationship explicitly: `prs.part.drop_rel(rId)` after removing from `sldIdLst`. The relationship ID is found via the slide's part name.
**Spec rule:** Any spec involving python-pptx slide deletion must specify both the `sldIdLst` removal and the relationship drop. Document this as a known python-pptx quirk.
**First seen:** May 2026, Cloud Function (python-pptx)
**Related:** none
**Last updated:** May 2026

## History

---

## #022 — Template-driven row cap silently truncates at render

**Symptom:** Generated document (.pptx, .xlsx, .docx) has fewer data rows than the input data contained. No error logged in n8n. The renderer silently dropped overflow rows. (Or the inverse: an upstream "capacity" validator throws and blocks a quote whose data actually renders to a *different, uncapped* artifact.)
**Cause:** Downstream template/renderer has a fixed row capacity. Input data has more rows than the template can hold. Renderer iterates over template slots and stops; extra input data is discarded. A validator-side cap that mirrors the template capacity drifts out of scope as the template/data model evolves — it counts a proxy unit, and it fires on inputs that never reach the constrained artifact.
**Platform:** Cloud Function, any template-based document renderer
**Node types / context:** Any node generating a document from a fixed-capacity template.
**Fix:** Prefer **dynamic rendering** — grow the artifact to fit the data (e.g. clone the template's prototype row N times) instead of relying on a fixed-capacity template. If a hard cap is genuinely unavoidable: (a) count the unit the artifact actually constrains, not a proxy for it; (b) enforce it **only on the code path that reaches the constrained artifact**, not on inputs that render to a different/uncapped output; (c) the renderer must **fail loudly** — assert rendered-rows == input-rows and raise on overflow; never silently truncate.
**Spec rule:** When a renderer has a fixed-capacity template, the default is to make the renderer dynamic (grow to fit), not to cap upstream. Only add a cap as a last resort, scoped to the exact constrained path, and never let the renderer truncate without erroring.
**First seen:** May 2026, n8n + Cloud Function (python-pptx)
**Related:** none
**Last updated:** June 2026

## History

- **June 2026** — reversed the original guidance after it mis-scoped a live failure. Original **Fix**: "add an upstream validator that fails fast when input row count exceeds template capacity." Original **Spec rule**: "declare the cap as a validation rule at the data layer." In practice that upstream cap (a) counted a proxy unit that could legitimately exceed the cap, (b) fired on inputs that rendered to a *different, uncapped* artifact, and (c) was the only thing converting an oversized render into a visible failure rather than silent truncation inside the renderer. Resolution was to make the renderer dynamic (clone the prototype row to fit, assert no truncation) and remove the upstream cap.

---

## Category-level patterns

- **Prefer dynamic rendering over a fixed cap** (#022). Growing the artifact to fit the data is the default; a fixed-capacity template should be made dynamic. If a cap is unavoidable, scope it to the exact constrained path and make the renderer fail loudly — never silently truncate. (An upstream validator cap drifts out of scope as the template evolves: it counts a proxy unit and fires on paths that never hit the constrained artifact.)
- **python-pptx slide deletion is a two-step operation** (#012). The library does not enforce consistency between the slide list and the package relationships. Both must be updated.
