# Prior Art: Token-Spike Detection & Agent-Spend Throttling

**For:** OpenClaw token-spike guardrails, OpenClaw kanban row k_5eca5366
**Date:** 2026-06-11
**Author:** Claude session 627cb8a0 (background `researcher` agent + main session synthesis)

## Scope

Survey the field of agent-spend control to inform a guardrail design for David's Claude Code rig after the 2026-06-10 Fable-5 burn (5hr window exhausted in ~20 min + $50 API spend). Goal: borrow the right ideas instead of reinventing.

## 1. LiteLLM — gateway-level budget & rate limits

**What it is.** Open-source LLM proxy/gateway (`BerriAI/litellm`) that fronts multiple LLM providers. Used in production by teams that need spend caps spanning many model vendors.

**Mechanism.**
- Multi-tier budgets: per-virtual-key, per-user, per-team, per-tag, per-end-customer. Stack from broad to narrow; the lowest-tier cap binds.
- Rate limits: TPM (tokens-per-minute) and RPM (requests-per-minute) per key/user/model.
- Enforcement is reject-at-proxy: 429 to caller, no provider call made.
- Streams cost/usage events to a Postgres ledger; dashboard + Prometheus exporter.
- Budget reset cycles: daily / weekly / monthly.

**What's applicable.**
- The **tier-stacking** pattern is what David's rig needs even at single-user scale: an "always-on driver" budget *and* a per-task budget that catches runaway loops.
- LiteLLM-as-gateway is the heavy path — would mean running a daemon and pointing Claude Code's `ANTHROPIC_BASE_URL` at it. Worth keeping as the "if-shell-hooks-aren't-enough" fallback.

**Sources** (retrieved 2026-06-11):
- https://docs.litellm.ai/docs/proxy/users
- https://docs.litellm.ai/docs/proxy/cost_tracking

## 2. OpenTelemetry GenAI semantic conventions

**What it is.** Standardized telemetry schema for LLM/agent observability. Defines metric names and span attributes that any compliant tool can emit and any collector can consume.

**Mechanism.**
- Core metric: `gen_ai.client.token.usage` (Histogram, attributes `gen_ai.system`, `gen_ai.operation.name`, `gen_ai.token.type ∈ {input, output}`).
- Spans carry `gen_ai.request.model`, `gen_ai.response.model`, finish reason, etc.
- Collectors can apply sampling/redaction; enforcement is downstream (alerting rules in Grafana/Prometheus).

**What's applicable.**
- For a one-machine rig, OTel is overkill — but the *attribute names* are worth borrowing so the rig's own logs are convertible later.
- Specifically: emit `gen_ai.token.usage` with `model` + `cache_status` (read/created/none) so cache-cost separation falls out for free.

**Sources:**
- https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/

## 3. Anthropic rate-limit response headers

**What it is.** Anthropic's API surfaces remaining-quota headers on every response. Claude Code already exposes a subset via its statusLine JSON contract.

**Mechanism.** Per-response headers include:
- `anthropic-ratelimit-input-tokens-remaining`
- `anthropic-ratelimit-input-tokens-reset` (ISO 8601 timestamp)
- `anthropic-ratelimit-output-tokens-remaining` / `-reset`
- `anthropic-ratelimit-requests-remaining` / `-reset`
- 429 responses include `retry-after` in seconds.

The **5-hour window** that David's subscription burned through is a Claude.ai/Claude Code subscription-tier concept, not a direct API header. Claude Code's statusLine `rate_limits.five_hour.used_percentage` is the closest direct read.

**What's applicable.**
- Read `rate_limits.five_hour.used_percentage` from the statusLine JSON every turn; that's the canonical signal.
- Maintain a small ring buffer of `(timestamp, used_percentage)` samples to compute deltas.
- The 15%/10-min rule maps directly: if `pct(now) − pct(now − 10min) > 15`, trigger.

**Sources:**
- https://platform.claude.com/docs/en/api/rate-limits
- https://code.claude.com/docs/en/statusline (statusLine JSON contract)

## 4. ccusage — community Claude-Code spend monitor

**What it is.** `ryoppippi/ccusage` is a third-party CLI + statusline integration that reads Claude Code's local transcript ledger and reports spend, broken down by 5-hour blocks, daily, model, and project.

**Mechanism.**
- Reads `~/.claude/projects/**/*.jsonl` directly (no API).
- Aggregates `usage` blocks per turn, attributes cost by model.
- Provides a statusline binary that prints live consumption (token totals, dollar estimates, % of 5hr block used).
- Configurable "block report" mode shows the active 5-hour window with elapsed/remaining time.

**What's applicable.**
- Best signal *source*: ccusage already does the JSONL-to-cumulative-spend math. The rig can either invoke `ccusage statusline --json` and reuse the output, or replicate the JSONL parsing inline.
- ccusage doesn't ship spike-detection or auto-throttle out of the box (as of 2026-06) — David's rig adds that layer on top.
- The 5-hour block accounting is non-trivial (anchored to first request in window, not wall-clock midnight); borrow ccusage's logic rather than re-deriving.

**Sources:**
- https://github.com/ryoppippi/ccusage
- https://ccusage.com/
- https://ccusage.com/guide/statusline (5-hour block reports)

## 5. Claude Code statusLine — the local hook point

**What it is.** Claude Code's `settings.json` `statusLine.command` runs a shell command every ~few seconds; the command receives a JSON blob on stdin (model, session, transcript path, costs, rate limits) and emits a single line of statusbar text on stdout.

**Mechanism.**
- Input JSON includes: `session_id`, `model`, `transcript_path`, `total_cost_usd`, `total_input_tokens`, `total_output_tokens`, `cache_read_tokens`, `cache_creation_tokens`, `rate_limits.five_hour.{used_percentage, reset_at}`, `rate_limits.weekly`, etc.
- Output is purely cosmetic (status bar text), BUT the script can have side effects: write to a state file, write a sentinel to a watchdog, etc.
- Runs ~every 5 seconds while a session is active.

**What's applicable.**
- The cheapest signal-emit path: statusLine writes `(ts, five_hour_pct, cumulative_tokens)` to `~/.claude/memory/state/spike.jsonl` on every tick. Spike-detector reads the last 10 min and triggers if delta > 15%.
- Statusline already exists in David's rig (`/Users/davidsteeves/.claude/statusline-command.sh`); the spike logger is a 20-line addition.

**Sources:**
- https://code.claude.com/docs/en/statusline
- https://github.com/daniel3303/ClaudeCodeStatusLine (community implementation showing the JSON contract in practice)

## 6. Circuit-breaker patterns for runaway agents

**What it is.** Pattern lineage from microservices (Netflix Hystrix → resilience4j) applied to LLM agents. The core idea: trip when a fast-moving signal crosses a threshold, then refuse new requests until cool-down.

**Patterns surveyed.**
- **LangChain agent loops:** `max_iterations` (default 15) and `max_execution_time` (wall-clock seconds). Hard caps that abort the loop with a friendly "couldn't finish" reply. *Crude but effective for "agent stuck in tool-call loop" failure modes.*
- **AutoGPT / similar:** declarative `budget_usd` per task; agent stops and reports when reached.
- **Token-velocity detection** (TokenFence, "cost circuit breaker" blog patterns): rolling-window rate; if current-window rate > N × trailing-baseline-rate, trip. Trip mode is configurable: warn, downgrade model, hard-block.
- **Multi-tier budgets** (Runyard "swarm budgets"): swarm-wide cap → per-agent-type cap → per-task cap. Lowest cap wins; useful for hierarchical agent fan-outs.
- **Graceful degradation** (TokenFence): at 80% budget consumed, downgrade to cheaper model rather than block; at 95%, block.
- **Claude Code Workflow hard cap:** workflows cap at 1000 agent invocations total. Backstop, not a fine-grained guardrail.

**What's applicable.**
- The **velocity rule** is what the kanban row already proposed (15% of 5hr window in 10 min). Borrow the threshold-and-cooldown shape, but anchor it to the Claude-specific 5hr block, not a wall-clock minute window.
- The **graceful-degradation** pattern is the right way to express the Fable-5 rule: when a task is expensive, *downgrade* rather than *block* — but require explicit re-opt-in to use a premium model (Fable-5/Opus-4.8) for the next prompt.
- The **multi-tier budget** stack is a future direction. v1 of David's rig only needs a single global tier.

**Sources:**
- https://dev.to/thedailyagent/how-to-stop-ai-agent-cost-spirals-before-they-start-4ce7
- https://runyard.io/blog/swarm-budgets-cost-control
- https://tokenfence.dev/
- https://dev.to/sebastian_chedal/the-cost-circuit-breaker-how-we-prevent-runaway-spending-across-9-ai-agents-4i5k
- https://python.langchain.com/docs/modules/agents/how_to/max_iterations/
- https://code.claude.com/docs/en/workflows (1000-agent cap)

## Synthesis — what David should borrow

Ranked by fit × inverse-cost-to-build for the rig:

1. **statusLine writes a spike ledger** *(eat your own dogfood)* — extend the existing `statusline-command.sh` to append a `(ts, five_hour_pct, cumulative_tokens, model)` line to `~/.claude/memory/state/spike.jsonl`. Zero new daemons. ~20 LOC.
2. **UserPromptSubmit hook reads the ledger** — before each new prompt, tail the spike ledger, compute Δ-pct over last 10 min. If > 15%, inject a warning into `additionalContext`. If > 25%, hard-block with a stdout message ("burn-rate guardrail: paused, edit ~/.claude/state/spike-unblock to resume"). Modeled on Claude Code's `UserPromptSubmit` hook contract. ~40 LOC.
3. **Borrow ccusage's 5hr-block math, don't re-derive** — invoke `ccusage statusline --json` from the spike logger if installed; fall back to inline parse if not. Saves a class of subtle bugs around block anchoring.
4. **Graceful degradation on premium-model use** — when `model ∈ {claude-fable-5, claude-opus-4-8}` AND the user did not type the model name in the last prompt, emit a confirmation prompt. This is the policy fix that the 2026-06-10 burn would have caught.
5. **OTel-style attribute naming in logs** — name the spike-ledger fields like OTel's GenAI conventions so the data is portable if David later wires it into a real collector.
6. **Multi-tier budgets** — defer to v2. v1 is one window, one threshold.
7. **LiteLLM as gateway** — defer indefinitely. Only justified if Claude Code's local signals prove insufficient.

The **velocity rule + premium-model confirm-prompt** is the minimum-viable guardrail. Both fit inside existing rig conventions (statusLine + UserPromptSubmit hook) — no new infrastructure required.
