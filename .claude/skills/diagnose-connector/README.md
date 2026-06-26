# Connector Doctor

**An AI agent that diagnoses and fixes failing Lakeflow community connectors — domain-specialized, not a generic stack-trace reader.**

Q2 2026 Hackathon submission · Wesley Dias

---

## The problem

A connector fails in production and you get a 40-line `PYTHON_STREAMING_DATA_SOURCE_RUNTIME_ERROR` traceback. Figuring out *why* means: read the traceback, open the connector source, recall the source API's auth/pagination quirks, check the connection options, cross-reference the framework's gotchas — then guess. For a connector you didn't write, that's a 30-minute spelunk. Every SA deploying a connector and every developer building one hits this wall.

The failures aren't even exotic — they're the **same dozen classes** over and over: a connection param that ate a full URL, a stale token, a missing scope, a rate-limit, a metadata-lag, a swallowed exception. That knowledge lives in people's heads, not in a tool.

## What it is

`/diagnose-connector` — a Claude Code skill that takes a **failing run** (pasted error or a Databricks `run_id`) plus the connector source, and returns in seconds:

```
ROOT CAUSE: <localized to file:line, or the config field>
EVIDENCE:   <the error signal> ↔ <the proving line of source>
FIX:        <code patch, or ALTER CONNECTION / admin action>
WHY:        <one line>
CONFIDENCE: high | medium | low
NEXT:       <the failure this fix will reveal next>
```

It is **domain-specialized**. It knows the community-connector framework internals (`sources/{source}/` layout, `connector_spec.yaml`, connection options, `base_url`/auth-header construction, the simulator/cassette harness) and a taxonomy of source-API failure classes (A–L) distilled from real connectors. A generic RCA bot sees a traceback; Connector Doctor knows `host='https'` + a doubled domain means a connection param ate a full URL — and which line built it.

## How it works

1. **Extract the signal** — final exception, HTTP status + body, the request URL actually attempted, the failing handler, the connector `file:line`.
2. **Match the taxonomy** (`resources/failure_taxonomy.md`) — 12 encoded classes, each with signature → root cause → where to confirm → fix.
3. **Confirm in source** — open the connector and verify against real code before asserting; localize to a line.
4. **Emit the diagnosis** — the structured block above (agent-readable).
5. **Apply + verify** (on request) — minimal `Edit` under `sources/{source}/`, then `pytest` simulate-mode to confirm green. Config/policy issues (tokens, scopes, PCDA) get the admin fix, never a code change.

## Demo (≈3 minutes)

**Act 1 — the cascade it lived.** Paste the three real failures from a Shopify deployment, in order:
- **URL bug** → Doctor: `shop` set to a full URL; `base_url` template (`shopify.py:72-73`) doubles the scheme + domain. Fix: `ALTER CONNECTION … OPTIONS (shop 'lakeflow-test-store')`. **And it predicts the next failure** in `NEXT:`.
- **401** → stale token (`shopify.py:77-79`); reinstall → new token → update connection; verify with one `curl`. *Not a connector bug.*
- **403** → app missing `read_products` scope (`shopify.py:255`); grant + reinstall. *Not a connector bug.*

The point: it reproduces the exact URL→401→403 cascade **and forecasts each step before it happens** — because it understands the framework, not just the stack trace.

**Act 2 — it travels.** Point it at **HubSpot, a connector it has never seen**:
- A **401** re-localizes cleanly to `hubspot.py:16` (`Authorization: Bearer …`) — different connector, different line, same class. Proves it isn't hardcoded to Shopify.
- Then it **finds a real, previously-unknown bug**: `hubspot.py:340` raises a *non*-f-string (`"API error: {resp.status_code} {resp.text}"`) and `:343-344` swallows it into an `{"error": …}` dict — while both callers iterate the result as a list, turning every real API error into an unrelated `TypeError` downstream. Doctor flags it as a new class (`L`), **applies the fix** (f-string + re-raise), and **simulate-mode tests stay green (9 passed)**.

**Act 3 — it learns.** The HubSpot find wasn't pre-encoded — class `L` was added to the taxonomy from what the Doctor surfaced. The knowledge base grows with use.

## Why it's differentiated

- **Domain-specialized, not generic.** Several "debug agent" entries read tracebacks; this one reasons about connector internals + source-API behavior, so it localizes to `file:line` and separates **connector bug** (patch code) from **config** (`ALTER CONNECTION`) from **source-side policy** (grant on the provider).
- **It predicts the cascade.** Fixing the URL reveals a 401 reveals a 403 — Doctor says so up front.
- **Proven on unseen code.** It found and fixed a real latent bug in a connector it had never been shown — verified green.
- **It's a living knowledge base.** The taxonomy codifies tribal connector-debugging knowledge and grows as new classes surface.

## Results

| | |
|---|---|
| Failure classes encoded | 12 (A–L) |
| Real failures diagnosed in demo | 3 (Shopify) + 2 (HubSpot) |
| Connectors it generalized to unseen | HubSpot (✅ localized + found new bug) |
| Real bugs found & fixed via the project | 1 (`hubspot.py` masked error reporting) |
| Tests after auto-fix | 9 passed, simulate mode |

## Files

- `SKILL.md` — the diagnosis procedure + structured-output contract
- `resources/failure_taxonomy.md` — the encoded knowledge base (classes A–L)

## Beyond the hackathon

Every connector developer and every SA deploying a connector hits these failures. Connector Doctor turns a 30-minute traceback-and-docs spelunk into a 10-second localized fix — and the HubSpot bug it caught is a real, PR-worthy repo improvement found *by* the project.
