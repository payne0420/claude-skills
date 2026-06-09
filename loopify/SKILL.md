---
name: loopify
description: >-
  Design, scaffold, and audit autonomous agent loops — recurring automations that
  find work, hand it to an agent, check the result against an objective gate,
  record state, and decide the next move without a human in the chair. Two modes:
  DESIGN ("loopify <task>") qualifies a recurring task with the 4-condition test
  (and refuses when it fails), designs the minimum viable loop (one automation,
  one standing context, one state file, one gate), scaffolds it under the repo's
  .claude/loops/<name>/, dry-runs it, then schedules it via /loop, cron, a cloud
  routine, or a hook. AUDIT ("audit my loops") inspects existing loops, crons,
  routines, and hooks for the classic failure modes — no objective gate, maker
  grading its own work, no hard stop, stale state, gate rot, permission creep.
  Use when the user wants to automate a recurring task, turn a manual prompt into
  a scheduled loop, decide whether something should be a loop at all, or health-
  check existing automations.

  Triggers: "loopify", "turn this into a loop", "design a loop", "automate this
  task", "should this be a loop", "set up a recurring agent", "minimum viable
  loop", "loop engineering", "audit my loops", "loop audit", "check my
  automations", "is my loop healthy".
---

# Loopify — design loops that earn their keep

The leverage with coding agents moved one floor up: from typing prompts to
designing the system that prompts — the loop that finds work, hands it to an
agent, gates the result, records state, and decides the next move. But loops are
not free: they re-read context, retry, and spend whether or not they ship. **Most
candidate tasks should not be loops, and this skill's first job is to say no.**
When a task qualifies, build the smallest loop that works — four parts, no swarm —
in an order that is not optional.

```
1. QUALIFY      →  2. DESIGN        →  3. SCAFFOLD       →  4. DRY-RUN        →  5. SCHEDULE
4-condition        trigger tier,       .claude/loops/       run the body once     /loop, cron,
test; any fail     gate command,       <name>/: LOOP.md     manually; PROVE       routine, or hook
→ hand back a      hard stops,         + STATE.md           the gate can fail     — never before
good prompt        escalation                               bad work              dry-run passes
```

## Mode selection

- **DESIGN** — "loopify X", "turn X into a loop", "automate X": run steps 1–5 below.
- **AUDIT** — "audit my loops", "is my loop healthy": skip to the audit section.

---

## DESIGN mode

### Step 1 — Qualify (be willing to say no)

All four conditions must hold. Check them against reality — run the commands,
don't take the premise on faith:

| # | Condition | How to verify |
|---|---|---|
| 1 | **The task recurs ≥ weekly.** | Ask, or check git/CI history. Less than weekly → setup never amortizes; a saved prompt or plain skill wins. |
| 2 | **An objective gate exists.** A command with an exit code — tests, type check, build, linter — that can reject bad output with nobody in the room. | Actually run the candidate gate now. "A second agent reviews it" is not a gate — that's two optimists agreeing. |
| 3 | **The budget absorbs the waste.** Loops retry and re-read; heavy verification on a metered plan ends in a rate limit or an invoice. | Size it: a single-agent run is ~50–200k tokens; a daily schedule compounds to millions per week. On consumer plans, check headroom with `track-usage` / `harness-usage` before committing. |
| 4 | **The executor has senior-engineer tools.** Logs, a repro environment, the ability to run the code it changes. | Confirm the loop's runtime context can actually execute the gate and reproduce failures. |

**Fail any condition → STOP.** Deliver the alternative instead — a well-aimed
reusable prompt or an ordinary skill — and name the condition that failed. This
refusal is half the skill's value.

One exception: when **budget is the only failing condition**, a cheaper executor
can flip it — run the loop body on a low-cost provider via `opencode`
(model-agnostic) or `llm-endpoint`. The objective gate keeps quality honest
regardless of executor strength; budget-bound loops are exactly where
engine-agnosticism pays.

Hard exclusions regardless of the test (these keep a human in the chair):
architecture rewrites, auth/payments/billing code, production deploys, dependency
*merges* (drafting bump PRs is fine), and anything where "done" is a judgment
call. Good first loops look like: CI-failure triage, lint-and-fix passes, flaky
test reproduction, dependency-bump *drafts*, issue-to-PR drafts on a well-tested
codebase.

### Step 2 — Design the minimum viable loop

Four parts, no swarm: **one automation, one standing context, one state file, one
gate.** Decisions to lock, in order:

**Trigger tier** — match the mechanism to where the loop must survive:

| Tier | Use when | Mechanism |
|---|---|---|
| Session cadence | Watch something while you work; dies with the session | `/loop` skill (fixed interval or self-paced) |
| Survives restarts, runs locally | Nightly/weekly jobs on this machine | `CronCreate` / desktop scheduled tasks |
| Laptop-off | Must run in the cloud on a schedule | `/schedule` (routines) |
| Event-driven | On PR open, file change, tool event | hooks via `update-config` skill, or GitHub Actions |

**The gate** — the only definition of done. A command with an exit code, recorded
in `LOOP.md`. For open-ended output (writing, classification, triage quality)
where no exit code exists naturally, build one with the `eval-loop` skill: rubric
+ judge + threshold, judged by bare `claude -p`. Note: the `/goal` primitive from
viral posts does not exist here — the real equivalent is the gate command in the
loop's stop condition, optionally double-checked by a fresh judge that never saw
the maker's reasoning.

**Hard stops** — all three, no exceptions: a per-run cap (iterations / time /
budget), a stop-on-success condition (the gate passes), and a
stop-on-repeated-failure condition (after K consecutive failed runs, escalate
and *stop spending* — never loop until something external kills it).

**Maker/checker split** — the run that writes must not be the thing that decides
"done." The cheapest honest version is the gate command itself. When a second
opinion is worth paying for, use an independent verifier — a subagent, or an
external agent via `codex-exec` / `cursor-agent` / `opencode` — but it
supplements the gate, never replaces it.

**Escalation policy** — what the loop does when blocked: append the item with
evidence to `STATE.md → ## Escalated`, optionally notify (PushNotification, or
Slack/Linear via MCP if connected). It must never disable failing tests, widen
its own permissions, or edit its own `LOOP.md` to make the problem go away.

**Parallelism** — only if runs can collide on files: give each run its own
checkout (`--worktree`, or `isolation: worktree` on subagents). Otherwise skip
it; review bandwidth, not the tool, is the real ceiling on parallel loops.

### Step 3 — Scaffold

Per-project layout — state is version-controlled and diff-readable:

```
<repo>/.claude/loops/<name>/
  LOOP.md    # standing context, reread every run: mission, scope fences,
             # gate command, hard stops, escalation. The loop never edits it.
  STATE.md   # the only file the loop mutates: last run, in progress,
             # completed, escalated, lessons learned.
```

Copy `templates/loop-spec.md` → `LOOP.md` and `templates/state.md` → `STATE.md`,
fill them in from the Step 2 decisions. `LOOP.md` rereads cure goal drift
(summarization is lossy; "don't do X" constraints vanish at turn 47 unless they
live in a file). Write it engine-agnostic — it must read correctly whether the
executor is Claude in a session, a routine, or a delegated external agent.

### Step 4 — Dry-run before scheduling

Order matters: get one manual run reliable → then wrap → then schedule. Skipping
ahead is how loops fail in production.

1. **Execute the loop body once, manually, now** — find one unit of work, do it,
   run the gate, update `STATE.md` exactly as a scheduled run would.
2. **Prove the gate can say no.** Feed it known-bad work (revert a fix, plant a
   failing case). A gate that has never failed anything is not a gate — this is
   the single check that prevents the Ralph Wiggum loop (agent declares done
   early, loop exits on a half-done job, keeps spending quietly).
3. Fix whatever was awkward. Schedule only when the manual run is boring.

### Step 5 — Schedule and hand off

Wire the trigger chosen in Step 2. The scheduled prompt stays one line — all the
real content lives in the files:

```
Read .claude/loops/<name>/LOOP.md and STATE.md. Do the next unit of work within
the scope fences, run the gate, update STATE.md. Stop per the hard stops.
```

Then tell the user what to watch: **cost per accepted change** — not tokens
spent, not tasks attempted. If the accepted rate drops below ~50%, the human is
doing the review work the loop was meant to remove, and the loop is losing: audit
it or kill it.

---

## AUDIT mode

Enumerate every loop, then grade each against the failure catalog.

**Find them:** `.claude/loops/*/` in the repo(s), `CronList`, `/schedule` list,
hooks in `settings.json` / `settings.local.json`, any `/loop` running this
session, and GitHub Actions that invoke agents.

| Failure mode | Smell | Fix |
|---|---|---|
| **Ralph Wiggum** (quiet early exit) | "Done" entries in STATE.md with no gate evidence | Gate command output required in the stop condition |
| **Self-grading maker** | The verifier is the same agent, or just an opinion | Objective command; independent judge if open-ended |
| **No hard stop** | No iteration/budget cap; stops only when something external kills it | Add caps + escalate-after-K-failures |
| **Gate rot** | Gate hasn't failed anything in memory | Feed it known-bad work now; tighten until it bites |
| **Stale state** | STATE.md last-run is old or never written | Run prompt must update state — or the loop is dead; kill it |
| **Goal drift** | Work outside LOOP.md's scope fences | Reread LOOP.md each run; tighten the fences |
| **Permission creep** | Loop's allowlist grew since design ("just one write permission") | Re-audit to least privilege; recheck every ~30 days |
| **Comprehension debt** | Merged diffs nobody has read | Sample recent diffs with the user; block the loop from judgment-call work |
| **Security tax** | Secrets in verbose logs; auto-installed skills; unreviewed merges to sensitive paths | Sanitize logging, review skill sources before install, human gate on merge/deploy |

Output a verdict table — one row per loop: **keep / fix / kill**, with the
specific finding and fix. Apply fixes when asked.

---

## Honest scope

- **This skill builds closed loops** — bounded unit of work, gate on every pass,
  hard stops. Open/exploratory looping (hand the agent a goal and let it roam)
  is a research tool for unmetered budgets; pointed at real work with loose
  standards it's a slop machine. Out of scope here — say so when asked for one.
- **Most tasks fail qualification.** The honest version of loop engineering is
  that a single well-aimed prompt still wins for one-offs, exploration, and
  judgment-call work. Saying "no loop" with a good prompt instead is a success.
- **A loop manufactures review work.** If review capacity is already the
  bottleneck, a loop makes the queue longer, not shorter.
- **Gates rot and this skill can't prevent it** — it can only make the rot
  findable. Re-run AUDIT periodically (a quarterly loop that audits the loops is
  fine; an hourly one is the disease it's checking for).
- **The metric is cost per accepted change.** A loop that's busy is not a loop
  that's winning.

## Files in this skill

- `templates/loop-spec.md` — scaffold for `LOOP.md`: mission, qualification
  record, trigger, one unit of work, gate, hard stops, scope fences, escalation,
  executor. Copy to `<repo>/.claude/loops/<name>/LOOP.md`; the loop never edits it.
- `templates/state.md` — scaffold for `STATE.md`: last run, in progress,
  completed (with gate evidence), escalated, lessons learned, gate runs. The only
  file the loop mutates.

## Notes / gotchas

- **Run the gate during qualification, not just at dry-run.** A gate that exists
  but doesn't run (missing env, broken CI config) fails condition 2 today.
- **The loop never edits LOOP.md, the gate, or its own permissions.** Any change
  there is a human design decision — put it in the scope fences.
- **Lessons learned go in STATE.md, not chat.** The agent forgets each run; the
  file does not. Durable environment facts ("this runner needs bash, not
  PowerShell") belong there the moment they're discovered.
- **Keep the executor pluggable.** Claude in-session, a routine, `codex exec`,
  `cursor-agent`, `opencode run` — pick per task; never hardcode one vendor into
  LOOP.md.
- **Read the diffs.** The skill schedules work; it does not absolve the human of
  comprehension. Unread merged code is debt at compound interest.
