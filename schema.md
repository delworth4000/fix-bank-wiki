# Fix Bank Wiki — Schema

This folder is the Fix Bank wiki. It is maintained by Claude, not by the user. The user uploads the folder link at the start of a debugging or spec-writing session; Claude reads the index, loads relevant category pages, and uses them.

## Folder structure

```
Fix Bank/
├── schema.md          ← this file; how the wiki works
├── index.md           ← catalogue of all category pages + entry count + last updated
├── log.md             ← append-only chronological record of all session activity
└── categories/
    ├── 01-data-flow-and-wiring.md
    ├── 02-expressions-and-encoding.md
    ├── 03-node-config-and-infrastructure.md
    ├── 04-cloud-function-and-gcs.md
    ├── 05-data-sources-and-config-architecture.md
    ├── 06-llm-prompt-and-schema-contracts.md
    ├── 07-multi-llm-pipelines-and-review-panels.md
    ├── 08-debugging-process-and-session-discipline.md
    └── 09-document-generation.md
```

New categories are created when enough entries justify one — not speculatively.

---

## Category pages — structure

Each category page is a markdown file with:

```markdown
# [Category Name]

_One-line description of what this category covers._

**Entry count:** N  
**Last updated:** [Month Year]  
**Related categories:** [links to other category files]

---

## Entries

[All entries in this category, in original Fix Bank format]

---

## Category-level patterns

[Optional. 3–5 bullet points synthesising the recurring pattern across entries in this category. Written by Claude after enough entries exist to see the pattern. Not present until warranted.]
```

---

## Operations

### Session start

1. Fetch `index.md` to get the category map and entry counts.
2. Identify which categories are relevant to the current session's task.
3. Fetch those category files. Do not fetch all categories unless doing a lint pass.
4. Check `log.md` for recent session activity relevant to the current task.

### Debugging session

- Match symptoms against entry **Symptom** lines across relevant categories.
- If a match is found, surface the entry ID and category, and propose the documented fix. Do not silently apply.
- If a new fix is discovered, propose a new entry at session end using the standard entry template (see below).
- After the user approves, add the entry to the correct category file and update `index.md` and `log.md`.

### Spec-writing session

- Walk every entry's **Spec rule** line in relevant categories and check the spec against it.
- Surface any spec gaps as findings before the build starts.
- Note which categories were skipped and why.

### Adding a new entry

1. Identify the correct category. If no category fits, propose a new one.
2. Add the entry to the category file in standard format (see below).
3. Update the category's **Entry count** and **Last updated** fields.
4. Update `index.md` — entry count for the category, and last updated date.
5. Append to `log.md`.

### Lint pass

Run when: every ~10 new entries, or when the user requests it.

- Look for: duplicate or near-duplicate entries across categories, entries that should be merged, entries in the wrong category, category-level patterns that have emerged and should be written up.
- Propose all changes to the user before applying.
- After applying, update `index.md` and append to `log.md`.

### Adding a new category

- Threshold: 3+ entries that don't fit any existing category, or 1 entry of sufficient standalone importance.
- Add the category file, add it to `index.md`, add a `schema.md` entry under Folder structure.

---

## Entry template

```markdown
## #NNN — [One-line title]

**Symptom:** [What the user / execution trace actually shows. Grep-friendly.]
**Cause:** [Root cause, one or two sentences.]
**Platform:** [n8n / Cloud Function / GCS / OpenRouter / Anthropic API / etc.]
**Node types / context:** [Which node types or contexts this applies to.]
**Fix:** [The actual fix. Concrete enough to apply.]
**Spec rule:** [The design-level lesson. What should the spec say to prevent this?]
**First seen:** [Date and platform context. No client names.]
**Related:** [Other entry IDs, comma-separated, or "none".]
**Last updated:** [Date.]

## History

[Empty for new entries. Old versions appended here when amended, newest first, capped at 3.]

---
```

---

## Maintenance rules

- **Entries must be generic.** No client names, no client-specific node names, no specific credential names or IDs.
- **History is appended, not rewritten.** Old versions move into History with date and reason. Cap at last 3 amendments.
- **Cross-reference related entries.** Use entry IDs. Cross-category references are fine.
- **`log.md` is append-only.** Never edit past entries.

---

_Schema version: 1.0 — May 2026_
