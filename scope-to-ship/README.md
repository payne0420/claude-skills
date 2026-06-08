# scope-to-ship

A [Claude Code](https://claude.com/claude-code) skill that runs a feature from a
fuzzy idea to dogfooded, shipped code with **one agent and a front-loaded alignment
phase** — instead of fanning the work out across a swarm of parallel sub-agents.

> Part of the [`claude-skills`](https://github.com/payne0420/claude-skills) collection.

The leverage in a modern coding agent is no longer parallelism. Splitting a feature
across summarizing sub-agents fragments the one thing that matters: a single mind
holding the whole logic in context. This skill installs the opposite discipline.

```
  1. DISCUSS   →   2. DECIDE   →   3. EXECUTE   →   4. DOGFOOD
   scope &          write a         Master PRD       use it like a user,
   align hard:      tiny numbered   + ONE mind       E2E; net ALL related
   2 options,       ADR under       runs it to       breakage before
   ≤5 Q's + recs    docs/adr        completion       calling it done

   └── a failure in use goes back to the ONE mind that still holds full context ──┘
```

## The loop

1. **Discuss** — the phase that matters. However you phrase the ask, the agent
   researches, reads the real code itself (no summarizing sub-agents), and reports
   back with *a couple of design options* (simplest, yet most correct long-term) and
   *at most 5 high-impact architectural questions* — each with a recommended answer,
   so you mostly just confirm.
2. **Decide** — crystallize the alignment into a tiny, numbered **Architectural
   Decision Record** under `docs/adr`. The ADR is the compounding asset: it makes the
   *next* feature cheap because the architecture is already settled.
3. **Execute** — write a **Master PRD** (the source of truth that survives context
   compaction), then run it to completion. One mind owns the whole run.
4. **Dogfood** — actually *use* the software end-to-end the way a real user would
   before reporting it done; on failure, net all related breakage and redesign
   holistically rather than patching one symptom.

## Engine-agnostic

The executor is **pluggable, never hardcoded.** The default is Claude doing the work
directly in one continuous thread. When a task is genuinely better served elsewhere,
the skill delegates *one entire self-contained run* to a coding CLI of your choice
(e.g. Codex, Cursor, opencode) — the forbidden thing is fragmenting the *reading and
understanding* across sub-agents, not delegating a whole run. Dogfooding maps onto
whatever drivers you have (a runner/verifier, a browser driver, or a native-GUI
driver). All of these are optional companions; the methodology is the invariant.

## Install

This skill ships in the [`claude-skills`](https://github.com/payne0420/claude-skills)
collection. Clone the collection and link this skill into a `skills/` directory that
Claude Code scans (`~/.claude/skills/` for yourself, or a project's `.claude/skills/`):

```bash
git clone https://github.com/payne0420/claude-skills ~/Github/claude-skills
ln -s ~/Github/claude-skills/scope-to-ship ~/.claude/skills/scope-to-ship
```

## Use

In Claude Code:

```
/scope-to-ship
```

…or just say *"scope this feature," "let's align before building,"* or *"write an
ADR"* and describe what you want to build. The skill will run the discussion phase
with you first.

## What's in here

- `SKILL.md` — the skill itself (the four-phase methodology).
- `templates/adr-template.md` — the ADR scaffold (the decision stated as a rule, with
  numbered, checkable invariants). Copy to `docs/adr/ADR-NNNN-<slug>.md`.
- `templates/master-prd.md` — the Master PRD skeleton. Copy to `docs/prd/<feature>.md`.

## Credits

Inspired by the "just talk to it" philosophy of working with a single capable agent,
and by the enterprise practice of Architectural Decision Records.
