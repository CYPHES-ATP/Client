# glm-5.3-flash thinking-budget exhaustion diagnostic snapshot

Captured for the v0.17.9 reliability cycle, following the format established by
`release/v0.17.7/diagnostics/deepseek-empty-stream-2026-08-04.md`. This document
contains no model prompts, repository source, private keys, node identities,
claim IDs, or ledger rows.

## Summary

`glm-5.3-flash:cloud` answers simple prompts but earns zero credit on audit work
units. On every audit task the model spends its entire output allowance on
reasoning tokens and terminates before emitting an answer. This is the same
signature already recorded for `kimi-k3` (operator tests dated 2026-08-09 and
2026-08-19).

The cap appears to be enforced at the provider endpoint, not by this client. See
"Proxy rewrite test" below, which is the observation that rules out client-side
causes.

## Operator-supplied observation

- Runtime: CYPHES node v0.17.8
- Provider: Ollama (`ollama.com` cloud endpoint)
- Model: `glm-5.3-flash:cloud`
- Work unit types exercised: `repo-inventory`, `dependency-config-review`,
  `scope-mapping`
- Successful submissions across the full test window: 0

Every failed attempt carried the same telemetry signature:

| Field | Observed value |
| --- | --- |
| `thinking_bytes` | ~24,000-28,000 |
| `done_reason` | `length` |
| `eval_count` | 6500 |
| Resulting error | `Ollama returned an empty streamed response` |
| Retry behavior | 3 attempts, then work unit failed |

These counts came from an operator report. They are recorded as supplied
evidence, not as independently reproduced measurements.

## Proxy rewrite test

The operator routed the node through a local proxy that rewrote the outbound
request before forwarding it, raising the output allowance on both request
shapes this client emits:

- native `/api/chat` options path: `num_predict` 6500 -> 20000
- OpenAI `/chat/completions` path: `max_tokens` 6500 -> 20000

Result: unchanged. Every attempt still reported `eval_count = 6500` and
`done_reason = "length"`.

The proxy rewrite was verified working independently — a manual request through
the same proxy returned full content on a simple prompt. The conclusion the
operator draws is that the 6500-token output cap is applied server-side at the
provider endpoint regardless of what the client requests.

Note for reviewers: the client-side constant `MAX_MODEL_OUTPUT_TOKENS` in
`src-tauri/src/audit_runtime.rs` is also 6500 and is sent on both paths. The
proxy test is what separates the two explanations, because it overrode the
client value and the observed ceiling did not move.

## Locally verified v0.17.8 behavior

- `run_ollama_chat_once` assembles `message.content` from the stream and returns
  `Ollama returned an empty streamed response` when nothing was assembled.
- That error is constructed as **retryable**, so `run_ollama_chat` repeats it up
  to `OLLAMA_MAX_ATTEMPTS` (3) with backoff delays of 750 ms and 2000 ms.
- `done_reason` and `thinking_bytes` are already captured and logged on that
  path, so the signature above is visible in existing node logs without new
  instrumentation.
- A `length` stop with empty content is deterministic rather than transient: the
  identical request exhausts the identical cap. The retries cannot recover it,
  and each failed unit holds its claim against the 15-minute
  `WORK_UNIT_CLAIM_TTL_MS` window.

## Implication for model tiers

Thinking models that reason before answering cannot currently complete audit
work units at this endpoint cap, which means their tier multipliers are
unreachable in practice for as long as the cap stands. Among the cloud tiers,
`deepseek-v4-flash` (17.5x) remains viable because its reasoning fits inside the
cap.

This is a provider-side constraint, not a scoring one. Both existing gates
already stand between a declared tier and a paid multiplier: the throughput gate
requires a surviving cloud claim at 25 tok/s or better, and the output-quality
gate caps any contribution without a reportable finding or three evidence-backed
coverage items at 1.0x. A model that emits no content clears neither, so no
multiplier is paid on a failed work unit regardless of its declared tier.

v0.17.9 sets `glm-5.3-flash` to 25x as an exact-family match, so the tier is in
place when the endpoint cap is addressed. Operators choosing a model today
should note the measured result recorded above.

## Requested upstream remedy

Both items are server-side, at the provider:

1. Raise the output cap for thinking models on the `ollama.com` endpoint
   (20K or above would clear the observed 24K-28K reasoning volume), or
2. Expose a separate thinking-budget parameter so reasoning tokens and answer
   tokens are budgeted independently.

Until one of these lands, no client-side change can make these models earn.

## Client-side change included in this PR

Fail fast instead of retrying. When the stream ends with `done_reason = "length"`
and no assembled content, the error is now terminal rather than retryable. This
does not make the model work; it stops the node from spending three times the
tokens and three times the claim window on a failure that cannot recover, and it
surfaces the actual cause in the error text instead of a generic empty-response
message.

## Data requested from operators

For each model, report aggregate request attempts, successful contributions,
terminal failures, empty-content events broken out by `done_reason`, retry
recoveries, latency, and provider-reported token usage. Do not share prompts,
source context, private keys, full database files, or unredacted node logs
publicly.
