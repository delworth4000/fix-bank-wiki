# 09 — Document Generation

_python-pptx slide deletion leaving orphaned parts, template-driven row cap silently truncating at render._

**Entry count:** 2
**Last updated:** May 2026
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

**Symptom:** Generated document (.pptx, .xlsx, .docx) has fewer data rows than the input data contained. No error logged in n8n. The renderer silently dropped overflow rows.
**Cause:** Downstream template/renderer has a fixed row capacity. Input data has more rows than the template can hold. Renderer iterates over template slots and stops; extra input data is discarded.
**Platform:** Cloud Function, any template-based document renderer
**Node types / context:** Any node generating a document from a fixed-capacity template.
**Fix:** Add an upstream validator that fails fast with a clear error when input row count exceeds template capacity. In n8n, place in the validation middleware closest to the data origin. Throw with the actual count and the cap value.
**Spec rule:** Whenever a downstream renderer has a fixed row/column capacity, the spec must declare that cap as a validation rule at the data layer, not discover it at render time.
**First seen:** May 2026, n8n + Cloud Function (python-pptx)
**Related:** none
**Last updated:** May 2026

## History

---

## Category-level patterns

- **Template renderers fail silently on overflow** (#022). Truncation without error is the default behaviour. The cap must be enforced upstream, at the data layer, with a loud failure.
- **python-pptx slide deletion is a two-step operation** (#012). The library does not enforce consistency between the slide list and the package relationships. Both must be updated.
