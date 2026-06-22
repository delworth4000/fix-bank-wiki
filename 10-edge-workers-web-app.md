# 10 — Edge / Workers Web App (Cloudflare Pages Functions)

_HMAC token signing over the verifier's exact message, Workers-only Web Crypto extensions, auth-gated deploy hand-off, multipart pass-through. Patterns for static SPAs fronted by Cloudflare Pages Functions / Workers that proxy to backend workflows._

**Entry count:** 4
**Last updated:** June 2026
**Related categories:** 01-data-flow-and-wiring, 04-cloud-function-and-gcs

---

## Entries

## #070 — HMAC: sign over the exact bytes the verifier checks (the base64url string), not the raw payload

**Symptom:** Tokens always fail verification even though signing and verifying use the same secret and algorithm. The generator and verifier each look correct in isolation.
**Cause:** The two sides sign/verify *different messages*. The verifier checks the HMAC over the `base64url(payload)` string bytes (`verify('HMAC', key, sig, encode(payloadB64))`), but the generator signed over the raw JSON payload. The signed message is ambiguous from prose alone; only the verifier's code disambiguates it.
**Platform:** Cloudflare Workers / Pages Functions (Web Crypto), generalises to any HMAC token scheme
**Node types / context:** Token-issuing function (`auth.js`) paired with a verbatim verifier helper (`_tokenVerify.js`).
**Fix:** Sign over the encoded string the verifier consumes: `sign('HMAC', key, enc.encode(payloadB64))`, then emit `payloadB64 + '.' + base64url(sig)`. Use base64url **without padding** (the verifier's `atob` after `-_`→`+/` tolerates missing `=`). When a spec gives the verifier verbatim but the generator only in prose, treat the verifier as the contract and derive the generator from it.
**Spec rule:** A token spec must state the exact signed message (raw vs base64url-encoded), encoding (base64url, padding or not), and concatenation format — not just "HMAC-SHA256." Where one side is given verbatim, the spec must mark it as the canonical definition the other side conforms to.
**First seen:** June 2026, Cloudflare Pages Functions (staff web app — access token)
**Related:** none
**Last updated:** June 2026

## History

(none — new entry)

---

## #071 — `crypto.subtle.timingSafeEqual` is a Workers extension, not standard Web Crypto

**Symptom:** Timing-safe comparison code that works in a Cloudflare Pages Function fails (undefined / not a function) if ported to Node or a browser.
**Cause:** `crypto.subtle.timingSafeEqual` exists in the Cloudflare Workers runtime but is not part of standard Web Crypto. It is safe to use *within* a Worker/Pages Function but is non-portable.
**Platform:** Cloudflare Workers / Pages Functions
**Node types / context:** Passphrase/secret comparison in a Pages Function. Both operands here are fixed-length SHA-256 digests (32 bytes), so the call cannot throw on a length mismatch.
**Fix:** Use `crypto.subtle.timingSafeEqual` only in Workers contexts. Do not copy the same line into a Node service or browser bundle — there, use a constant-time compare appropriate to that runtime. Keep the digests fixed-length so the comparison is well-defined.
**Spec rule:** Specs must mark runtime-specific crypto primitives (Workers extensions, Node `crypto`, browser SubtleCrypto) as non-portable and state which runtime each helper targets. "Use Web Crypto" is insufficient when the code relies on a runtime extension.
**First seen:** June 2026, Cloudflare Pages Functions (staff web app — access gate)
**Related:** none
**Last updated:** June 2026

## History

(none — new entry)

---

## #072 — Cloudflare deploy is auth-gated; the Git-integration route is the canonical hand-off — don't treat deploy as automatable

**Symptom:** A build session asked to "build + push + deploy" cannot complete the deploy: `wrangler pages deploy` needs an interactive `wrangler login` (browser OAuth) or a `CLOUDFLARE_API_TOKEN`, and neither is present in a headless session. The connected Cloudflare MCP exposes KV/R2/D1/Workers tools only — no Pages create/deploy.
**Cause:** Cloudflare deploys are authenticated actions that require credentials a headless agent does not hold. "Deploy" is not fully automatable from the build session by default.
**Platform:** Cloudflare Pages / Workers (deployment)
**Node types / context:** Any CI-less build session expected to deploy a Pages site.
**Fix:** Push the repo, then hand off via the canonical model — connect the repo in the Cloudflare dashboard's Git integration (auto-deploys on push) — or, if the CLI path is required, request a scoped `CLOUDFLARE_API_TOKEN`. State explicitly that the deploy did not run, what remains, and which route to take. Do not retry an unauthenticated `wrangler` push silently.
**Spec rule:** Specs that include a Cloudflare deploy must name the deploy model (dashboard Git integration vs CLI token) and list the required credential as a prerequisite the owner provides. A build session delivers the repo + a deploy hand-off; it must not claim "deployed" without confirmed credentials. See #015 for the parallel "deploy is auth-gated, state the mechanism" lesson on Cloud Functions.
**First seen:** June 2026, Cloudflare Pages (staff web app build)
**Related:** none
**Last updated:** June 2026

## History

(none — new entry)

---

## #073 — Multipart pass-through: re-append every field, never reconstruct, and don't set Content-Type manually

**Symptom:** A proxy function forwards a multipart form to a backend webhook, but the backend receives no binary (the file field arrives empty or missing).
**Cause:** Two failure modes. (1) Reconstructing the outbound form loses the original field names and the `File` object's filename, so the binary lands under the wrong property (or none). (2) Setting `Content-Type` manually on the outbound `fetch` overrides the runtime's generated multipart boundary, corrupting the body.
**Platform:** Cloudflare Workers / Pages Functions (`FormData` / `fetch`)
**Node types / context:** A Pages Function proxying a multipart upload to a backend (e.g. an n8n webhook expecting the binary at `file0`).
**Fix:** Iterate the inbound form and re-append each entry verbatim: `for (const [name, value] of inForm.entries()) outForm.append(name, value)` — this preserves field names and filenames, keeping the binary at its expected property (e.g. `file0`, per #002). Do **not** set `Content-Type` on the outbound fetch; let the runtime set the multipart boundary.
**Spec rule:** Specs for multipart-proxying functions must require entry-by-entry re-append (preserving names + filenames) and explicitly forbid setting `Content-Type` on the forwarded request. The binary property name the backend expects must be stated (links to #002).
**First seen:** June 2026, Cloudflare Pages Functions (staff web app — content upload proxy)
**Related:** #002
**Last updated:** June 2026

## History

(none — new entry)

---

## Category-level patterns

- **The verifier is the contract.** When a token scheme gives one side verbatim and the other in prose, sign/verify over the exact same bytes the verbatim side defines (#070). Ambiguity in "what is signed" is the default cause of always-failing tokens.
- **Workers runtime ≠ Node ≠ browser.** Web Crypto extensions like `timingSafeEqual` (#071) are non-portable; mark runtime-specific primitives in the spec and don't copy them across runtimes.
- **Deploy is an authenticated action, not a build step** (#072). A headless session delivers the repo and a deploy hand-off; it cannot claim "deployed" without owner-provided credentials. State the deploy model up front.
- **Forward bytes, don't rebuild them** (#073). Re-append multipart entries verbatim and never set `Content-Type` manually — both break the binary. The backend's expected binary property name is part of the contract (#002).
