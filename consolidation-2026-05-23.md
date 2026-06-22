# Consolidation Pass — May 2026

## Scope

Entries #001–#051. This is the overdue pass flagged at #044 and deferred until #051.

---

## Findings

### No merges required

All 51 entries are distinct. The following pairs were examined as potential merge candidates and retained as separate entries:

- **#006 and #013** — complementary, not duplicate. #006 = cause (use Merge node); #013 = diagnosis (itemCount wrong). Both needed.
- **#016 and #051** — sequential, not duplicate. #016 = set explicit timeout; #051 = that timeout may be silently capped at platform level. Cross-referenced.
- **#020 and #050** — same relationship. #020 = configure retry-on-fail; #050 = SDK drops timing fields even when configured. Cross-referenced.
- **#033 in both cat-01 and cat-08** — intentional dual-filing with a cross-reference note in cat-08. Not a duplicate.

### No category reassignments required

All 51 entries are correctly placed.

### No stale content found

All entries are from May 2026. No superseded claims.

### Cross-references added (4)

| Entry | Was | Now |
|-------|-----|-----|
| #019 (cat-06) | #013, #017 | #013, #017, #033 |
| #043 (cat-01) | #011, #033 | #011, #018, #033 |
| #028 (cat-05) | #027, #029 | #027, #029, #038 |
| #049 (cat-03) | none | #031 |

Rationale:
- **#019 ↔ #033**: Both are "field exists in design, fails in practice" — #019 is no writer; #033 is writer exists but secondary consumers missed. Same class.
- **#043 ↔ #018**: Both are about `.first()` resolving to stale/wrong context — #043 inside loops, #018 across HTTP boundaries. Root pattern is "named-node ref in wrong context."
- **#028 ↔ #038**: Both are "what you inspect is not what the system stored" — #028 from execution data, #038 from JSON-encoded wire values. Related epistemic pattern.
- **#049 ↔ #031**: Both are "SDK accepts it, instance rejects it" — #031 for updateNode parameter wipes, #049 for conditions.options.version. Same SDK/instance gap pattern.

### Category-level patterns updated

**Category 03** — patterns section tightened to explicitly name the SDK/instance validation gap as a pattern class covering #031, #049, #050.

### Structural issue: stale files in folder

The following files are superseded versions left over from earlier in today's session. They cannot be deleted via the Drive connector — please remove them manually:

- `index.md` — id: `1Cy6OQx6eAkkpEBvTgqE-D7XuwsiaHptP` (2817 bytes, created 10:01)
- `log.md` — id: `1zSStG1mbbkCr-o5cmiqRNfDfKo_jp9bw` (756 bytes, created 10:01)
- `03-node-config-and-infrastructure.md` — id: `1w-tzlacRzFriZuxNFw8BPd70Gf0ce60q` (9823 bytes, created 10:05)
- `test` — id: `1O4fUuvbrsKVOWZqylwLRZYcuUbajE3lm` (4 bytes)

The current canonical versions are the larger/later files with the same names.

---

## Next pass due: after entry #061 (10 entries from current baseline of #051)
