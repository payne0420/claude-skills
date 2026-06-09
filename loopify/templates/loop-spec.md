# Loop: <name>

Status: draft | dry-run-passed | scheduled | paused | killed
Created: YYYY-MM-DD · Last audited: YYYY-MM-DD

## Mission

One paragraph: what this loop finds, what it does with it, and what it hands
off to humans. If this paragraph needs a second paragraph, the loop is too big.

## Qualification (passed YYYY-MM-DD)

- **Recurs:** <cadence + evidence it actually repeats>
- **Gate:** `<command>` (exit 0 = pass) — proven able to FAIL bad work on <date>
- **Budget:** <cap per run / per week, and what plan absorbs it>
- **Tools:** <logs / repro env / run access available to the executor>

## Trigger

<one of: `/loop <interval>` (session) · cron `<expr>` via CronCreate (local,
survives restart) · routine via /schedule (cloud, laptop-off) · hook on <event>>

## One unit of work

What a single run does, bounded. Find work via <source — CI failures, lint
output, issue label, …>; take at most <N> items per run; leave the rest for the
next run.

## Gate — the only definition of done

- Command: `<cmd>` — exit 0 required before any commit/PR is proposed.
- Independent check (optional, supplements — never replaces — the command):
  <fresh judge via `claude -p`, or an external reviewer agent>

## Hard stops

- Per run: max <N> iterations / <T> minutes / <X> tokens.
- Stop on success: the gate passes.
- Stop on failure: after <K> consecutive failed runs, escalate and stop spending.

## Scope fences — never

- Touch <paths — e.g. src/payments/, auth, CI config>
- Disable, skip, or weaken failing tests — escalate instead
- Merge, deploy, or change dependencies without human approval
- Edit this file, the gate, or its own permissions

## Escalation

When blocked or failed: append the item with evidence to `STATE.md → ##
Escalated to humans`; notify via <channel — PushNotification / Slack / Linear /
none>.

## Executor

Engine-agnostic — pick per run. Default: <Claude in-session / routine /
`codex exec` / `cursor-agent` / `opencode run`>.
