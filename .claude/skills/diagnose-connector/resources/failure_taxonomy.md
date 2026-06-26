# Connector Failure Taxonomy

The encoded, hard-won knowledge base. Each entry: **signature** (how to spot it) →
**root cause** → **confirm in source** → **fix**. Match a run's error signal to a
signature, confirm against the connector source, then emit the diagnosis.

Classes are ordered by how often they bite and how cheap they are to confirm.

---

## A. URL / base-URL construction

**Signature:** `NameResolutionError` / `Failed to resolve 'https'`; `host='https'`
in an `HTTPSConnectionPool`; a request URL with a **doubled domain**
(`foo.myshopify.com.myshopify.com`) or a scheme embedded mid-URL
(`https://https://…`).

**Root cause:** A connection parameter that must be a **bare host/subdomain** was
given a **full URL** (scheme + domain). The connector's `base_url` template
prepends `https://` and/or appends a domain suffix, so the extras stack up.

**Confirm in source:** find the `base_url`/`self.base_url` assignment, e.g.
`f"https://{shop}.myshopify.com/admin/api/{api_version}"`. Substitute the bad
value and check it reproduces the malformed URL in the error.

**Fix:** set the param to the bare value (`ALTER CONNECTION … OPTIONS (shop 'sub')`).
Optional hardening (in `sources/{source}/`): normalize in `__init__` — strip a
leading `https?://` and a trailing domain suffix so the footgun can't recur.

**NEXT:** once DNS resolves, the next call actually authenticates — expect a
401/403 to surface if creds/scopes are also wrong (class B/C).

---

## B. Auth — 401 invalid/expired token

**Signature:** `401 {"errors":"... Invalid API key or access token ..."}` /
`401 Unauthorized`. URL resolves fine (no DNS error).

**Root cause:** token expired, revoked, regenerated (app reinstalled), or issued
for a different account/shop than the one in the connection.

**Confirm in source:** check the auth header build (`X-Shopify-Access-Token`,
`Authorization: Bearer …`, basic-auth PAT) reads the right connection param.

**Fix:** mint a fresh token (often reinstalling the app **regenerates** it — copy
the *new* one), update the connection (`access_token`). Verify cheaply:
`curl -s -o /dev/null -w "%{http_code}" -H "<auth header>" <a small GET endpoint>` → expect `200`.
**Not a connector bug** — don't patch code.

---

## C. Auth — 403 missing scope / insufficient access

**Signature:** `403 ... requires <scope> scope` / `Insufficient access` / 403 on
**some** tables but not others (e.g. `orders` works, `inventory_levels` 403s).

**Root cause:** the app/token authenticates but lacks the scope for that resource.

**Confirm in source:** map the failing table/handler to the scope it needs
(connector README / `connector_spec.yaml` usually lists required scopes).

**Fix:** grant the missing scope on the app, **reinstall** (this regenerates the
token — use the new one), update the connection. **Not a connector bug.**

---

## D. Secret substitution / wrong connection param

**Signature:** a credential arrives as the literal string `{{secrets/...}}`, or is
empty/`None` where a value is expected; `KeyError` on a connection option.

**Root cause:** `{{secrets/...}}` substitution isn't applied in this runtime path,
or the spec param name doesn't match what the code reads.

**Fix:** pass `secret_scope` + key as plain params and read via
`dbutils.secrets.get(...)` at runtime; or align the `connector_spec.yaml` param
name with the code. Confirm the param exists in `external_options_allowlist` if
it's table-scoped.

---

## E. Rate limiting — 429

**Signature:** `429 Too Many Requests`, `Retry-After` header, sporadic slow runs,
backoff log lines.

**Root cause:** source's leaky-bucket / cost-based limit exceeded; often naive
parallelism against a **per-shop/per-tenant** limit (more workers ≠ more throughput).

**Confirm in source:** check `request_with_retry` honors `Retry-After` and does
exponential backoff; check for unthrottled fan-out.

**Fix:** honor `Retry-After` + capped exponential backoff; reduce run frequency /
window size; prefer bulk endpoints over per-record calls. Parallelism is not the lever.

---

## F. Pagination

**Signature:** only the first page lands; missing records; or an infinite loop /
hang in `paginate_get`.

**Root cause:** wrong pagination style (Link header vs cursor vs page/offset),
wrong `records_key`, or a next-cursor that never terminates.

**Confirm in source:** the pagination helper + (simulate mode) `endpoints.yaml`
`response.pagination_style` and `wrapper.records_key`.

**Fix:** correct the style/records_key; ensure the terminating condition (empty
page or absent next-link) is handled.

---

## G. Provider redacts fields (Shopify PCDA and friends)

**Signature:** rows arrive but specific fields are uniformly `null` (e.g.
`customers.email/first_name/phone`).

**Root cause:** a source-side **data-protection policy** redacts fields unless the
app is granted access (Shopify Protected Customer Data Access). The records exist;
the API filters at response time.

**Fix:** configure the access grant on the app (PCDA). **Not a connector bug** —
no code change, no token reissue needed after granting.

---

## H. Metadata lag / wrong source of truth

**Signature:** a freshness/count check reads 0 (or stale) right after data lands,
though the data is present.

**Root cause:** table **metadata** (e.g. BigQuery `num_rows`) lags the streaming
buffer; the connector trusted metadata instead of querying rows.

**Fix:** use `SELECT COUNT(*)` (or an actual row read) instead of metadata for
"has the data landed?" semantics.

---

## I. Hang / no timeout / over-wide query

**Signature:** no output, no error; logs stop at `Starting new HTTPS connection`;
record runs that never return.

**Root cause:** a `requests`/`session` call without a `timeout`, or a query window
so wide the source never responds.

**Fix:** every HTTP call needs a `timeout`. If a bounded call times out, **narrow
the window/limit** (halve it) rather than raising the timeout. Add server-side
date filtering / a sliding window for large accounts.

---

## J. Schema drift / validation error

**Signature:** schema-validation failure; new/unexpected fields; a column the
connector's pinned schema doesn't know.

**Root cause:** the source added fields in a newer API version; the connector's
schema is pinned to an older `api_version`.

**Fix:** pin `api_version` to a tested version for stability; to adopt new fields,
update the connector's `TABLE_SCHEMAS` and (simulate) the corpus.

---

## K. Framework / packaging gotchas

**Signature & cause (each → fix):**
- **Param coercion rejects `str | None`** (union types) → make params plain `str`
  with an empty-string sentinel.
- **`ModuleNotFoundError` for a runtime package** (e.g. `python_operator_task`) →
  bundle the runtime package inside the wheel; it isn't auto-injected.
- **A task type silently dropped on deploy** → `engine: direct` in the bundle
  (the terraform engine can drop newer task types).
- **`externalOptions` not honored** → the table option isn't in
  `external_options_allowlist` in `connector_spec.yaml`.

---

## L. Broken / masked error reporting

**Signature:** an error surfaces as a literal template string with unexpanded
braces (`API error: {resp.status_code} {resp.text}`); or a real failure shows up
as an unrelated downstream error (e.g. a `TypeError: string indices must be
integers` where a list of dicts was expected); or failures vanish entirely.

**Root cause (any of):**
- A raise/log message that should be an f-string but isn't (missing `f` prefix),
  so the real status/body never appear.
- An `except` block that **swallows** the exception — returns an error sentinel
  (`{"error": …}`, `None`, `[]`) instead of re-raising — while callers assume the
  success-shape, turning the real error into a confusing downstream crash.

**Confirm in source:** read the raise/except around the failing handler. Check
(a) every error message is an f-string, and (b) whether the `except` returns vs.
re-raises, and whether callers can actually handle the returned sentinel.

**Fix:** make the message an f-string (include status + body); **re-raise** with
context (`raise RuntimeError(...) from e`) instead of returning a sentinel, unless
a caller genuinely handles that sentinel. This is a real connector bug — patch
`sources/{source}/`.

**Why it matters:** this class is insidious because it makes *every other class*
harder to diagnose — the real signal (A–K) is hidden behind a useless string or a
misleading downstream stack trace. Fix it first when present; it unblocks
diagnosing everything else.

---

## Using this taxonomy

- More than one class can apply at once — fix the **blocking** one first, predict
  the next in `NEXT:` (the classic cascade is **A → B → C**: URL bug hides an auth
  bug hides a scope bug).
- Always separate **connector bug** (patch `sources/{source}/`) from **config**
  (`ALTER CONNECTION` / admin) from **source-side policy** (grant on the provider).
  Classes B, C, G are config/policy — never patch code for them.
- If nothing matches with confidence, say `low` and name the one fact that would
  disambiguate (the response body, the request URL, the connection options).
