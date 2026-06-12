# Post-mortem — 2026-06-10 "Fable 5" Token Burn

**Author:** Claude (post-mortem agent)
**Date written:** 2026-06-11
**Incident date:** 2026-06-10
**Status:** Final
**Cost basis:** ~$50 in API credits + the 5hr Pro/Max bundled window exhausted in ~20 minutes (per David's recollection in `~/.claude/projects/-Users-davidsteeves/memory/feedback_fable5_cost_burn.md`)

---

## 1. Summary

On 2026-06-10, David's Claude Code rig burned through what he described as "the 5hr token window in ~20 min + an additional ~$50 of API credits" during a session where he was experimenting with promoting Claude Fable 5 to the default-driver tier in `~/.claude/settings.json`. He attributed the burn to Fable 5's higher per-token pricing and reverted.

The session logs tell a **different story** than the one captured in memory. The headline findings:

1. **Fable 5 never ran.** Every assistant turn on 2026-06-10 across all five active session JSONLs ran on `claude-opus-4-7`, not `claude-fable-5`. The settings.json flip and the Constitution §6 amendment landed during the session — but the in-process client retained the model handle it had booted with, and the new selection never took effect.
2. **The burn was a runaway cron loop, not a premium-model premium.** The single largest spike (10.1M tokens in 6 minutes, 22:34–22:40 UTC, session `f2d7a7ab`) was a scheduled task firing a stale "Sakaiai LLC formation" prompt every ~2 seconds. The model responded "Skipping." (output_tokens=9) on each fire, but each call still paid the full ~128K cache_read tab. 79 round-trips in 6 minutes.
3. **A second high-volume spike (22.9M tokens over 15:50–16:30 UTC)** was legitimate sakaiai LLC/EIN work in the same session, but the *same cron* was also firing 53 times during that window, inflating the bill further.
4. **Total 2026-06-10 spend across all five sessions: ~66.9M tokens** — 93% of which was cache_read. cache_creation alone was 4.4M, output was 264K. Real "thinking work" was a thin layer on top of an enormous re-played context.
5. **The "Fable 5 is expensive" mental model is wrong for this incident.** The cost was structural — a re-entrant cron + a session that never compacted between fires — not the per-token rate of any specific tier. Promoting Fable 5 to default would have made the same incident roughly 2× worse, but reverting Fable 5 does **not** prevent recurrence. The guardrail must target firing rate, not model selection.

The corrective Constitution rule David wrote ("Never default to Fable 5; premium tiers are opt-in per task") is still a sound principle but does not address the actual failure mode that caused the burn.

---

## 2. Timeline

Times are UTC. David's local time is PDT (UTC-7), so 2026-06-10 15:50 UTC = 08:50 PDT (morning) and 22:34 UTC = 15:34 PDT (afternoon).

### Active sessions on 2026-06-10 (after filtering by `.timestamp`)

| Session ID | First 2026-06-10 turn | Last 2026-06-10 turn | Turns w/ usage | 2026-06-10 total tokens |
|---|---|---|---|---|
| `1a8dc356-364f-4786-a16a-68d0b8f73fcb` | 00:03 | 16:30 | 143 | 17,962,658 |
| `f2d7a7ab-7912-4f51-b36a-368c4acd36d3` | 03:33 | 22:39 | 336 | 43,122,470 |
| `4097217a-4105-47da-a361-8e1f96499d38` | 16:20 | 16:34 | 8 | 817,498 |
| `3ed7c70a-1c86-4d85-9652-2f4f9f15b3ae` | 22:20 | 22:30 | 17 | 1,375,127 |
| `c0e2990c-04b4-4234-a29b-0c48aab00722` | 22:26 | 22:39 | 30 | 3,642,499 |
| **All five** | | | **534** | **~66.9M** |

Source: `jq -r 'select(.timestamp != null) | select(.timestamp | startswith("2026-06-10")) | select(.message.usage != null) | [.timestamp, .message.usage.input_tokens, .message.usage.cache_creation_input_tokens, .message.usage.cache_read_input_tokens, .message.usage.output_tokens] | @tsv'` over each file.

### Cross-session 10-minute buckets (combined, sorted by time)

| UTC bucket | Total tokens | input | cache_creation | cache_read | output | Note |
|---|---:|---:|---:|---:|---:|---|
| 00:00 | 3,582,330 | 56 | 293,788 | 3,272,343 | 16,143 | overnight cleanup |
| 00:10 | 1,636,191 | 12 | 144,102 | 1,482,476 | 9,601 | overnight cleanup |
| 03:30 | 1,440,696 | 20 | 291,129 | 1,146,226 | 3,321 | session start |
| 03:40 | 308,353 | 2 | 154,033 | 148,538 | 5,780 | |
| 15:00 | 777,993 | 10 | 465,693 | 309,717 | 2,573 | LLC formation work |
| 15:10 | 6,011,162 | 57 | 98,408 | 5,897,183 | 15,514 | LLC docs ramp |
| 15:30 | 369,649 | 14 | 189,860 | 177,832 | 1,943 | |
| 15:50 | 1,150,436 | 155 | 662,565 | 474,580 | 13,136 | EIN walkthrough |
| **16:00** | **8,132,688** | **9,561** | **326,332** | **7,740,556** | **56,239** | **afternoon spike begins** |
| **16:10** | **8,706,650** | **329** | **204,556** | **8,483,309** | **18,456** | **cron firing in parallel w/ EIN work** |
| **16:20** | **9,369,680** | **210** | **205,905** | **9,139,635** | **23,930** | **afternoon peak** |
| 16:30 | 4,568,009 | 21,671 | 221,603 | 4,299,887 | 24,848 | tapering |
| 22:20 | 8,025,661 | 16,458 | 614,001 | 7,348,070 | 47,132 | Fable 5 discussion + sub-agents |
| **22:30** | **12,840,754** | **4,419** | **536,948** | **12,273,852** | **25,535** | **runaway cron loop dominates** |

Two distinct spike windows, both >8M tokens / 10 min. The 22:30 bucket alone is **12.8M tokens — ~18% of total daily spend in a single 10-minute window.**

### Evening burn microscope (session `f2d7a7ab`, 22:34:58 → 22:39:13)

| Time (UTC) | Cum spend (session) | Event |
|---|---:|---|
| 22:34:51 | ~32.97M | Assistant finishes prior work; says "Saved. State at exit:". `output_tokens=249`, `cache_creation_input_tokens=127,808` (a fresh prompt cache). Intends to end. |
| **22:34:58** | **33.10M** | **Scheduled task fires** (`type=system, subtype=scheduled_task_fire`): enqueues a 121-token prompt asking Claude to "Check `davesteeves@gmail.com` for the UBI-issued email from WA Secretary of State…" |
| 22:35:00 | 33.23M | Assistant replies: `"Skipping."` — `input=6, cache_creation=1,014, cache_read=127,808, output=9`. |
| 22:35:02 | 33.36M | Cron fires again. Same prompt. Same reply. cache_read=128,822 (one previous reply now folded in). |
| 22:35:00 → 22:36:57 | **~33.10M → ~40.33M** | **~50 identical "Skipping." turns**, each costing ~130K cache_read, ~5K cache_creation accretion. **~7.2M tokens in 2 minutes.** |
| 22:38:00 | n/a | A `compact_boundary` system event fires — the conversation crosses an internal compaction threshold. cache_read resets from ~166K to ~51K. |
| 22:38:02 → 22:39:13 | ~40.5M → ~43.1M | **30 more "Skipping." turns** on the freshly compacted prompt. Loop continues. |
| 22:39:14 | 43.12M (final) | Session terminates (last record in file). David presumably killed the terminal. |

Total elapsed: **4 minutes 16 seconds.** Total burn during the loop alone: **~10.15M tokens.** Total `output_tokens` during the loop: **~951** (i.e. ~79 turns × ~12 tokens average — "Skipping." plus a few stray longer replies).

### Cron firing rates

| Session | Day-total `scheduled_task_fire` events | 2026-06-10 fires |
|---|---:|---:|
| `1a8dc356` | 0 | 0 |
| `f2d7a7ab` | 155 | **142** |
| `4097217a` | 0 | 0 |
| `3ed7c70a` | 0 | 0 |
| `c0e2990c` | 0 | 0 |

All 142 cron fires happened in **one session** — `f2d7a7ab` — and they were the same `MODEL_TIER_REVIEW_2026_06_20` / "Sakaiai LLC formation was filed" prompt repeatedly re-enqueued. 53 fires during the 15:50–16:30 afternoon window; 89 fires after 22:00.

---

## 3. Root cause analysis

### 3.1 Why was Fable 5 blamed?

The memory file `feedback_fable5_cost_burn.md` says: *"2026-06-10 Fable-5-as-driver change burned 5hr window in 20 min + ~$50; reverted; premium tiers are opt-in per task."*

The session evidence contradicts this:

- **`message.model` for every assistant turn on 2026-06-10 across all 5 session files = `claude-opus-4-7`.** Not `claude-fable-5`. Not even once.
- Session `c0e2990c` (22:26–22:39 UTC) contains the actual settings/Constitution change: David asks "Update Claude Code CLI… Fable 5… is there a new Claude model?" and the assistant writes the §6 amendment + edits `~/.claude/settings.json`. The session also enqueues a deferred-review cron (`MODEL_TIER_REVIEW_2026_06_20`). But the model serving that very session is `claude-opus-4-7`.
- Claude Code 2.1.101 reads `settings.json` at startup. A mid-session edit to the `model` field does **not** hot-reload the running client. Any sessions David already had open kept using whatever model was set when they booted.

So the "Fable 5 burn" framing is at best partially correct (David did flip the default, which would have raised future cost) and at worst a red herring — the actual burn ran on Opus 4.7 at its standard $15/$75 per Mtok rate.

If Fable 5 *had* been live for the runaway loop at its $10/$50 per Mtok output rate but with the 90/10 cache-read split on Anthropic's standard caching pricing (cache_read at 10% of input), the same 10.1M-token loop would have cost roughly:

```
cache_read 9.89M × $1.00/Mtok (10% of $10 input) = $9.89
cache_creation 0.26M × $12.50/Mtok (125% of input, 5-min ephemeral) = $3.25
input 469 × $10.00/Mtok = ~$0.00
output 951 × $50.00/Mtok = ~$0.05
≈ $13.19 for the 6-minute loop
```

On Opus 4.7 ($15/$75 base, 10%/125% cache modifiers):
```
cache_read 9.89M × $1.50/Mtok = $14.83
cache_creation 0.26M × $18.75/Mtok = $4.88
input 469 × $15/Mtok ≈ $0.01
output 951 × $75/Mtok ≈ $0.07
≈ $19.79 for the 6-minute loop
```

i.e. the loop on Opus 4.7 was actually **more expensive in pure $ terms** than the same loop would have been on Fable 5. The "Fable 5 is 2× the cost" framing held in the abstract, but in this specific incident swapping models wouldn't have helped.

**The cost model David should internalize: it's not the per-token rate that hurts — it's the firing rate × prompt-cache size.**

### 3.2 The real root cause: re-entrant cron + Stop-hook + non-blocking reply

Reconstructed from `f2d7a7ab` records between 22:34:58 and 22:39:14:

1. **A scheduled task was registered earlier in the day** with a prompt of the form "Sakaiai LLC formation was filed with WA SOS on 2026-06-08 at 10:36 AM PT… Check `davesteeves@gmail.com`…". Evidence: the prompt's text already mentions the EIN-related kanban update, which suggests it was written during the morning EIN session and scheduled to recheck for the UBI email.

2. **The scheduling system fired this task repeatedly with no exponential back-off and no de-duplication.** First fire 16:08:04, second 16:10:27, third 16:11:50 — roughly 2-minute intervals. Then a burst at 16:14–16:18 with **~2-5 second** intervals.

3. **Each fire enqueued the same prompt via `type=queue-operation, operation=enqueue`** (confirmed in 1222 queue-operation records in this session — though most are unrelated routine ops; the cron-fired enqueues all have identical content).

4. **The assistant's reply was always trivially short** ("Skipping." for 79 of the 22:34–22:39 turns). The model recognized the prompt was a duplicate, but the API turn still happened — including paying for the full prompt cache replay (~128K tokens) plus any incremental cache_creation.

5. **The Stop hook fired after each turn**, attempting to run `~/.claude/memory/bin/stop-log.sh` plus a plugin-warp hook that failed (`Plugin directory does not exist: …/claude-code-warp/warp/2.0.0`). The failure was non-blocking, so the loop continued.

6. **`UserPromptSubmit` hook on each cron fire re-injected the kanban context** as `hook_additional_context` attachment — that's the source of the ~1KB-per-turn cache_creation growth (see how `cache_creation_input_tokens` ticked up 764→925→764→925 in a regular pattern — alternating between hook-context and a slightly larger variant).

7. **At 22:38:00 the conversation hit `compact_boundary`** — Claude Code's internal compaction kicked in. Cache_read reset from 166K to 51K. The loop continued unaffected, just with a freshly built prompt.

**Why didn't the model recognize and stop the loop?** It tried — the "Skipping." reply IS the model's recognition. But the harness has no "if assistant says X then suppress next cron fire" rule. The cron and the model are decoupled. Anthropic's API doesn't see scheduled-task firing as a meaningful event; from its perspective, each call is just another user turn.

### 3.3 Contributing factor: long-running session with growing prompt cache

By the time the evening loop started at 22:34, session `f2d7a7ab` had been running since 2026-05-31 (10 days). Its cache prefix had accumulated to ~128K tokens — every system prompt change, every additional CLAUDE.md sub-file injection, every kanban update on the prompt. The compact at 22:38 dropped this to 51K. Even after compaction, every "Skipping." cycle still costs 51K cache_read.

**A short-running session would have made the same loop ~2.5× cheaper** (51K vs 128K per cycle). The fact that David never restarted the terminal between the morning EIN work and the evening Fable 5 discussion meant the prompt cache was already at its expanded ceiling when the loop began.

### 3.4 Contributing factor: morning prompt-cache inflation under live agent fan-out

The morning 15:50–16:30 spike (~28M tokens) was *partially* legitimate work — the `tryout-app-admin-jesse-onboarding` team was running Stripe Connect spec work in `1a8dc356`, with multiple sub-agents touching files, reading specs, and writing code. cache_read per turn grew from ~94K (15:59) → ~115K (16:09) → ~129K (16:08) as the system prompt + tool-result attachments accreted. This is normal sub-agent inflation, not a bug.

But the *same* cron was firing in `f2d7a7ab` during this same window, inflating that session's spend by 22.9M tokens of which the cron's contribution is hard to disentangle from real LLC work. Conservatively: if 53 cron fires × ~100K cache_read average ≈ **5.3M tokens** were pure cron waste during the afternoon, in addition to the 10.1M evening burn. **Total estimated cron waste: ~15.4M tokens out of ~66.9M (23% of the day's spend).**

---

## 4. Contributing factors (consolidated)

| Factor | Evidence | Magnitude |
|---|---|---|
| **Cron re-fires without back-off or de-dupe** | 142 fires of the same prompt in one session | ~15.4M wasted tokens (~23% of day) |
| **Stop hook didn't suppress next fire** | Loop continued after each `stop_hook_summary` | enabled the runaway |
| **No cache eviction between fires** | cache_read = 127,808 → 166,299 then resets at compact, climbs again | each cycle pays full prompt-cache replay |
| **No firing-rate ceiling in the harness** | 79 turns in 6 minutes = ~13 turns/min | rate would have triggered any sane circuit-breaker |
| **In-session `settings.json.model` change does not take effect** | All turns 2026-06-10 ran on `claude-opus-4-7` despite Constitution flip | masked the actual root cause in postmortem-1 |
| **Old long-running session (10 days)** | `f2d7a7ab` first record 2026-05-31 | inflated baseline cache to 128K vs ~50K fresh |
| **`claude-code-warp` plugin error firing on every Stop** | `"Plugin directory does not exist: …/claude-code-warp/warp/2.0.0"` 50+ times | not the cause; pure log noise but indicates hooks not cleanly tested |
| **No spend telemetry visible to David in-session** | David didn't notice until "after $50 of API credits" | nothing surfaced the burn |
| **`feedback_fable5_cost_burn.md` mis-attributes cause** | Memory says "Fable-5-as-driver change burned"; logs say all turns were Opus 4.7 | locks in a wrong remediation pattern (downgrade tier) instead of fixing cron |

---

## 5. Recommendations for guardrails

The kanban row called for "early spike detection" with a rule of thumb: flag if >15% of the 5hr window is consumed within any 10-min span. **The 22:30 bucket alone consumed 12.8M tokens — that's the upper bound benchmark to design against.** Apply 15% of whatever the 5hr ceiling is and compare.

Suggested signals, ranked by leverage:

### 5.1 Same-prompt-fired-twice detector (highest leverage, simplest)

Track a rolling 60-second hash window of all enqueued prompts. If the same prompt-hash fires ≥3 times within 60 seconds, **block further enqueues of that hash for 5 minutes** and post a system-notification: `"Cron MODEL_TIER_REVIEW_2026_06_20 fired 3 times in 60s — suppressing. Investigate scheduled-task config."` This would have caught the 22:35:00–22:35:11 burst within ~6 seconds, saving 9.5M of the 10.1M-token loop.

### 5.2 Per-turn output-to-input ratio alarm

If `output_tokens < 50` AND `cache_read_input_tokens > 50,000` AND **the previous 5 turns had the same shape**, fire a `LOW_VALUE_TURN_SPIRAL` alarm. The 22:35:00 loop turns averaged output=9, cache_read=130K — output/input ratio ≈ 0.00007. That's a clear "wheel spinning" signature.

### 5.3 10-minute rolling window vs 5hr ceiling

The kanban's rule of thumb. Pseudo-code:

```
window_start = now - 10 minutes
tokens_in_window = sum(turn.input + turn.cache_creation + turn.cache_read + turn.output for turn in turns where turn.ts >= window_start)
five_hr_ceiling = <plan-specific; e.g. ~80M for Pro/Max Opus tier or ~40M for Pro>

if tokens_in_window > 0.15 * five_hr_ceiling:
    emit_alarm("TOKEN_SPIKE: {tokens_in_window/1e6:.1f}M tokens in 10 min — {percent:.0f}% of 5hr ceiling")
```

For this incident, the 22:30 bucket would have crossed any reasonable threshold (12.8M / 80M = 16%). The 16:20 bucket (9.4M / 80M = 11.7%) would have been a warning, not an alarm — appropriate, because that 11.7% was a mix of legitimate work and cron waste, not pure waste.

### 5.4 Cron-fire global ceiling

Per session, allow at most **N scheduled-task fires per hour** (e.g. N=12, one every 5 min). On exceeding, pause the scheduler and require David to manually unpause. The `f2d7a7ab` session would have hit the ceiling within ~10 minutes of the 16:08 burst and again in ~10 minutes of the 22:34 burst, suppressing both.

### 5.5 Stale-prompt detector

When a scheduled task fires, compare its prompt-content against the assistant's last 5 replies. If the topic of the prompt has already been resolved (e.g. assistant said "Saved. State at exit:" in the prior turn referencing the same kanban item), suppress the fire. This is more heuristic but catches the specific pattern of "session ended, cron didn't get the memo".

### 5.6 Settings-change recompute

When `~/.claude/settings.json` `model` field changes mid-session, either (a) prompt the user "Model change detected — restart session to apply?" or (b) actually hot-swap. The current silent-no-op behavior is what caused the post-mortem confusion. **Logging the effective `model` per session at startup and on every API call would have made this debuggable on the spot.**

### 5.7 Cost-burn surface

Display a header line in the terminal status bar after every assistant turn:

```
[Session: 43.1M tok | Last 10min: 12.8M (16% of 5hr ceiling)  ← ALARM]
```

David's complaint was that he didn't notice. Surface the data and the human eye catches what the harness's logic misses.

### 5.8 Long-running-session expiry

Hard-cap session age at 24h or trigger a forced compact at 12h. A 10-day-old session with 128K of cache_read baseline is a structural cost hazard. The "Restart this session to clear the cron" hint that the model itself emitted at 22:34:51 is the right idea — make it enforceable, not advisory.

### 5.9 Update the memory note

`~/.claude/projects/-Users-davidsteeves/memory/feedback_fable5_cost_burn.md` should be amended:

- Remove the framing "Fable-5-as-driver change burned the 5hr window."
- Replace with "2026-06-10 runaway cron-fire loop in session `f2d7a7ab` burned ~10M tokens in 6 minutes by re-firing a stale scheduled task while the assistant replied 'Skipping.' Each fire paid ~128K cache_read. Root cause: no firing-rate cap, no prompt-hash dedupe."
- Keep the principle "premium tiers are opt-in per task" as a separate concern — sound advice, just not what caused this incident.

---

## 6. Open questions

1. **What does the 5hr Pro/Max window's ceiling actually look like in tokens?** This post-mortem assumed a ballpark of ~80M for Opus tier based on Anthropic's published guidance, but the exact number depends on plan, tier, and any internal rate-limiting Anthropic applies between the rolling 5hr window and the per-minute limit. Without knowing the actual ceiling, "15% in 10 min" is hard to operationalize. **Recommend:** capture the rate-limit response headers (e.g. `anthropic-ratelimit-tokens-remaining`) from the Anthropic API responses going forward, and store them in session logs — they reveal the ceiling directly. None of the 2026-06-10 records include those headers.

2. **Was the cron set up by Claude or by David?** Session `c0e2990c` shows the assistant scheduling `MODEL_TIER_REVIEW_2026_06_20` for 2026-06-20 09:07 local. But the cron that actually fired the burn was a **different** prompt ("Sakaiai LLC formation was filed with WA SOS on 2026-06-08…") — set up in some earlier session (likely the morning EIN session). Recommend grep `scheduled_task_fire` in the JSONLs going back further to find when that cron was originally registered and whether it had a `repeat` flag that should have been `once`.

3. **Why did the cron re-fire every 2 seconds instead of on its scheduled cadence?** The originally scheduled interval is unknown from the logs alone. The 2-second cadence is suspiciously close to the API turn-around time, which suggests the scheduler might have been **firing on every Stop event** instead of on a wall-clock cron expression. If true, that's a scheduling-system bug and the fix is to enforce minimum interval between fires regardless of trigger.

4. **Did the API ever return a 429 or `anthropic-ratelimit-*` warning?** None visible in the 3 `api_error` records in `f2d7a7ab` (those were 2026-06-01). The model kept returning successful turns through the entire 79-turn loop. This is consistent with the 5hr-window being a separate ceiling from per-minute rate limits, but worth confirming.

5. **What's the actual Claude Code CLI behavior on `settings.json` model field change?** The 2.1.101 client clearly didn't pick up the live edit — but is this documented, or is it accidental? If accidental, file an Anthropic bug. If documented, surface it in `~/.claude/CLAUDE.md` so future post-mortems don't get derailed.

6. **Why does the JSONL not record the API `usage` headers from rate-limit responses?** Knowing the rate-limit remainder per response would have made spike detection trivially deterministic. Investigate whether Claude Code's logger can be extended to capture response headers.

7. **Cost accounting: is cache_read counted toward the 5hr window 1:1, or at a discount?** The math throughout this doc assumed 1:1 because that matches David's billing experience, but Anthropic's caching pricing applies a 10% discount to cache_read in raw $ — whether the 5hr window's tokens-in-flight ceiling discounts equivalently is unclear from public docs.

8. **Did the 22:38:00 `compact_boundary` actually save tokens?** Cache_read dropped from 166K to 51K — but the rebuild of the new cache at 22:38:02 cost cache_creation=51,310. The compact is net-negative in the first turn after, then net-positive once the cache amortizes. Need to verify whether the compaction logic is aware of cost when deciding when to fire.

---

## Appendix A: Reproducible queries

All findings come from the five files in `/Users/davidsteeves/.claude/projects/-Users-davidsteeves/`:

```
1a8dc356-364f-4786-a16a-68d0b8f73fcb.jsonl   (32 MB, 18166 lines)
f2d7a7ab-7912-4f51-b36a-368c4acd36d3.jsonl   (31 MB, 12547 lines)
4097217a-4105-47da-a361-8e1f96499d38.jsonl   (4.8 MB, 2474 lines)
3ed7c70a-1c86-4d85-9652-2f4f9f15b3ae.jsonl   (173 KB, 115 lines)
c0e2990c-04b4-4234-a29b-0c48aab00722.jsonl   (374 KB, 153 lines)
```

Per-session per-day total spend:
```sh
jq -r 'select(.timestamp != null) | select(.timestamp | startswith("2026-06-10")) | select(.message.usage != null) | [.message.usage.input_tokens, .message.usage.cache_creation_input_tokens, .message.usage.cache_read_input_tokens, .message.usage.output_tokens] | @tsv' SESSION.jsonl \
  | awk 'BEGIN{i=0;cc=0;cr=0;o=0} {i+=$1;cc+=$2;cr+=$3;o+=$4} END{printf "input=%d cache_creation=%d cache_read=%d output=%d total=%d\n", i, cc, cr, o, i+cc+cr+o}'
```

10-minute buckets:
```sh
jq -r 'select(.timestamp != null) | select(.timestamp | startswith("2026-06-10")) | select(.message.usage != null) | [.timestamp, .message.usage.input_tokens, .message.usage.cache_creation_input_tokens, .message.usage.cache_read_input_tokens, .message.usage.output_tokens] | @tsv' SESSION.jsonl \
  | awk '{tot=$2+$3+$4+$5; bucket=substr($1,1,15)"0"; a[bucket]+=tot; ai[bucket]+=$2; ac[bucket]+=$3; ar[bucket]+=$4; ao[bucket]+=$5} END {for (k in a) printf "%s tot=%d in=%d cc=%d cr=%d out=%d\n", k, a[k], ai[k], ac[k], ar[k], ao[k]}' | sort
```

Identify cron fires:
```sh
jq -r 'select(.type=="system" and .subtype=="scheduled_task_fire") | .timestamp' SESSION.jsonl
```

Identify enqueued prompts:
```sh
jq -r 'select(.type=="queue-operation" and .operation=="enqueue") | "\(.timestamp) \(.content[0:120])"' SESSION.jsonl
```

Model used per turn:
```sh
jq -r 'select(.message.model != null) | .message.model' SESSION.jsonl | sort -u
```

Per-turn detail in the 22:34–22:40 runaway loop:
```sh
jq -c 'select(.timestamp >= "2026-06-10T22:34:50" and .timestamp < "2026-06-10T22:39:20") | select(.message.usage != null) | {ts: .timestamp, content: [.message.content[]?.text // ""][0][0:60], usage: .message.usage}' f2d7a7ab-7912-4f51-b36a-368c4acd36d3.jsonl
```

---

## Appendix B: What did NOT cause the burn

For the record, to head off mis-attribution in future post-mortems:

- **Not Fable 5.** No `claude-fable-5` ever ran. All turns: `claude-opus-4-7`.
- **Not agent fan-out.** `isSidechain=true` was rare in the burn window. The runaway loop was the main-driver assistant replying directly. (Sidechain turns appear in 1a8dc356 morning Stripe work but that was legitimate workload.)
- **Not a runaway sub-agent.** No nested TaskCreate/Agent calls during the loop.
- **Not a giant file read.** The largest `cache_read` per turn was ~167K, not the multi-million-token reads you'd see from a 32MB file being slurped.
- **Not extended thinking.** Opus 4.7 doesn't have extended thinking on by default, and the loop turns showed no `thinking` content blocks for the 79-turn loop window — just `text` blocks with "Skipping."
- **Not the kanban context auto-inject.** That's ~1KB per turn (cache_creation grew by ~764-925 tokens per fire). Real but a small contributor; killing it would save maybe 80K total in the loop.
- **Not user input.** David didn't type anything during the runaway loop. The user prompts were all `type=user` records auto-generated by the cron-enqueued task (with `isMeta:true` marking them as harness-injected, not human).

---

*End of post-mortem.*
