# Spike-Detection Guardrail — Specification

**Kanban:** OpenClaw: token-spike guardrails (k_5eca5366)
**Date:** 2026-06-11
**Status:** Draft v1

## Problem

On 2026-06-10 the Claude Code rig burned its 5-hour token window in ~20 min and overran by ~$50 of API credits. **The original diagnosis ("default-model swap to Fable-5") was wrong** — the 2026-06-11 post-mortem (`postmortem-2026-06-10-fable5-burn.md`) reconstructed all session JSONLs and found Fable-5 never ran. The real cause was a **runaway scheduled-task cron** that fired the same prompt 142× in one session, including a peak burst of 79 fires in 4 minutes (~10.1M tokens / 6 min, ~93% cache_read). The user couldn't see the trajectory until the window was already gone.

This reframes the guardrail design: the spike-velocity rule is necessary but not sufficient. We also need to detect *the specific failure shape that caused this burn* — a scheduler firing the same prompt repeatedly. Both rules are specified below.

## Goal

Detect spend velocity that puts the 5-hour window at risk *before* exhaustion. Make the warning loud enough to interrupt the user. Fail-safe by default (warn, don't silently allow), but never block work the user explicitly wants done.

## Non-goals

- Per-tool or per-agent budgets (v2).
- Multi-tier budget stacks (v2).
- Cross-session aggregation (v2 — for now, treat each Claude Code session as the unit).
- Replacing ccusage / Anthropic billing dashboards (those are the source of truth for after-the-fact accounting).

## The rule

> **Burn-rate trip:** If the 5-hour window's `used_percentage` increased by more than **15 percentage points within any 10-minute rolling span** during the current window, the guardrail trips.
>
> **Premium-model confirm:** Before any prompt is sent against a model in the *premium tier* (`claude-fable-5`, `claude-opus-4-8`, `claude-opus-4-7` while ultracode is on), the guardrail requires explicit per-session opt-in.

### Why 15% / 10 min

At baseline, sustained 100% utilization of a 5-hour window = ~3.33% / 10 min. A burn rate of 15% / 10 min is **4.5× sustained**. That is high enough to catch the Fable-5 runaway (which hit ~75% / 10 min during the 2026-06-10 incident — assumption to verify with the post-mortem) without false-positiving normal heavy use (a parallel-agent fan-out for an audit may briefly push 8–10% / 10 min).

### Why % of window, not absolute tokens

Subscription-tier 5-hour windows are opaque token caps that vary by plan and Anthropic's internal accounting (cache vs raw, model surcharges). `rate_limits.five_hour.used_percentage` from Claude Code's statusLine JSON is the canonical signal — it already does the bookkeeping correctly.

## Trip behavior — three modes

| Trip mode | Behavior | When chosen |
|-----------|----------|-------------|
| **`warn`** *(default)* | Status bar reads `BURN 18% /10m ⚠`. UserPromptSubmit hook injects `<spike-warning>` into `additionalContext`. No block. | First trip in a window, OR after explicit acknowledgement. |
| **`degrade`** | Same warning, plus prepend `MODEL=sonnet` policy advisory: "Driver should fall back to Sonnet for the next turn unless judgment-heavy." | Two consecutive trips, OR `used_percentage > 60` and a trip. |
| **`block`** | UserPromptSubmit hook prints a stdout block message and exits non-zero. User must `touch ~/.claude/state/spike-unblock` or restart the session. | `used_percentage > 85` and a trip, OR explicit user setting `SPIKE_MODE=block`. |

Trip mode escalates one step per repeat within the same window; never auto-de-escalates.

## Cron-loop detector (post-mortem-driven, NEW)

The 2026-06-10 burn was a scheduled-task cron firing the same prompt 142× in one session. The spike-velocity rule above would have caught it (the 22:34 burst was ~16% / 6 min, above the 15% / 10 min threshold), but only after ~5M tokens had already been spent. A *prompt-shape* rule catches it on the **second** fire, not the seventy-ninth.

### Rule

> **Same-prompt-hash dedupe:** A `UserPromptSubmit` whose SHA-256 of the normalized prompt body matches a prompt sent in the same session within the last 60 seconds is suppressed. The hook exits non-zero with a one-line message: `cron-loop guard: duplicate prompt suppressed (hash=<8 chars>)`.
>
> **Output-spiral detector:** Across the last N=5 turns in the session, if `output_tokens < 50` AND `cache_read_tokens > 50000` for every turn, the next prompt is blocked with: `cron-loop guard: 5 consecutive zero-output high-cache-read turns suggest a runaway scheduled task. Pause the cron and edit ~/.claude/state/spike-unblock to resume.`

Normalized prompt = `re.sub(r'\s+', ' ', prompt).strip().lower()`. Ignores leading/trailing whitespace and case variations. Hash is per-session, not global — different sessions can re-use the same prompt without colliding.

### Why two rules

The dedupe rule is the cheap, deterministic catch. The output-spiral rule catches cases where a legitimate cron fires varied prompts but the model is bored to tears (e.g. "check status" cron where nothing has changed for hours).

### Per-session cron-fire ceiling

A separate static limit: any individual `CronCreate`-registered prompt can fire at most **48 times per session** (once-every-15-min for 12 hours). Tracked in `~/.claude/memory/state/cron-fires.<session_id>.json`. On 49th fire, the hook blocks with `cron-fire ceiling exceeded for prompt hash <h>. This usually means the schedule is too aggressive or never converges. Disable the cron via /scheduled-tasks delete.`

Ceiling is per-prompt-hash, not global. A session can register many distinct crons without hitting the limit; only one runaway hits 48.

## Session-age cap

A separate finding from the post-mortem: the burn session was **10 days old**. Long-lived sessions accumulate stale context, register crons that no longer make sense, and drift from any settings.json changes (the model field doesn't hot-reload).

> **Hard rule:** sessions older than 24 hours are flagged at every `UserPromptSubmit` with `<session-age-warning>session is N days old — consider /clear or starting fresh</session-age-warning>` injected into `additionalContext`. After 7 days the warning escalates to a `block` (matching the spike rule's behavior).

This is not a token-spend rule per se, but it prevents the conditions under which spend rules become necessary.

## Settings hot-swap detector

Final finding from the post-mortem: changing `model` in `~/.claude/settings.json` did NOT propagate to running sessions in Claude Code 2.1.101. David thought he had moved to Fable-5; in reality he was still on Opus 4.7. Either outcome is fine *if visible*; the failure mode is silent divergence.

> **Rule:** statusLine reads `claude_settings_model` from the settings file on each tick AND `model` from the JSON contract. If they disagree, the status bar shows `MODEL≠SETTINGS <session>=<a> <file>=<b>` until the session is restarted.

This is informational only; no block. The point is to make the divergence visible the moment a user edits settings expecting it to take effect.

## Premium-model confirm

A second, narrower rule that targets the *specific failure mode* that caused 2026-06-10: silently making a premium model the default driver.

Trigger: before a prompt is sent, the hook reads the active model. If model is in the premium-tier list AND the session has not yet recorded explicit opt-in for that model, the hook:

1. Writes a sentinel: `~/.claude/state/premium-pending.<session_id>.json`
2. Returns a stdout message: `Premium model <name> requires opt-in. Reply with /opt-in-premium <name> or change driver.`
3. Returns exit code 1 (blocks the prompt).

Opt-in is per-session and per-model. Recorded in `~/.claude/state/premium-optin.<session_id>.json`. Auto-cleared on session end.

The literal `/opt-in-premium <model>` is processed by another hook or a slash-command skill that writes the opt-in file and lets the next prompt through.

This rule is independent of the burn-rate rule and runs at the same hook (UserPromptSubmit).

## Where the signals come from

| Signal | Source | Cadence |
|--------|--------|---------|
| `five_hour.used_percentage` | Claude Code statusLine JSON (stdin to `statusline-command.sh`) | every status tick (~5s) |
| `model` | Claude Code statusLine JSON | every status tick |
| `total_cost_usd`, `total_input_tokens`, `total_output_tokens`, `cache_read_tokens`, `cache_creation_tokens` | Claude Code statusLine JSON | every status tick |
| Per-turn cumulative spend | Inferred from statusLine cumulative counters (delta between ticks) | every status tick |

ccusage's 5-hour block math is the fallback when statusLine doesn't surface what we need; the spec assumes statusLine is sufficient and only references ccusage as a sanity check.

## Where state lives

```
~/.claude/memory/state/
  spike.jsonl                            # append-only ring buffer of statusline ticks
  spike.lock                             # writer mutex for statusline (avoid concurrent corruption)
  trip.<session_id>.json                 # current trip mode for the session
  spike-unblock                          # touch this to release a block trip

~/.claude/state/
  premium-optin.<session_id>.json        # which premium models are opted-in for this session
  premium-pending.<session_id>.json      # sentinel for "awaiting opt-in"
```

`spike.jsonl` is bounded — rotated every 1 MB or 24h, whichever first. Format:

```json
{"ts": "2026-06-11T13:42:01Z", "session": "627cb8a0", "model": "claude-opus-4-7", "pct": 12.4, "cum_in": 425112, "cum_out": 38211, "cache_read": 1882000, "cache_create": 412000}
```

## Components

### 1. `~/.claude/memory/bin/spike-log.sh`

Called from `statusline-command.sh`. Reads the statusLine JSON on stdin, appends a row to `spike.jsonl`, prints the existing status bar text on stdout (transparent passthrough). Fails open: any error logs to `spike-log.err` and returns the unmodified stdin.

### 2. `~/.claude/memory/bin/spike-check.sh`

Called from `UserPromptSubmit` hook (after the existing `prompt-context.sh`). Reads the last 10 min of `spike.jsonl` for the active session, computes `Δpct`. If `Δpct > 15`:
- Updates `trip.<session_id>.json` (mode escalation: `warn` → `degrade` → `block`).
- Per mode, takes the corresponding action (inject warning, advisory, or block).

Also checks premium-model opt-in if applicable.

### 3. `statusline-command.sh` integration

One-line addition near the start: pipe stdin through `spike-log.sh` (which prints unmodified stdout) before the existing status logic. The existing statusline keeps working; the spike ledger is a side effect.

### 4. `settings.json` hook order

```json
"UserPromptSubmit": [{
  "hooks": [
    {"type": "command", "command": "~/.claude/memory/bin/prompt-context.sh"},
    {"type": "command", "command": "~/.claude/memory/bin/spike-check.sh"}
  ]
}]
```

`spike-check.sh` runs *after* `prompt-context.sh` so it can be skipped if context injection failed; both must succeed for the prompt to proceed in block mode.

## Implementation budget

| Component | LOC est. | Risk |
|-----------|----------|------|
| `spike-log.sh` | ~25 | low — read JSON, append line |
| `spike-check.sh` | ~80 | medium — JSONL tail parse, time math, mode state |
| `cron-loop-check.sh` (dedupe + spiral + ceiling) | ~80 | medium — prompt hashing, per-session state |
| Premium opt-in slash-command/skill | ~40 | low |
| Session-age + settings-divergence checks | ~30 | low |
| `statusline-command.sh` patch | 2 lines | low |
| `settings.json` hook patch | 1 line | low |
| Tests (golden-file ledger fixtures) | ~80 | low |
| **Total** | **~340 LOC** | |

## Failure modes & mitigations

| Failure | Mitigation |
|---------|------------|
| `spike.jsonl` grows unbounded | Rotate at 1MB/24h. `spike-log.sh` checks size on every write. |
| Statusline JSON schema changes | Default to fail-open: log warning, skip this tick. Never block a prompt because the ledger is unreadable. |
| Concurrent statusline writes (multiple sessions) | `flock(spike.lock)`. Statusline runs at ~5s cadence so contention is rare. |
| Cross-session pollution | Filter `spike.jsonl` rows by `session_id` in `spike-check.sh`. |
| User wants to legitimately burn the window (deep research, audit) | `block` trip is opt-out via `spike-unblock` file. `warn` and `degrade` never block. |
| Hook script crashes mid-prompt | Exit 0 on any internal error; print warning to stderr. Never let a guardrail script become the reason a prompt is lost. |
| Premium-model rule blocks at session start before any model context exists | First prompt of a session is always allowed regardless. Rule only fires once a model is recorded in statusLine state. |

## Testing strategy

1. **Golden fixtures.** Hand-craft 5 `spike.jsonl` files representing: (a) idle session, (b) steady use, (c) slow burn, (d) Fable-5-style runaway, (e) window-exhausted. `spike-check.sh` is a pure function of `(now, jsonl, session_id)` → `{mode, message}`; assert outputs.
2. **Hook integration.** Mock UserPromptSubmit invocation with stdin JSON; assert injection text matches.
3. **Premium opt-in flow.** Cold session → premium model → block → opt-in slash command → next prompt clears.
4. **Statusline regression.** With `spike-log.sh` patched in, the existing status bar still renders identically.

## Open questions

- **Cache-read vs raw input accounting:** Anthropic's 5hr cap weighs cache-read tokens differently than uncached input. The statusLine `used_percentage` already does this math, but should the spike ledger preserve the components for post-mortem reconstruction? *Recommended: yes, log all four counters.*
- **5-hour block reset:** when the window resets mid-session, does the statusLine emit `used_percentage = 0` or does it lag? *Need to verify experimentally — log it.*
- **Multiple Claude Code windows open simultaneously:** statusLine is per-window. The 5hr cap is per-account. Cross-window aggregation deferred to v2.
- **Whether to integrate with `ccusage statusline`** rather than parsing JSON ourselves. ccusage gives nice 5hr-block reporting "for free" but adds a dependency. *Recommended: optional integration — if `ccusage` is in PATH, use its JSON; else fall back to inline parse.*

## Rollout

1. Land `spike-log.sh` + statusLine patch first (data collection only, no enforcement). Run for 48h to validate the ledger captures what we expect.
2. Add `spike-check.sh` in `warn` mode only. Run for 48h. Tune the 15% / 10min threshold if false-positive rate is wrong.
3. Enable `degrade` mode. Hold for one week.
4. Enable `block` mode at >85% window. Document the unblock procedure in CLAUDE.md.
5. Add premium-model opt-in. Document the slash command.

Each phase is reversible by reverting one file (`spike-check.sh` or `settings.json` block).
