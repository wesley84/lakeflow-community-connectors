---
name: diagnose-connector
description: "Connector Doctor — diagnose a failing Lakeflow community-connector pipeline run from an error/traceback (or a Databricks run_id) plus the connector source, localize the root cause to a specific line, and propose (or apply + verify) a fix. Domain-specialized for the community-connector framework and source-API auth/pagination quirks — not a generic stack-trace reader."
---

# Connector Doctor

## Goal

Take a **failing connector run** and return, fast:

1. **Root cause** — the actual reason, localized to a file:line in the connector source or to a connection/config setting.
2. **Evidence** — the specific signal in the error + the matching line of source that proves it.
3. **Fix** — a concrete code patch (`Edit`) and/or a config action (`ALTER CONNECTION …` SQL), with a one-line "why this works".
4. **Confidence** — `high` (signature + source both confirm) · `medium` (signature matches, source unconfirmed) · `low` (best guess; say what else you'd need).

This is the difference between a 30-minute stack-trace-and-docs spelunk and a 10-second answer. It is **domain-specialized**: it knows the community-connector framework internals (the `sources/{source}/` layout, `connector_spec.yaml`, connection options, the `base_url`/auth-header construction, the simulator/cassette test harness) and the long tail of source-API auth, pagination, and rate-limit quirks. A generic RCA bot sees a traceback; Connector Doctor knows that `host='https'` + a doubled domain means a connection param ate a full URL — and which line built it.

## Inputs

The invocation gives you one or both of:

- **An error / traceback** — pasted text (e.g. a `PYTHON_STREAMING_DATA_SOURCE_RUNTIME_ERROR`, an HTTP status body, a Python exception).
- **A Databricks `run_id`** (optional) — if given and you have CLI/API access, pull the failure with `databricks jobs get-run <run_id>` (or the pipeline events API) to recover the error text. If you can't reach it, ask for the pasted error instead of guessing.

Plus the **connector** under test — infer the source name from the traceback path (`sources/{source}/…`) or ask. If the source isn't obvious, default to inspecting whatever `sources/{source}/` the error names.

## Procedure

### 1. Extract the signal

From the error, pull the discriminating facts — don't skim:
- The **final exception type + message** (the bottom of the chain, not the top).
- Any **HTTP status code + response body** (`401`/`403`/`429`/`404`/`5xx`).
- The **request URL** actually attempted (reveals URL-construction bugs).
- The **failing table / handler** (`_read_products`, `_read_cdc_table`, `paginate_get`, `latestOffset`).
- The **connector file:line** in the traceback.

### 2. Match against the taxonomy

Read `resources/failure_taxonomy.md`. Match the signal to a failure class by its **signature**. The taxonomy is the encoded, hard-won knowledge — each entry has: signature → root cause → where to confirm in source → fix. Pick the best match; more than one can apply (e.g. a URL bug masks an auth bug — fix the first, predict the second).

### 3. Confirm in source

Open the connector and verify the cause against real code before asserting it. Typical reads:
- `sources/{source}/{source}.py` — `base_url` construction, auth-header build, the failing handler, pagination helper.
- `sources/{source}/connector_spec.yaml` — connection params + `external_options_allowlist`.
- The connection's actual option values (via `DESCRIBE CONNECTION <name>` if a run is live).

Localize to a line. "The `base_url` template at `{source}.py:NN` is `f\"https://{host}.x.com/...\"`, so a `host` of `https://foo.x.com` produces `https://https://foo.x.com.x.com/...`" beats "looks like a URL problem."

### 4. Emit the diagnosis

Output this structure (always — it's the demo artifact and it's agent-readable):

```
ROOT CAUSE: <one line, with file:line or the config field>
EVIDENCE:   <the error signal> ↔ <the source line / config value>
FIX:        <patch or ALTER CONNECTION / config action>
WHY:        <one line>
CONFIDENCE: high | medium | low
NEXT:       <if low, what to gather; if a fix masks a downstream issue, name it>
```

### 5. Apply + verify (only if asked, or in `mode=fix`)

- **Config fixes** (connection options, scopes, tokens): emit the exact `ALTER CONNECTION` SQL or admin step; don't fabricate secrets. Offer a cheap pre-check (e.g. a single authed `GET` returning `200`) before re-running the whole pipeline.
- **Code fixes** (`sources/{source}/`): apply a minimal `Edit`, then run simulate-mode tests to confirm green:
  ```bash
  pytest tests/unit/sources/{source}/ -v
  ```
  Respect the repo rule: only touch files under `sources/{source}/`. Never edit library/interface/framework code to paper over a connector bug. If after merging you changed the source, regenerate the merged file: `python tools/scripts/merge_python_source.py {source}`.

## Rules

- **Confirm before asserting.** Read the source line; don't diagnose from the traceback alone unless the signature is unambiguous (e.g. `host='https'`).
- **Distinguish connector bug vs config vs source-side policy.** A 403-missing-scope or Shopify PCDA-null is *not* a connector bug — say so, and give the admin fix. Don't patch code for it.
- **A fix that resolves DNS may reveal auth; an auth fix may reveal scope.** When you fix one layer, predict the next in `NEXT:` so the user isn't surprised (this is exactly the URL → 401 → 403 cascade).
- **No fabricated credentials.** For token/secret fixes, give the rotation/grant steps; never invent a token value.
- **Cheap verification first.** A one-call `curl`/`GET` beats a full pipeline re-run for confirming an auth/URL fix.
