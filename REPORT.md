# Kimi K3 on SREGym-Lite across inference providers

Same agent (Claude Code), same 17 incidents × 3 attempts = 51 runs per provider; only the provider changes. Method and every decision: [METHOD.md](METHOD.md). Prices and sources: [pricing.json](pricing.json).

## Correctness

| Provider | Graded | Diagnosis | Mitigation | E2E | E2E 95% CI |
|---|---|---|---|---|---|
| AWS Bedrock | 51 | 82.4% | 76.5% | **68.6%** | 55–80 |
| Fireworks | 51 | 82.4% | 76.5% | **66.7%** | 53–78 |
| Nebius | 51 | 76.5% | 78.4% | **66.7%** | 53–78 |
| Modal | 51 | 88.2% | 72.5% | **66.7%** | 53–78 |
| Baseten | 51 | 74.5% | 72.5% | **62.7%** | 49–75 |
| Together AI | 51 | 74.5% | 74.5% | **58.8%** | 45–71 |
| Claude Sonnet 5 (Bedrock, reference) | 51 | 58.8% | 64.7% | **45.1%** | 32–59 |

E2E = the agent both diagnosed the root cause (LLM judge) and fixed the cluster (oracle). Published leaderboard, Claude Code + Claude Sonnet 5: diagnosis 54.9%, mitigation 62.7%, **E2E 45.1%**.

## Latency

| Provider | TTFT p50 | Call p50 | Call p95 | Output tok/s | Agent time / scenario (mean) | (median) | 17 scenarios | 51 runs |
|---|---|---|---|---|---|---|---|---|
| Nebius | 0.83s | 3.1s | 13.4s | 115 | 5.1 min | 3.0 min | 1.5 h | 4.4 h |
| AWS Bedrock | 2.74s | 5.1s | 14.0s | 73 | 5.4 min | 4.0 min | 1.5 h | 4.6 h |
| Claude Sonnet 5 (Bedrock, reference) | - | 4.5s | 24.2s | 102 | 6.5 min | 4.4 min | 1.8 h | 5.5 h |
| Modal | 2.29s | 5.8s | 21.0s | 62 | 7.1 min | 4.9 min | 2.0 h | 6.0 h |
| Baseten | 2.60s | 6.2s | 23.3s | 57 | 8.2 min | 5.7 min | 2.3 h | 6.9 h |
| Together AI | 2.40s | 9.1s | 34.6s | 36 | 9.1 min | 7.3 min | 2.6 h | 7.7 h |
| Fireworks | 3.36s | 11.2s | 37.8s | 30 | 9.6 min | 7.5 min | 2.7 h | 8.2 h |

TTFT = time to first token per LLM call. Agent time = wall-clock for one scenario attempt; the 17 / 51 columns are that time × runs, back to back.

## Cost

| Provider | Price in / cached / out ($ per 1M) | Input tokens / run (cached) | Output tokens / run | Cost / run | 17 scenarios | 51 runs |
|---|---|---|---|---|---|---|
| Together AI | 3.0 / 0.3 / 15.0 | 0.94M (96%) | 16.4K | $0.65 | $11.00 | $32.99 |
| Fireworks | 3.0 / 0.3 / 15.0 | 0.97M (96%) | 17.5K | $0.66 | $11.21 | $33.64 |
| Baseten | 3.0 / 0.3 / 15.0 | 1.05M (95%) | 18.1K | $0.74 | $12.54 | $37.63 |
| Modal | 3.0 / 0.3 / 15.0 | 1.32M (97%) | 21.6K | $0.84 | $14.33 | $42.99 |
| AWS Bedrock | 3.3 / 0.33 / 16.5 (+4.125 cache write) | 1.21M (96%) | 19.1K | $0.87 | $14.83 | $44.49 |
| Claude Sonnet 5 (Bedrock, reference) | 2.2 / 0.22 / 11.0 (+2.75 cache write) | 3.27M (98%) | 30.1K | $1.22 | $20.66 | $61.97 |
| Nebius | 3.0 / no discount / 15.0 | 1.46M (97%) | 21.8K | $4.72 | $80.22 | $240.65 |

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
| Fireworks | 1,779 | 0 (0%) | 6 | 0 | 1 |
| Nebius | 1,992 | 0 (0%) | 1 | 0 | 0 |
| Baseten | 1,921 | 87 (5%) | 3 | 20 | 0 |
| AWS Bedrock | 1,916 | 0 (0%) | 1 | 0 | 0 |
| Together AI | 2,259 | 538 (24%) | 2 | 12 | 0 |
| Modal | 2,004 | 0 (0%) | 14 | 1 | 0 |

**Together AI rate limiting.** Together's limits are dynamic and follow its live capacity. Even at 2 agents in parallel, 24% of calls in graded runs drew HTTP 429 and were retried, which adds waiting time to every run. At times it throttled almost everything: the worst run got 429 on 190 of 217 calls. 12 runs ended this way and were re-run rather than scored, so its E2E reflects the model, while its latency and wall-clock time reflect the throttling.

**Modal stream aborts.** Under parallel load Modal's shared endpoint cancels some generations mid-stream (finish_reason "abort") right before the tool call, so the agent thinks the model ended its turn and stops. Reproduced with plain direct API calls: 9 of 180 parallel requests aborted, 0 of 60 sequential, against 0 of 110 on Fireworks. In the benchmark 13 runs stopped early this way and were re-run (at most 3 times each); 0 still failed after 3 tries. Its E2E therefore reflects the model; the aborts cost re-runs and wall-clock time.

**Baseten account limit.** The default tier allows 100,000 tokens and 15 requests per rate-limit window; one agent re-sending a long transcript nearly fills it, so Baseten ran one agent at a time. 20 runs that were throttled before or during the agent's work were re-run rather than scored.

## Consistency across providers

E2E passes per incident on attempts every provider has graded (51 attempts).

| Incident | Fireworks | Nebius | Baseten | AWS Bedrock | Together AI | Modal |
|---|---|---|---|---|---|---|
| cronjob_sidecar_blocks_completion_hotel_reservation | 1/3 | 3/3 | 1/3 | 2/3 | 2/3 | 2/3 |
| duplicate_pvc_mounts_social_network | 1/3 | 0/3 | 0/3 | 2/3 | 0/3 | 2/3 |
| edge_request_filter_cpu_saturation | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 |
| env_variable_shadowing_astronomy_shop | 3/3 | 2/3 | 3/3 | 3/3 | 3/3 | 3/3 |
| finalizer_deadlock_controller_hotel_reservation | 3/3 | 2/3 | 2/3 | 1/3 | 3/3 | 3/3 |
| internal_traffic_policy_local_astronomy_shop | 3/3 | 3/3 | 3/3 | 3/3 | 2/3 | 2/3 |
| kafka_poison_pill_hol_block | 2/3 | 1/3 | 3/3 | 2/3 | 3/3 | 1/3 |
| mutating_webhook_resource_limits_social_network | 1/3 | 1/3 | 0/3 | 0/3 | 0/3 | 1/3 |
| namespace_memory_limit | 1/3 | 2/3 | 2/3 | 2/3 | 1/3 | 1/3 |
| network_policy_block | 3/3 | 3/3 | 3/3 | 2/3 | 3/3 | 3/3 |
| rolling_update_misconfigured_social_network | 0/3 | 0/3 | 0/3 | 0/3 | 0/3 | 0/3 |
| search_rate_retry_collapse_hotel_reservation | 0/3 | 0/3 | 1/3 | 1/3 | 0/3 | 0/3 |
| secret_rotation_stale_env_credentials_astronomy_shop | 3/3 | 3/3 | 2/3 | 3/3 | 2/3 | 2/3 |
| service_dns_resolution_failure_social_network | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 |
| service_wrong_pod_selection_hotel_reservation | 2/3 | 3/3 | 1/3 | 2/3 | 2/3 | 3/3 |
| unschedulable_incorrect_port_assignment | 2/3 | 2/3 | 2/3 | 3/3 | 1/3 | 2/3 |
| valkey_auth_disruption | 3/3 | 3/3 | 3/3 | 3/3 | 2/3 | 3/3 |
