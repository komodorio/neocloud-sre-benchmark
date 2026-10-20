# SREGym lite: Kimi K3 across inference providers

## What runs

| Item | Value |
|---|---|
| Benchmark | SREGym 1.1 (`SREGym/SREGym@a90b43cc`), SREGym-Lite cohort `sregym-lite-1004`, 17 problems |
| Agent | upstream's stock `claudecode` agent (Claude Code CLI installed by `install-claudecode.sh`), no prompt changes; tool definitions limited to SREGym's own allowed-tools list (see below) |
| Command per job | `uv run main.py --problem <id> --agent claudecode --model anthropic/<config> --judge-model anthropic/<config> --n-attempts 1` |
| Upstream defaults kept | `--internet-access filtered`, `--profile full`, `--agent-timeout 1800`, judge = agent model (upstream default) |
| Repeats | 3 attempts per problem, 51 runs per config, same as the published leaderboard |
| Isolation | one pod per (config, problem, attempt), own on-demand `c7i/m7i` amd64 node (14 vCPU / 26Gi requested), fresh kind cluster |
| Compute | KAOP (Komodor Agentic Operations Platform) Evaluation Factory infrastructure: EKS, privileged Docker-in-Docker; one pod per run, scheduled by our own harness |
| Model routing | LiteLLM `v1.101.6` sidecar on `127.0.0.1:4000`; Claude Code and the judge both call it (`ANTHROPIC_BASE_URL`, `JUDGE_API_BASE`) |

## Configs

| Config | Route |
|---|---|
| `claude-sonnet-5` | Bedrock `us.anthropic.claude-sonnet-5` (harness check against the published Claude Code + Claude Sonnet 5 row) |
| `kimi-k3-aws` | Bedrock OpenAI-compatible endpoint `us.moonshotai.kimi-k3` (bearer Bedrock API key) |
| `kimi-k3-modal` | Modal Shared Endpoint `moonshotai/Kimi-K3` (`inference.us-west.modal.direct/v1`) |
| `kimi-k3-together` | Together AI `moonshotai/Kimi-K3`, LiteLLM `together_ai/` route (Chat Completions) |
| `kimi-k3-nebius` | Nebius AI Studio `moonshotai/Kimi-K3`, LiteLLM `nebius/` route (Chat Completions; served FP4, eu-west2) |
| `kimi-k3-fireworks` | Fireworks `accounts/fireworks/models/kimi-k3`, LiteLLM `together_ai/` class pointed at Fireworks' endpoint, so `reasoning_content` is passed back (Chat Completions; FP8 per the Fireworks model API) |
| `kimi-k3-baseten` | Baseten `moonshotai/Kimi-K3`, LiteLLM `baseten/` route (Chat Completions; served FP8) |

All Kimi K3 configs drop `temperature`, `top_p`, `thinking` and `reasoning_effort` at the proxy, so every provider runs the
model's own default sampling and default reasoning, for both the agent and the judge. Without this, LiteLLM maps Claude Code's
thinking request differently per provider (Bedrock `thinking.budget_tokens=4096`, Together `reasoning_effort=high`, Nebius
nothing), and Bedrock rejects `temperature` for Kimi K3 outright. `calls.jsonl` records the reasoning fields each request
actually carried (`reasoning`), so this is checkable per call.

The OpenAI-compatible providers go through LiteLLM's provider-native routes. The generic `openai/` route translates Claude
Code's Messages calls into the OpenAI Responses API, which the providers reject once the conversation has an assistant turn
(`422 input.N.id: Field required`), so the agent exits after one or two turns. Every attempt made on that route is kept under
`failed-infra/responses-api/` and not scored.

All Kimi K3 configs run Claude Code with `CLAUDE_CODE_TOTAL_TOKENS_REMINDER=off` (SREGym's env allowlist is patched to forward it).
Stock Claude Code 2.1.302 appends a new `<total_tokens>N tokens left</total_tokens>` block to the system prompt on every
turn. A non-Anthropic backend sees a different system prompt each call, so no provider can prefix-cache: measured on Bedrock,
a reminder-on run (`cronjob_sidecar_blocks_completion_hotel_reservation`) wrote 4.26M tokens to cache and read 0 ($18.31), and a
reminder-off run (`network_policy_block`, a different problem) read 78% of its prompt tokens from cache ($3.86). The Sonnet 5 baseline keeps the stock default, since it only validates the harness against the
published row. Three Kimi attempts made with the reminder on are kept under `failed-infra/reminder-on/` and not scored.

Modal runs on its Shared Endpoint (per-token billing, $3 / $0.30 cached / $15 per MTok). Under parallel load the
shared endpoint aborts ~5% of streamed generations under parallel load (`finish_reason: "abort"`), cutting the agent's tool call.
Reproduced with direct API calls (no proxy, no agent), below Modal's documented 16-request concurrency limit; the same
requests sent to Fireworks had 0 aborts in 110. Modal's recommended routing-affinity header reduced the aborts but did not
stop them. In the benchmark 13 runs stopped early this way; each was re-run and all finished within 3 tries, so Modal
completed 51/51 like the other providers.

Every Kimi config sets LiteLLM `stream_timeout: 45`. Some Bedrock streams returned HTTP 200 and then sent no events, so
Claude Code's 60s pre-flight timed out; across 182 completed Bedrock calls the slowest first token took 10.0s, so 45s cannot cut
off a legitimate call. A stalled stream now fails, is retried, and shows up as a failed call in `calls.jsonl`.

Every Kimi config runs Claude Code with `--tools` set to SREGym's own `ClaudeCodeAgent.ALLOWED_TOOLS` (SREGym already
passes that list as `--allowedTools`; it only gated permission, so all tool definitions were still sent). Claude Code
2.1.302 then sends 9 tool definitions (`Agent, Bash, Edit, Glob, Grep, NotebookEdit, Read, Skill, Write`) instead of 21.
Reason: Kimi K3 on Bedrock degenerates into mixed-language gibberish when the tool definitions exceed roughly 52-59k
characters. Replaying a pod's exact pre-flight request directly against Bedrock: 21 tools (59,141 chars) degenerated in 4/4
calls, 18 tools (52,001 chars) and fewer were clean, no tools was clean 4/4, and the SREGym tool subset was clean 6/6. Every
Kimi provider gets the same reduced tool list so requests stay identical; the Sonnet baseline stays stock. Attempts made with
the full tool list are kept under `failed-infra/full-toolset/` and `failed-infra/bedrock-tool-degeneration/`.

Kimi K3 must receive its earlier reasoning back in multi-turn tool loops (model card: "trained with preserved thinking
history"; applications "must return the complete prior assistant message ... including reasoning content"). LiteLLM's
routes differ: `nebius/`, `baseten/` and `together_ai/` send `reasoning_content` back, while `fireworks_ai/` and
`bedrock/converse/` drop it. Measured effect in agent runs: 30% of turns carried thinking with reasoning sent back vs 5%
without (median thinking per run ~24k vs ~0.5k chars). Fireworks therefore runs through LiteLLM's `together_ai/` route class
pointed at Fireworks' endpoint (sends `reasoning_content`, not the `thinking_blocks` field Fireworks rejects); its canary
showed 23% thinking turns. The first 41 Fireworks runs (reasoning dropped) are kept under `failed-infra/reasoning-dropped/`
as a control, not scored. Bedrock's Converse route cannot carry it; Bedrock's OpenAI-compatible endpoint
(`bedrock-runtime/openai/v1/chat/completions`) accepts `reasoning_content` in history (verified), so `kimi-k3-aws` now runs there through LiteLLM's `together_ai/` class with a
bearer Bedrock API key; a local 3-turn Claude Code tool loop sent `reasoning_content` on every later request with 0 errors and
prefix-cache hits (2,792/3,194 prompt tokens on turn 2). The 51 Converse runs (52.9% E2E, reasoning dropped) are kept under
`failed-infra/reasoning-dropped/kimi-k3-aws-converse/` as a control, not scored.

Claude Code is pinned to 2.1.302 through SREGym's `agents.yaml` `agent_version` (upstream installs `latest` per run).

## Scoring

- Upstream's own conductor and judge produce `Diagnosis.success`, `Mitigation.success`, `TTL` (TTD), `TTM` per run.
- E2E = diagnosis and mitigation both correct on the same run.
- Same rule as upstream's `sregym/results/report.py`: an attempt with `run_status=complete` and both stage results is scored
  (a `complete` row with `timed_out=True` is scored); an `incomplete` attempt (any reason, including `agent_timeout` with
  missing stages and `agent_exited_before_all_stages_completed`) is inconclusive and re-run.
- Re-runs are capped at 3 per job. A job still incomplete after 3 re-runs is scored as a failure and listed per provider, so a
  provider that keeps stalling cannot be re-run into a better score. Re-run counts per provider are reported.
- Infra failures (deploy failed, pre-flight failed, no or corrupt result CSV) are re-run, never scored.
- Throttling: a run is infra (re-run) only if Claude Code itself received an API error, or an HTTP 429 outlasted LiteLLM's 3
  retries. A 429 that LiteLLM retried successfully never reaches the agent, so its trajectory and score are unaffected; the
  retry delay stays in that run's TTD/TTM and is counted per provider (`absorbed_429s`). Throttled re-runs are kept under
  `failed-infra/rate-limited/` and do not use up a job's 3 re-runs. When a job has two valid attempts, the earlier one counts
  (chosen by time, not outcome). Per-provider concurrency caps keep runs under each account's limit (Baseten 1, Together 2, Nebius 4, Fireworks 6, AWS Bedrock 6); Baseten's account throttles even one agent intermittently.
- 95% intervals are Wilson.

## Cost and latency

- `calls.jsonl` per run: one line per LLM call from the LiteLLM callback (start/end, time to first token, usage, caller user-agent).
- Cost = tokens x published per-provider price (`pricing.json`), cache reads at the cache price where the provider bills one.
  Nebius publishes no cache price and bills cached prompt tokens at the full input rate. Checked against its invoice
  after all runs: Nebius billed $313.28 and our log prices the same calls at $300.49 (-4%, the same gap on input and
  output tokens; the rest is calls from ~4 pods deleted mid-run, never collected).
- Latency: per-call latency and TTFT for the agent's calls, and SREGym's TTD/TTM per run.

## Modal routing affinity

Modal Shared Endpoints spread requests across containers, so consecutive agent turns land on different prefix caches. A local
5-turn Claude Code tool loop without an affinity key hit the cache on 2 of 5 calls (cached tokens: none, 2,816, 3,072, none, 128).
Modal's docs ask agent loops to send `Modal-Routing-Affinity-Key` with one value per conversation. Each pod's LiteLLM sends its
pod name (`MODAL_AFFINITY_KEY` from the downward API) via `extra_headers`; with the key, the second call hit 3,072 of 3,191
prompt tokens. One key per run keeps a whole agent session on one container, the same as a user following Modal's docs.

## Pre-flight throttling

SREGym runs `claude -p "say ok"` with a 60s limit before the agent starts. When a provider answers that call only with HTTP 429
until the limit expires, the run aborts before any agent work (`RuntimeError: Pre-flight check failed`). These runs are filed under
`failed-infra/rate-limited/` and re-run without using the run's 3-attempt infra budget, the same as runs where the agent saw a
rate-limit error. Counts: Baseten 13, Together 1.
