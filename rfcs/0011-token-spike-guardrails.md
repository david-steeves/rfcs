---
title: Token-Spike Guardrails for OpenClaw Sessions
authors:
  - David Steeves
created: 2026-06-11
last_updated: 2026-06-11
status: draft
issue:
rfc_pr:
---

# Proposal: Token-Spike Guardrails for OpenClaw Sessions

## Summary

A two-layer guardrail for OpenClaw-compatible Claude Code rigs that detects token-spend spikes before window exhaustion and blocks the specific failure mode that burned David Steeves's rig on 2026-06-10 (a runaway scheduled-task cron that fired the same prompt 142×, including 79× in 4 minutes). Layer one is a *velocity rule* (15 percentage points of the 5-hour window consumed in any 10-minute span trips a warning, escalates to model-downgrade, then to block). Layer two is a *cron-loop detector* (same-prompt-hash dedupe within 60s + output-spiral detector + per-prompt fire ceiling). Both layers hook into the existing `statusLine` and `UserPromptSubmit` extension points — no new daemons, no new infrastructure, ~340 LOC of shell and a one-line `settings.json` patch.

## Motivation

On 2026-06-10 David's rig burned its 5-hour subscription window in ~20 minutes plus $50 of overflow credits. The first diagnosis blamed a config change that swapped the default driver model to a premium tier (Fable-5). A post-mortem of all five session JSONLs from that day showed that diagnosis was wrong: Fable-5 never executed a turn, because Claude Code 2.1.101 does not hot-reload the `model` field from `settings.json`. The real cause was a `CronCreate`-registered prompt that fired 142 times in a single 10-day-old session, including a single-burst of 79 fires in ~4 minutes paying ~128K cache-read tokens each, with the model responding "Skipping." (9 output tokens). The total day spend was ~66.9M tokens, of which ~15.4M (~23%) was waste from the runaway cron.

The original "premium tier as default = burn" rule (already encoded in David's memory feedback) is correct and still load-bearing as a separate policy. But it is *necessary and not sufficient*: any rig running scheduled tasks against subscription auth is one bad recurring prompt away from window exhaustion. OpenClaw's promise of agentic operation makes scheduled tasks a first-class feature, which makes this failure mode a structural risk for every adopter. A standard guardrail belongs in the rig's substrate, not as a per-user invention.

## Goals

- Detect spend velocity that puts the 5-hour subscription window at risk **before** exhaustion, with a default rule of *15 percentage points of the window consumed in any 10-minute span*.
- Block the runaway-cron failure mode at the second fire, not the seventy-ninth, via a deterministic same-prompt-hash dedupe.
- Catch the "model thinks there's nothing to do" failure mode via an output-spiral detector (≥N consecutive turns where `output_tokens < 50` AND `cache_read_tokens > 50_000`).
- Make the *premium-tier-as-default* failure visible: a settings-divergence indicator on the statusbar, and an explicit per-session opt-in before any prompt is sent against `claude-fable-5`, `claude-opus-4-8`, or `claude-opus-4-7`.
- Cap session age at 24h (warn) / 7d (block) to prevent the contributing-factor "10-day-old session with stale crons".
- Hook into the existing `statusLine` and `UserPromptSubmit` extension points only. No new daemons. No new permissions.
- Fail open. Any guardrail-script error logs and exits 0; a broken guardrail must never become the reason a prompt is lost.

## Non-Goals

- Per-tool, per-agent, or hierarchical multi-tier budget stacks. Deferred to v2.
- Cross-session aggregation. Each session is treated as the unit; multi-window aggregation is a v2 question that needs careful design around subscription-account vs session identity.
- Replacing `ccusage` or Anthropic's billing dashboards as the source of truth for retrospective accounting. The guardrail's accounting is for *prevention*, not audit.
- Acting as a gateway proxy. LiteLLM-style proxy enforcement is the heavy fallback if local-hook signals prove insufficient; for now, OpenClaw rigs can ship the local hooks without a gateway.

## Proposal

### Architecture

```
                          settings.json
                                |
        +------------------------+--------------------+
        |                                             |
    statusLine                                UserPromptSubmit hook chain
   "command":...                              [prompt-context.sh, spike-check.sh, cron-loop-check.sh]
        |                                             |
        v                                             v
   spike-log.sh        --append--->   ~/.claude/memory/state/spike.jsonl
   (statusline                              (ring buffer of
    passthrough +                            statusline ticks)
    spike-ledger
    writer)
                                                      |
                                                      v
                                              spike-check.sh
                                              cron-loop-check.sh
                                              session-age-check.sh
                                              premium-confirm-check.sh
                                                      |
                                          (warn / degrade / block)
                                                      |
                                                      v
                                          stdin to next hook or model
```

### Layer 1 — Spike velocity rule

A statusline-side passthrough script (`spike-log.sh`) appends each statusline tick to `~/.claude/memory/state/spike.jsonl`:

```json
{"ts":"2026-06-11T13:42:01Z","session":"627cb8a0","model":"claude-opus-4-7",
 "pct":12.4,"cum_in":425112,"cum_out":38211,
 "cache_read":1882000,"cache_create":412000}
```

On every `UserPromptSubmit`, `spike-check.sh` tails the last 10 min of the ledger for the active session and computes `Δpct`. If `Δpct > 15`:

| Trip mode | Behavior | Trigger |
|-----------|----------|---------|
| `warn` (default) | statusbar `BURN N% /10m ⚠`; inject `<spike-warning>` in `additionalContext` | first trip this window |
| `degrade` | as `warn` + advisory: "fall back to Sonnet next turn unless judgment-heavy" | second trip OR `used_pct > 60` |
| `block` | hook exits non-zero with stdout message; user must `touch ~/.claude/state/spike-unblock` | `used_pct > 85` OR `SPIKE_MODE=block` env |

### Layer 2 — Cron-loop detector

Three sub-rules in `cron-loop-check.sh`. The dedupe and ceiling rules are deterministic; the spiral rule is statistical.

**Same-prompt-hash dedupe.** Compute `sha256(normalized_prompt)` where normalized = `re.sub(r'\s+', ' ', body).strip().lower()`. Per-session ring of (`hash`, `ts`). On `UserPromptSubmit`, if the same hash appears in the ring within the last 60s, the hook exits non-zero with:

```
cron-loop guard: duplicate prompt suppressed (hash=ab12cd34)
```

**Output-spiral detector.** Read the last 5 turns from `spike.jsonl` for this session. If for every turn `output_tokens < 50` AND `cache_read_tokens > 50_000`, block:

```
cron-loop guard: 5 consecutive zero-output high-cache-read turns suggest a
runaway scheduled task. Pause the cron and edit ~/.claude/state/spike-unblock
to resume.
```

**Per-prompt fire ceiling.** Track `~/.claude/memory/state/cron-fires.<session>.json` keyed by prompt hash. On the 49th fire of any single hash within a session (once-every-15-min for 12h), block:

```
cron-fire ceiling exceeded for prompt hash <h>. This usually means the schedule
is too aggressive or never converges. Disable the cron via /scheduled-tasks delete.
```

### Premium-model confirm-prompt

Independent of spike/cron rules, runs at the same `UserPromptSubmit` hook. If the active model is in `{claude-fable-5, claude-opus-4-8, claude-opus-4-7}` AND no per-session opt-in file exists for that model, exit non-zero with:

```
Premium model <name> requires per-session opt-in.
Reply with /opt-in-premium <name> or switch driver via /model.
```

A `/opt-in-premium <name>` slash-command skill writes `~/.claude/state/premium-optin.<session>.json` keyed by model. Opt-in lasts the session; auto-cleared on session end.

### Session-age check

`UserPromptSubmit` injects `<session-age-warning>session is N days old</session-age-warning>` when the session is >24h old. After 7 days, escalate to `block` mode (matches spike rule).

### Settings-divergence indicator

`spike-log.sh` reads `~/.claude/settings.json`'s `model` field and compares to the `model` in the statusline JSON. If they disagree, statusbar shows `MODEL≠SETTINGS <session>=<a> <file>=<b>`. Informational only.

### Authentication note

Headless `claude -p` invocations (used by the existing `daily-distill.sh` cron and the new `morning-prep.sh` LaunchAgent) MUST NOT pass `--bare`. `--bare` disables keychain reads and Anthropic OAuth auth, which causes "Not logged in" failures in cron/LaunchAgent contexts. The 2026-06-10 incident occurred against a subscription-auth-only rig where `--bare` had been the silent reason the distill cron had stopped producing summaries for weeks. This is documented inline in both scripts.

## Rationale

**Why two rules instead of one.** The spike-velocity rule would have caught the 2026-06-10 burn around the 22:34 burst, after ~5M tokens already spent. The dedupe rule catches it on the second fire, before $0.05 of waste. They detect different failure modes — velocity catches *any* runaway, dedupe catches *recurring identical prompts*. Both are needed.

**Why fail-open.** A guardrail that crashes shouldn't take prompts with it. Any internal error logs and exits 0; the prompt proceeds unguarded. This is the same pattern as production circuit breakers.

**Why the 15% / 10min threshold.** Sustained 100% utilization of a 5-hour window = 3.33% / 10min. 15% / 10min is 4.5× that. The 22:34 burst on 2026-06-10 hit ~16% (assumption: 80M-token ceiling; exact rate-limit headers were not in the JSONLs). Tunable as we collect data, but this is a reasonable starting calibration that catches the post-mortem incident and doesn't false-positive a 10-agent parallel audit (~8–10% / 10min).

**Why borrow ccusage's 5hr-block math.** ccusage already correctly anchors the 5hr window to the first request in the window (not wall-clock midnight). Re-deriving this is a class of subtle bugs we can skip by invoking `ccusage statusline --json` if installed, falling back to inline parse otherwise.

**Why hook-based, not gateway-based.** Claude Code's `statusLine` JSON contract already surfaces every signal we need (`rate_limits.five_hour.used_percentage`, model, cumulative tokens). LiteLLM-as-gateway would be the right answer if we needed cross-vendor budget caps, but a single-vendor subscription rig can ship local hooks in ~340 LOC. The gateway path is documented in `docs/research-prior-art.md` as the v2 fallback.

**Why the premium-tier opt-in is a separate rule.** The 2026-06-10 incident demonstrated that even with the right diagnosis, the post-incident rule "never silently default to a premium tier" is independently load-bearing. A user changing `model` in `settings.json` may not realize it doesn't hot-reload; a future `claude` version may hot-reload it and instantly bill at 2× the prior rate. The opt-in confirm-prompt makes the policy actionable at the hook layer.

## Rollout

1. Land `spike-log.sh` and the statusLine patch first (collection only, no enforcement). 48h validation period.
2. Add `spike-check.sh` in `warn` mode. 48h. Tune the threshold if false-positive rate is wrong.
3. Add `cron-loop-check.sh` with all three sub-rules in `block` mode (dedupe is too cheap to gate progressively).
4. Enable `degrade` mode on spike rule. One week.
5. Enable `block` mode on spike rule at >85%. Document the unblock procedure.
6. Add premium-model opt-in and `/opt-in-premium` slash-command.

Each phase is reversible by reverting one file.

## Unresolved questions

- **5-hour ceiling sizing.** Anthropic's subscription tier ceilings aren't exposed as rate-limit headers in the API responses Claude Code logs. The JSONLs don't carry `anthropic-ratelimit-input-tokens-remaining`. We need either (a) statusLine to expose absolute remaining-tokens not just percentage, or (b) one-time experimentation to derive the ceiling from observed burn rates and document it per subscription tier. Decision: defer to spec-implementation; treat % as canonical for the rule.
- **Cache-read vs raw input weighting.** Cache-read tokens are billed at ~10% of raw input. Whether they count 1:1 against the 5hr window is unclear. The spec logs all four counters so the rule can be re-tuned after observation.
- **Cross-window aggregation.** A user with three Claude Code windows open shares one subscription cap. statusLine is per-window. v1 of this RFC treats each session independently; v2 needs a shared-state design.
- **Integration with ccusage.** Optional or mandatory? Spec proposes optional (use if in PATH, fall back to inline parse). Better may be to mandate it once ccusage's API stabilizes.
- **OpenClaw substrate vs rig-local skill.** This RFC proposes substrate-level standardization, but the implementation could ship as a `superpowers`-style skill that any rig adopts. Trade-off: substrate gets every rig the protection by default; skill respects rig autonomy. Recommendation: ship as a skill first, propose substrate adoption after one quarter of field data.

## References

- `docs/postmortem-2026-06-10-fable5-burn.md` — full reconstruction of the 2026-06-10 incident.
- `docs/guardrails-spec.md` — detailed implementation spec.
- `docs/research-prior-art.md` — survey of LiteLLM, OpenTelemetry, Anthropic rate-limit headers, ccusage, Claude Code statusLine, LangChain agent budgets, and cost-circuit-breaker patterns.
- `feedback_fable5_cost_burn.md` (David's memory, updated 2026-06-11) — current policy memo on premium-tier opt-in.
