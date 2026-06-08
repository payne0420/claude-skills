---
name: track-usage
description: >-
  Read and track THIS Claude Code session's own usage — the 5-hour rate-limit
  percentage, the weekly (7-day) percentage, the context-window percentage, and
  when each one resets. The numbers are the same server-computed values Claude
  Code shows in the status bar, read on demand from a local snapshot file.

  Use when the user asks "how much usage is left", "what's my 5h / weekly
  usage", "when does my limit reset", "how long until reset", "am I near the
  cap", "track / monitor / check usage", or asks the agent to slow down, pause,
  or resume around a usage threshold (e.g. "pause at 90%"). ALSO use when YOU
  (the agent) are running a long or autonomous task and need to self-monitor
  usage so you can checkpoint before the limit and resume after the window
  resets.

  Triggers: "check my usage", "how much of my 5h/weekly limit", "when does my
  usage reset", "am I close to the limit", "pause at 90%", "track usage",
  "monitor rate limit".
---

# Track usage

The authoritative usage numbers (5h block, weekly/7-day, context window) are
written to a local snapshot file every time the status line renders. **Reading
that file is how this agent sees its own usage on demand** — the numbers are
*not* in the transcript and are *not* otherwise fed to the model.

## Standing directive — report after every turn

Once this skill is invoked, treat usage reporting as a **persistent behavior for
the rest of the session**: at the end of every assistant turn, read the snapshot
and append a single compact footer line as the last thing in your reply. Do this
on every turn, not just when asked — the user wants a continuous readout.

```bash
jq -r '"usage · 5h \(.five_hour.used_percentage)% · 7d \(.seven_day.used_percentage)% · ctx \(.ctx_pct)% · 5h resets ~\(((.five_hour.resets_at - now) / 60) | floor)m"' ~/.claude/usage-snapshot.json
```

If your `jq` lacks `now`, compute the reset delta in the shell instead:

```bash
P=$(jq -r '"5h \(.five_hour.used_percentage)% · 7d \(.seven_day.used_percentage)% · ctx \(.ctx_pct)%"' ~/.claude/usage-snapshot.json)
M=$(( ( $(jq -r '.five_hour.resets_at' ~/.claude/usage-snapshot.json) - $(date +%s) ) / 60 ))
printf 'usage · %s · 5h resets ~%dm\n' "$P" "$M"
```

Rules for the footer:
- One line, last in the reply, prefixed `usage ·`. Keep it terse — it's a status bar, not a section.
- If the snapshot is **stale** (`find ~/.claude/usage-snapshot.json -mmin -5` fails) or missing, say so in the footer rather than reporting numbers as current.
- **Want it truly automatic without me remembering?** A skill directive only holds while it's in context. For a guaranteed every-turn readout, wire a `Stop` hook in `~/.claude/settings.json` that prints the snapshot — ask and I'll set it up via the `update-config` skill.

## TL;DR — read it

```bash
jq -r '"5h: \(.five_hour.used_percentage)%  7d: \(.seven_day.used_percentage)%  ctx: \(.ctx_pct)%"' ~/.claude/usage-snapshot.json
```

Minutes until the 5-hour window resets:

```bash
echo $(( ( $(jq -r '.five_hour.resets_at' ~/.claude/usage-snapshot.json) - $(date +%s) ) / 60 ))
```

## What's in the snapshot

`~/.claude/usage-snapshot.json`:

```json
{
  "five_hour":  { "used_percentage": 12.3, "resets_at": 1780370000 },
  "seven_day":  { "used_percentage": 34.0, "resets_at": 1780900000 },
  "ctx_pct": 42.5
}
```

- `used_percentage` — 0–100, server-authoritative (what the status bar shows), **not an estimate**.
- `resets_at` — **epoch SECONDS** (compare against `date +%s`).
- `ctx_pct` — current context-window fill for this session.

## Why this works (the mechanism)

Claude Code pipes a JSON blob to the status-line command on **every render**
(`input=$(cat)` in `~/.claude/statusline-command.sh`). That blob contains
`.rate_limits.five_hour.used_percentage`, `.rate_limits.five_hour.resets_at`,
the `.seven_day` equivalents, `.context_window.used_percentage`, token counts,
and `.cost.total_cost_usd`. This data is **delivered only to the status-line
stdin** — it is never written to the session transcript, so the model cannot
read it unless something persists it. The status-line script therefore tees the
key fields to `~/.claude/usage-snapshot.json`, which the agent can `cat`/`jq`
any time.

## Freshness

The snapshot is only as fresh as the last status-line render. The status line
re-renders on session activity (each turn / tool event) in an **interactive**
session, so it stays current within a turn or two — fresh enough to catch a
threshold crossing at a phase checkpoint. Check it:

```bash
find ~/.claude/usage-snapshot.json -mmin -5 >/dev/null 2>&1 && echo fresh || echo "stale or missing"
```

A **headless / cron** run has no status line → the snapshot will not update.
In that mode, fall back to checkpoint-and-manual-resume rather than trusting a
stale value.

## Setup / repair (if the file is missing or empty)

Requires `jq` and a `statusLine` configured in `~/.claude/settings.json`. Ensure
this line exists in `~/.claude/statusline-command.sh`, immediately after
`input=$(cat)`:

```sh
echo "$input" | jq -c '{five_hour:.rate_limits.five_hour, seven_day:.rate_limits.seven_day, ctx_pct:.context_window.used_percentage}' > "$HOME/.claude/usage-snapshot.json" 2>/dev/null
```

After adding it, the file populates on the next render (any activity triggers
one). To revert tracking, delete that line.

## Self-pause / timed-resume around a threshold

For a long autonomous task the user wants paused near the cap:

1. **Poll at checkpoints**, not mid-generation — read the snapshot at each phase
   boundary or every few tool calls.
2. **Pause at the threshold** — if `five_hour.used_percentage >= 90` (or the
   user's number), write your checkpoint and stop the active work.
3. **Schedule resume after reset** — read `resets_at` (epoch seconds) and
   schedule a one-time resume at `resets_at + ~60s`. The reset can be hours out,
   **past `ScheduleWakeup`'s 1-hour clamp** — use `CronCreate` or the `schedule`
   skill for the long wait; use `ScheduleWakeup` only for sub-hour waits (or
   chain it).
4. **On resume**, reload your checkpoint and continue.

## Notes / gotchas

- Numbers are server-authoritative — don't recompute from token counts.
- `resets_at` is epoch **seconds**, not milliseconds.
- The transcript does **not** contain rate limits; this snapshot is the only
  on-demand path for the model.
- Human-facing equivalents (not model-readable): the `/usage` and `/context`
  slash commands in the TUI.
- The snapshot holds metrics only — never write user content to it.
