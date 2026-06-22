# 04 — Cloud Function & GCS

_Binary payload sizing and signed URL patterns, GCS IAM self-binding, Cloud Function authentication, Python null/absent field handling._

**Entry count:** 6
**Last updated:** June 2026
**Related categories:** 03-node-config-and-infrastructure

---

## Entries

## #008 — Cloud Function inline binary response OOMs the n8n worker

**Symptom:** Workflow throws `WorkflowCrashedError` or worker OOMs after a Cloud Function returns a large binary payload (base64-encoded file). Trace shows crash on parse of the HTTP response.
**Cause:** Inline base64-encoded payloads >~5MB cause memory pressure on the n8n worker when parsed into a JSON object. n8n Cloud Starter is particularly susceptible.
**Platform:** n8n + Cloud Function
**Node types / context:** HTTP Request node consuming a Cloud Function that returns binary data inline
**Fix:** Refactor the Cloud Function to upload the binary to GCS and return a v4 signed URL (time-limited) instead of inline bytes. n8n downloads the file via a separate HTTP Request node with `responseFormat: file` if needed, or passes the URL through to the email/delivery step. The Cloud Function's runtime service account needs `iam.serviceAccountTokenCreator` on itself for v4 self-signing.
**Spec rule:** Any Cloud Function or external service that produces binary output must be specced to return a signed URL, not inline bytes, when output exceeds ~5MB. The spec must include the SA permission (`serviceAccountTokenCreator`) as a deployment prerequisite.
**First seen:** May 2026, n8n + GCP Cloud Function
**Related:** #014, #015
**Last updated:** May 2026

## History

---

## #010 — Code node downloading large arraybuffer crashes JS task runner

**Symptom:** n8n JS task runner crashes silently or with "task runner exited" when a Code node downloads a non-trivial binary via fetch/axios and calls `prepareBinaryData`.
**Cause:** n8n Cloud Starter's JS task runner has unreliable headroom for binary handling in Code nodes, even when theoretical memory limits suggest it should work.
**Platform:** n8n
**Node types / context:** Code node (JavaScript) downloading or processing binary data
**Fix:** Avoid binary handling in Code nodes entirely. Use the native HTTP Request node with `responseFormat: file` for downloads. Keep binaries out of n8n where possible (use signed URLs, see #008).
**Spec rule:** Specs must not place binary download or processing logic inside Code nodes. Use the HTTP Request node's native binary handling, or offload binary work to an external service (Cloud Function, etc.).
**First seen:** May 2026, n8n
**Related:** #008
**Last updated:** May 2026

## History

---

## #014 — GCS v4 signed URL self-signing fails with cryptic 403

**Symptom:** Cloud Function generates a v4 signed URL successfully, but the URL returns 403 when accessed.
**Cause:** v4 signed URL generation requires the runtime service account to have `iam.serviceAccountTokenCreator` on itself. Without it, signing silently produces a URL that fails on use.
**Platform:** Cloud Function, GCS
**Node types / context:** Any Cloud Function generating v4 signed URLs for GCS objects
**Fix:** Grant the runtime SA `roles/iam.serviceAccountTokenCreator` on itself: `gcloud iam service-accounts add-iam-policy-binding {SA_EMAIL} --member="serviceAccount:{SA_EMAIL}" --role="roles/iam.serviceAccountTokenCreator"`.
**Spec rule:** Any spec involving GCS v4 signed URLs must list `iam.serviceAccountTokenCreator` (self-binding) as a deployment prerequisite, separate from the bucket-level permissions.
**First seen:** May 2026, Cloud Function + GCS
**Related:** #008, #015
**Last updated:** May 2026

## History

---

## #015 — Cloud Function deployed without auth is open to the internet

**Symptom:** Cloud Function endpoint accessible without credentials. No authentication errors on unauthenticated requests.
**Cause:** Default Cloud Function deployment with `--allow-unauthenticated` is open. Header-based or IAM-based auth must be added explicitly.
**Platform:** Cloud Function
**Node types / context:** Any Cloud Function consumed by a workflow
**Fix:** Implement header-based check (e.g. `X-API-Key` against an env var) or IAM-based authentication. Redeploy with the auth check in the function code. Update consumers (n8n HTTP Request node) to send the credential.
**Spec rule:** Cloud Function specs must declare the authentication mechanism explicitly, not default to `--allow-unauthenticated`. Auth deployment is a prerequisite, not an optional hardening step.
**First seen:** May 2026, Cloud Function
**Related:** #008
**Last updated:** May 2026

## History

---

## #021 — Python dict.get with default doesn't catch explicit-null values

**Symptom:** Cloud Function (or any Python service) returns `'NoneType' object has no attribute 'get'` (or similar attribute error) on a field that the spec describes as "optional with a default." The caller sends `{"field": null}` rather than omitting the key.
**Cause:** `dict.get(key, default)` returns the default only when the key is absent, not when the value is explicit `None`. Common with JSON payloads from JS (where `null` is the natural empty value) hitting Python services.
**Platform:** Cloud Function (Python), any HTTP service consuming JSON from JS
**Node types / context:** Any Python service deserialising JSON with optional fields.
**Fix:** Use `data.get("field") or {}` (or `or []`, `or 0` per type) instead of `data.get("field", {})`. Covers both absent and explicit-null. For strict null-vs-absent distinction, check explicitly: `value = data.get("field"); if value is None: value = default`.
**Spec rule:** Any Python service receiving JSON from a JS source must use the `or default` pattern, not the `dict.get(key, default)` pattern, for fields that may arrive as explicit null.
**First seen:** May 2026, Cloud Function (Python) consuming n8n JSON
**Related:** none
**Last updated:** May 2026

## History

---

## #054 — Google APIs are enabled per-API per-project; one being enabled does not imply another

**Symptom:** n8n Google nodes (or any caller using the project's service account) 403 with "API has not been used in project NNN before or it is disabled," while other Google nodes against the same project and SA work fine. Example: every Google Docs read 403s although Google Sheets reads succeed.
**Cause:** GCP enables APIs individually per project. Enabling the Sheets API does not enable the Docs API (or Drive, etc.). The service account's IAM grants are irrelevant until the specific API is enabled on the project.
**Platform:** GCP (project API enablement), surfaced via n8n Google nodes
**Node types / context:** Any n8n Google node (Docs, Sheets, Drive, …) or Cloud Function call against a GCP project, where a new Google service is introduced to an existing project.
**Fix:** Enable each required API explicitly: `gcloud services enable docs.googleapis.com --project <project>` (and `sheets.googleapis.com`, `drive.googleapis.com`, … as needed). Propagation is ~1–2 minutes. Enumerate every Google API the build touches and enable them all up front.
**Spec rule:** Specs must list every GCP API the build depends on as an explicit enablement prerequisite, per project — not assume "the project already uses Google APIs" covers a newly-introduced one. Add the `gcloud services enable` commands to the deployment checklist.
**First seen:** June 2026, n8n + GCP (proposal-drafting workflow — content-doc reads)
**Related:** #015
**Last updated:** June 2026

## History

(none — new entry)

---

## Category-level patterns

- **Binaries never inline, always signed URLs** (#008, #010). Two entries from different angles converge on the same rule: keep binaries out of the n8n payload path entirely.
- **GCS signed URLs have a silent IAM dependency** (#014). The self-binding permission is not obvious from the error and must be in the spec as a deployment prerequisite.
- **JS null ≠ Python absent** (#021). A persistent JS↔Python boundary hazard. `dict.get(key, default)` is wrong; `or default` is right.
- **GCP grants are necessary but not sufficient — the API must also be enabled** (#054). IAM access to a Google service does nothing until that specific API is enabled on the project. Enable every API the build touches up front; one enabled API never implies another.
