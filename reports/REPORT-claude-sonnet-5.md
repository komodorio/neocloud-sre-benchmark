# Kimi K3 on SREGym-Lite across inference providers

Same agent (Claude Code), same 17 incidents × 3 attempts = 51 runs per provider; only the provider changes. Method and every decision: [METHOD.md](METHOD.md). Prices and sources: [pricing.json](pricing.json).

## Correctness

| Provider | Graded | Diagnosis | Mitigation | E2E | E2E 95% CI |
|---|---|---|---|---|---|
| Claude Sonnet 5 (Bedrock, reference) | 51 | 58.8% | 64.7% | **45.1%** | 32–59 |

E2E = the agent both diagnosed the root cause (LLM judge) and fixed the cluster (oracle). Published leaderboard, Claude Code + Claude Sonnet 5: diagnosis 54.9%, mitigation 62.7%, **E2E 45.1%**.

## Latency

| Provider | TTFT p50 | Call p50 | Call p95 | Output tok/s | Agent time / scenario (mean) | (median) | 17 scenarios | 51 runs |
|---|---|---|---|---|---|---|---|---|
| Claude Sonnet 5 (Bedrock, reference) | - | 4.5s | 24.2s | 102 | 6.5 min | 4.4 min | 1.8 h | 5.5 h |

TTFT = time to first token per LLM call. Agent time = wall-clock for one scenario attempt; the 17 / 51 columns are that time × runs, back to back.

## Cost

| Provider | Price in / cached / out ($ per 1M) | Input tokens / run (cached) | Output tokens / run | Cost / run | 17 scenarios | 51 runs |
|---|---|---|---|---|---|---|
| Claude Sonnet 5 (Bedrock, reference) | 2.2 / 0.22 / 11.0 (+2.75 cache write) | 3.27M (98%) | 30.1K | $1.22 | $20.66 | $61.97 |

Cost = tokens each provider reported for every call (agent + judge) × its published list price. Credits play no part.

### Checked against the vendors' own bills (read in each console on 2026-10-19)

| Vendor | Vendor billed | Our log, same window | Gap | Note |
|---|---|---|---|---|
| Nebius | $313.28 | $300.49 | -4% | finished; same 4% on input and output tokens, so per-call counts match and ~4 killed-mid-run pods are missing from our log |
| Baseten | $63.88 | $58.50 | -8% | read while runs were still in flight |
| Together AI | $11.92 | $10.70 | -10% | read while runs were still in flight |
| Modal | $11.08 | $10.31 | -7% | includes our direct API probes |
| AWS Bedrock (Oct 18) | $91.21 | $82.89 | -9% | discarded Converse route: it under-reports reasoning output tokens |

Our per-run cost is, if anything, a few percent conservative.

## Reliability

| Provider | LLM calls | Rate-limited calls (absorbed by retries) | Agent re-runs | Rate-limit re-runs (not counted) | Incomplete after 3 re-runs |
|---|---|---|---|---|---|
| Claude Sonnet 5 (Bedrock, reference) | 2,096 | 0 (0%) | 10 | 0 | 3 |
