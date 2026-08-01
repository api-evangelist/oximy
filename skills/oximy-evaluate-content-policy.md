---
name: Evaluate content against Oximy policy
description: Fetch the active policy and evaluate LLM input/output against server-side (SLM) rules for PII, prompt injection, and custom classification.
api: openapi/oximy-public-api-openapi.yml
operations: [getPolicy, evaluatePolicy]
---

# Evaluate content against Oximy policy

Use this to enforce AI-usage policy on LLM input or output before it is sent or returned.

## Auth
- `Authorization: Bearer <apiKey>` (`ox_` prefix). See `authentication/oximy-authentication.yml`.

## Steps
1. **Load policy** — call `getPolicy` (`GET /v1/policy`). It returns a `PolicyConfig` with `version`, `mode` (`shadow` | `quarantine` | `enforce`), and `rules[]`. Each rule has a `tier` (`local` | `slm`), a `target.scope` (`input` | `output` | `tool_call` | `tool_result` | `mcp`), a `match`, an `action`, and a `severity`.
2. **Evaluate local rules in-process** — regex, deny/allow lists, contains, token/rate/cost limits run client-side in under ~5ms; do not call the API for these.
3. **Evaluate SLM rules** — for `tier: slm` rules (AI PII detection, prompt-injection detection, custom classification), call `evaluatePolicy` (`POST /v1/evaluate`). Send `rule_id` + `content` + `context` for a single rule, or `projectId` + `requestId` + `content` + `rules[]` for a batch. Read `triggered`, `confidence`, `detections[]`, and any `transformedContent`.
4. **Apply the action** — honor `mode`: `shadow` logs only; `quarantine` logs and alerts but allows; `enforce` blocks (`block`) or rewrites (`redact`/`transform`). On a `block` in enforce mode, raise the equivalent of `OximyPolicyError` with the matched violation.

## Rules
- Respect `policyMode` — never block in `shadow`/`quarantine`.
- Evaluation should fail open (treat as not-triggered) on API error so it never breaks the app; see `conventions/oximy-conventions.yml`.
- Use `detections[].start`/`end` to redact precisely rather than dropping whole messages.
