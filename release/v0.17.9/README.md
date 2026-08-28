# CYPHES v0.17.9 Thinking-Model Diagnostics and Scoring Update

v0.17.9 is a non-mandatory mainnet release. It preserves the existing
`cyphes-final-testnet-v0.16.0` genesis ledger marker, the `/cyphes/atp/0.15.1`
labor wire, receipt format, and forward-only economics. No database reset is
required and older nodes stay compatible.

This release changes no consensus state, mints no credit, and rewrites no
history. Model economics remain forward-only: the multiplier is signed into each
new runtime receipt, so existing receipts are neither recomputed nor rewritten.

## The problem

Operator testing of `glm-5.3-flash:cloud` on v0.17.8 audit work units returned
zero submissions across the full test window. The model answers simple prompts
normally, but on structured audit prompts it spends its entire output allowance
on reasoning and terminates before emitting an answer.

Every failed attempt carried an identical signature:

| Field | Observed value |
| --- | --- |
| `thinking_bytes` | ~24,000-28,000 |
| `done_reason` | `length` |
| `eval_count` | 6500 |
| Error | `Ollama returned an empty streamed response` |
| Retry behavior | 3 attempts, then work unit failed |

The same signature was previously recorded for `kimi-k3` on 2026-08-09 and
2026-08-19.

Routing the node through a local proxy that raised `num_predict` and
`max_tokens` from 6500 to 20000 on both request paths changed nothing: still
`eval_count = 6500`, still `done_reason = "length"`. The proxy rewrite was
verified working independently on a simple prompt. The cap is enforced at the
provider endpoint, not by this client.

Full telemetry is recorded in
`release/v0.17.9/diagnostics/glm-5.3-flash-thinking-budget-2026-08-28.md`.

## Fixes

- A stream that ends with `done_reason = "length"` and no assembled content is
  now a **terminal** error rather than a retryable one. That condition is
  deterministic — the identical request exhausts the identical cap — so the
  previous behavior spent three attempts and held the work-unit claim three
  times as long on a failure that could not recover. Transient empty responses
  under any other `done_reason` keep the existing retry and backoff behavior
  unchanged.
- The resulting error now names the cause, reporting the model, the thinking
  byte count, and the generated token count, instead of returning a generic
  empty-response message.

## Model Scoring Registry

- `glm-5.3-flash` is set to `25.0x`, matched on the exact family so neither the
  earned `glm-5.2` tier (20x) nor the generic `glm-5` rule (10x) can reach or
  widen it.
- `kimi-k3` retains `50.0x`. Its basis line no longer claims an absence of
  network data, since operator tests are now on record.

Both scoring gates are unchanged and continue to sit between a declared tier and
a paid multiplier. The throughput gate assigns the `3.0x` large-local ceiling to
any cloud claim under 25 tokens/sec or missing its measurement. The
output-quality gate caps any contribution without at least one reportable
finding or three evidence-backed coverage items at `1.0x`. A work unit that
produces no content clears neither gate, so no multiplier is paid on it.

## Upstream

The remedy for the underlying failure is server-side at the provider, and no
client-side change substitutes for it:

1. Raise the output cap for thinking models on the `ollama.com` endpoint, or
2. Expose a separate thinking-budget parameter so reasoning tokens and answer
   tokens are budgeted independently.

## Verification

`cargo check` and `cargo test` pass on the modified crate. Model tier resolution
is covered by unit tests asserting that `glm-5.3-flash` resolves to 25x without
widening to `glm-5.3`, `glm-5.2`, or `glm-5.1`.
