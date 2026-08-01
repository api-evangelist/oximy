---
name: Instrument LLM telemetry with Oximy
description: Initialize an Oximy project and stream normalized LLM interaction events for observability.
api: openapi/oximy-public-api-openapi.yml
operations: [initProject, ingestEvent]
---

# Instrument LLM telemetry with Oximy

Use this to capture telemetry for every LLM call an application makes, without blocking the app.

## Auth
Send both headers on every request (see `authentication/oximy-authentication.yml`):
- `Authorization: Bearer <apiKey>`  (key has an `ox_` prefix)
- `X-Project-Id: <projectId>`  (`proj_` prefix)

## Steps
1. **Initialize** — call `initProject` (`GET /v1/init`). Read `settings.telemetryEnabled`, `settings.policyEnabled`, `settings.policyMode`, and `configVersion`. If it fails, fail open (assume telemetry enabled).
2. **Wrap the LLM client** — the SDK intercepts `chat.completions.create`, `embeddings.create`, `responses.create`, `messages.create`, etc. If integrating directly, build one `OximyEvent` per call.
3. **Emit the event** — call `ingestEvent` (`POST /v1/events`) with a normalized `OximyEvent`: a client-generated `id` (`evt_` prefix), `timestamp`, `source` (sdkVersion/provider/eventType), `raw` request+response, `timing`, and `outcome`. Attach `context` (userId/sessionId/traceId/tags) for correlation. Fire-and-forget with a short timeout (~100ms); drop silently on error.
4. **Watch configVersion** — if the `ingestEvent` response `configVersion` differs from the last `initProject` value, re-call `initProject` to refresh settings/policy.

## Rules
- Never let telemetry block or slow the host request — always fail open.
- Do not put secrets in `raw`; let policy redaction handle sensitive content.
- See `errors/oximy-problem-types.yml` for the `OximyError` codes and `conventions/oximy-conventions.yml` for the fail-open contract.
