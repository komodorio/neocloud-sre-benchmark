# Neocloud SRE Benchmark: Kimi K3 across inference providers

[Komodor](https://komodor.com) builds the agent runtime, Evaluation Factory and AI SRE
for production AI agents. We took an AI SRE agent running on a closed model (Claude Code on Claude Sonnet 5), switched it
to the open-weight Kimi K3, and ran it through six inference providers on [SREGym](https://github.com/SREGym/SREGym)-Lite:
17 broken Kubernetes clusters, 3 runs each.

**Results page:** https://komodorio.github.io/neocloud-sre-benchmark/

## Results

<!-- results:start -->
| Provider | Runs | E2E | Cost / run | Session p50 | LLM call p50 | TTFT p50 | Output tok/s | Tokens / run |
|---|---|---|---|---|---|---|---|---|
| Kimi K3 · AWS Bedrock | 51/51 | 68.6% | $0.87 | 4.0 min | 5.1s | 2.74s | 73 | 1.23M |
| Kimi K3 · Fireworks | 51/51 | 66.7% | $0.66 | 7.5 min | 11.2s | 3.36s | 30 | 0.98M |
| Kimi K3 · Nebius | 51/51 | 66.7% | $4.72 | 3.0 min | 3.1s | 0.83s | 115 | 1.49M |
| Kimi K3 · Modal | 51/51 | 66.7% | $0.84 | 4.9 min | 5.8s | 2.29s | 62 | 1.34M |
| Kimi K3 · Baseten | 51/51 | 62.7% | $0.74 | 5.7 min | 6.2s | 2.60s | 57 | 1.07M |
| Kimi K3 · Together AI | 51/51 | 58.8% | $0.65 | 7.3 min | 9.1s | 2.40s | 36 | 0.96M |
| Claude Sonnet 5 · Bedrock (baseline) | 51/51 | 45.1% | $1.22 | 4.4 min | 4.5s | - | 102 | 3.30M |

Updated 2026-10-20 07:51 UTC.
<!-- results:end -->

E2E = the agent found the root cause and fixed the cluster. Session p50 = median time the agent worked on one run.

## Findings

- **Open weights paid off.** Kimi K3 beat the Sonnet 5 baseline on every provider (58.8–68.6% vs. 45.1% E2E success) and cost less per run on 5 of 6.
- **The provider changed the bill, not the accuracy.** Cost per run varied 7.3× ($0.65–$4.72) and session time 2.5× (3.0–7.5 min), while E2E success stayed within run-to-run noise.
- **The list price hides the real bill.** 95–97% of an agent's prompt tokens are cache hits, and Nebius bills them at the full input price.
- **Rate limits cap how many agents you can run.** Together AI throttled 23.8% of calls at 2 agents in parallel and Baseten 4.5% at 1; the others had none.

## Method

Same agent (SREGym's stock Claude Code, CLI 2.1.302), prompt, tools and grading on every Kimi K3 provider; only the
provider changes. Each run gets a fresh cluster and its own node, and every model call is logged and priced at the provider's list
price. Full details: [METHOD.md](METHOD.md) · tables: [REPORT.md](REPORT.md) · prices: [pricing.json](pricing.json) ·
every number on the page: [docs/data.json](docs/data.json).
