---
name: scope-to-ship
description: >-
  Run a feature from fuzzy idea to dogfooded, shipped code with ONE agent and a
  front-loaded alignment phase — instead of fanning work out across parallel
  sub-agents. The loop: discuss + scope (surface a couple of design options, lock
  the simplest-yet-most-correct-long-term direction, answer ≤5 high-impact
  questions), crystallize the decision into a tiny numbered ADR under docs/adr,
  execute to completion against a Master PRD, then dogfood the result end-to-end
  before reporting it done. Use this when the user wants to scope or design a
  feature before building, align on architecture, write or follow an Architectural
  Decision Record, build a non-trivial feature end-to-end in one pass, or confirm a
  change by actually using the app. The executor is pluggable — Claude directly, or
  a delegated coding agent picked per task — never hardcoded to one vendor.

  Triggers: "scope this feature", "let's align before building", "discussion
  first", "talk it through then build", "write an ADR", "architectural decision
  record", "docs/adr", "master PRD", "build this to completion", "execute to
  completion", "dogfood it", "one agent A-Z", "no sub-agent fan-out".
---

# Scope to Ship — one agent, aligned first, dogfooded to done

The leverage in modern coding agents is no longer parallelism. Splitting a feature
across a swarm of summarizing sub-agents — "token-maxxing" — is productivity
theatre: it fragments the one thing that actually matters, a single mind holding
the whole logic in context. This skill installs the opposite discipline. **One
agent reads the real code, aligns hard on intent up front, records the decision as
a law, then runs the whole feature to completion and proves it works by using it.**

The win is overwhelmingly in the *discussion* phase: get intent genuinely shared
and everything downstream stops being guesswork (not *free* — Phase 3 is real
work — just unambiguous). So this skill spends its budget there.

```
  1. DISCUSS   →   2. DECIDE   →   3. EXECUTE   →   4. DOGFOOD
   scope &          write a         Master PRD       use it like a user,
   align hard:      tiny numbered   + ONE mind       E2E; net ALL related
   2 options,       ADR under       runs it to       breakage before
   ≤5 Q's + recs    docs/adr        completion       calling it done

   └── a failure in use goes back to the ONE mind that still holds full context ──┘
```

## When to run it

- The user wants to **design or scope a non-trivial feature** before any code.
- The work touches **architecture** that future code (and agents) must respect.
- A change needs to be **built end-to-end and actually verified by use**, not just
  "looks correct in the diff."
- The user says "talk it through first," "align on this," or "write an ADR."

Skip the ceremony for tiny, mechanical, or obvious edits — this is for features
where getting the architecture right the first time pays for the discussion.

The four phases are a discipline you (the agent) run, not a script. The hard part
is human judgment in Phase 1; the rest is execution hygiene.

---

## Phase 1 — Discuss (the phase that matters)

Don't let critical work start from a two-sentence prompt. **However the user phrased
the request — even one fuzzy line — run Phase 1 against this 3-part contract:** after
researching and scoping, come back with (a) a couple of design *options* and (b) at
most 5 high-impact architectural questions. The ideal user-side ask makes this
explicit, and it's worth coaching the user toward it:

```
"We need to implement <X>. Research and scope this out, then report back with
 a couple of options for the simplest design that is the most correct solution
 long-term — and a maximum of 5 important architectural questions I need to answer."
```

Why each clause earns its place:

| Clause | What it buys |
|---|---|
| **"a couple of options"** | Activates exploration — the model compares real alternatives instead of committing to its first idea. |
| **"the simplest design"** | Kills over-engineering. |
| **"most correct solution long-term"** | Kills the throwaway quick-fix that doesn't scale. |
| **"a maximum of 5 ... questions"** | Caps alignment cost; forces the model to surface only the decisions that actually shape the architecture. |

Then, as the agent:

1. **Read the real thing yourself.** The existing `docs/adr`, relevant code, and —
   for any library/framework/API/CLI in scope — current docs via the `find-docs`
   skill (don't trust training data for API surface). **Do not route this reading
   through summarizing sub-agents** — a summary always drops the one detail that
   mattered, and the agent that will build must hold the raw logic in context.
   (Delegating one *entire* self-contained run to a single external agent is fine —
   it still reads the raw code itself; what's forbidden is fragmenting the *reading*
   across summarizers.)
2. **Report back** with the 2 options and the ≤5 questions. For **every** question,
   give your **recommended answer + a one-line why.** This is what collapses
   alignment to a confirmation: with ADRs in place your recommendation matches the
   user's intent roughly 95% of the time, so they answer with a single "yes" and
   correct only the rare mismatch.

**Exit:** one design direction is chosen and every question is answered. You and
the user share one mental model. Now — and only now — write code-adjacent artifacts.

## Phase 2 — Decide (crystallize into an ADR)

An **ADR** (Architectural Decision Record) is a small `.md` file that states a
decision as a *law*, so every future contributor — the user, a new hire, you on the
next feature, any delegated agent — thinks about the codebase the same way. Phase 1
is where the *effort* goes; the ADR is what *compounds* — the highest-leverage
durable artifact of the method, fully vendor-neutral, benefiting Claude, any
external coding agent, and humans equally.

Write it the moment Phase 1 lands a real architectural decision:

- Location: `docs/adr/ADR-NNNN-<slug>.md`, numbered in order (`ADR-0001`, …).
- Copy `templates/adr-template.md` from this skill and fill it in. Sections:
  **title = the rule in one sentence**, a Status / Date / Last-reviewed metadata
  block, Context, Decision, **Invariants** (numbered, each a checkable clause citable
  as `ADR-NNNN §N`), Enforcement (which arm keeps each invariant — automated lint /
  semantic review / formal spec), Consequences (the honest tradeoff + what breaks if
  ignored), References (core files + symbols).
- **Keep it tiny.** It is read by an agent on every future task — minimal tokens to
  convey the law. You don't need an elaborate prompt; "turn this discussion into an
  ADR" produces a good record. Use the `documentation` skill if you want help with
  structure.

**Exit:** a small numbered ADR exists capturing the decision and its invariants.

## Phase 3 — Execute to completion (one mind, no fan-out)

1. Write a **Master PRD** (`templates/master-prd.md` → `docs/prd/<feature>.md`). It
   is the durable source of truth that survives context compaction during a long
   run — the executor re-reads *this*, not the chat scrollback, to know what "done"
   means. It links the ADR(s) and lists the acceptance criteria (the dogfood script).
2. **Pick the executor for THIS task** (engine-agnostic — never hardcode one).
   **Default to a single continuous run, usually Claude right here in this thread;**
   reach for a delegate only when this specific task is genuinely better served by it:

   | Executor | Reach for it when | How |
   |---|---|---|
   | **Claude, in this thread** *(default)* | In-repo work where intent is subtle and you already hold the code in context — keep the logic in one mind and build directly, driving yourself against the Master PRD to completion. | Just build it; keep the PRD current. |
   | **`codex-exec`** | You want an independent implementer behind a hard OS-level sandbox, or a precise file:line review. | `codex exec "<brief + PRD path>" -s workspace-write`; review with `codex review`. Pipe context via stdin; when launching headless/in the background, append `< /dev/null` or it hangs reading stdin. |
   | **`cursor-agent`** | Fast, repo-aware edits, or writes that must stay isolated from your working tree. | `cursor-agent -p --trust`; `-w <worktree>` for worktree isolation. (No hard sandbox; can't pipe stdin.) |
   | **`opencode`** | You need a specific model/provider, or genuine model diversity for a second opinion. | `opencode run "<brief>" -m <provider/model> --agent build`. |
   | **`llm-endpoint`** | A one-shot answer over context you already hold — never the implementer (it can't read files or edit). | `llm-endpoint` skill. |

   Whichever you pick, the principle holds: **one agent owns the whole run and reads
   the raw code itself.** "Self-directed execute-to-completion against a Master PRD"
   is the generic form of goal-mode — the executor authors and pursues its own
   completion goal; the user never hand-writes a goal.

**Exit:** the Master PRD is fully implemented and the code is ready to be used.

## Phase 4 — Dogfood (use it before you call it done)

Code that *looks* bug-free routinely breaks in real use — and those are usually
**architectural** problems, not typos. So before reporting back, **actually use the
software end-to-end the way a real user would.** Pick whichever driver fits the
target (start with `verify`, which orchestrates launch + drive + judge):

| Target | Driver | How |
|---|---|---|
| Confirm a change behaves vs. intent | **`verify` skill** | Orchestrates launch + a driver + judges observed behavior against each acceptance criterion. Start here. |
| Bring the app up | **`run` skill** | Project-type-aware launch (CLI / server / TUI / Electron / browser / library). |
| Native / Electron / Tauri GUI | **`cua-driver` skill** | Snapshot the accessibility tree → act by element → re-snapshot; the AX-tree diff is the proof the action landed (no diff ⇒ report a silent no-op, not success). |
| Web app in a browser | **Playwright MCP** | `mcp__playwright__browser_navigate` → `browser_snapshot` (get element refs) → click/type/fill → snapshot again + screenshot as evidence. |
| Feature emits open-ended / AI output | **`eval-loop` skill** | Score against a rubric, gate regressions, fold every dogfood failure back as a permanent case. |

For a pure library with no UI, "dogfood" = exercise the **public API** through a
real script or end-to-end test, not mocked units.

**Net the breakage.** When dogfooding surfaces a failure, the agent that holds full
context should throw a net over *all* related breakage and redesign properly —
never patch the one symptom and quietly break two other things. This is exactly why
one mind held the whole feature.

**Gate before "done":** run `/code-review` (correctness bugs), then `/simplify`
(reuse/cleanup), and `/security-review` if the feature touches auth, secrets, or
untrusted input. Capture any new build/test/run conventions with `create-agentsmd`.

**Exit:** every acceptance criterion passes in real use, gates are clean, and the
ADR's invariants still hold. *Now* report back.

---

## Honest scope

- **This is a habit, not magic in a script.** No file in this skill builds your
  feature. The leverage is Phase 1 judgment + the discipline of dogfooding before
  declaring done. If you skip the discussion, the rest doesn't save you.
- **The ADR is the compounding asset.** Without a `docs/adr` that the next task
  reads, you re-litigate the same architecture every feature and lose most of the
  method's value. The first ADR is the expensive one; they pay off across features.
- **"One agent reads the code" ≠ "never delegate."** Delegating one *entire,
  self-contained run* to a single external agent (codex / cursor / opencode) is
  consistent — that agent still reads the raw code itself and holds the logic. What
  the method forbids is fragmenting the *reading and understanding* across multiple
  summarizing sub-agents. If a feature genuinely splits into independent streams
  that each need their own full context, that's orchestration, not fan-out — see
  the `ultracode-external-agents` skill for bridging real workflow steps. Default to
  one continuous run; reach for orchestration only when the work truly is parallel.
- **Dogfooding needs a runnable target + the right driver present** (`run`/`verify`,
  `cua-driver`, or Playwright MCP). If none can drive the app, say so rather than
  claiming it was verified.
- **Out of scope:** the "one orchestrator looping over many *stateful* coding
  agents" idea is an explicitly unproven future direction. It is mentioned only to
  be excluded — don't let it smuggle the fan-out anti-pattern back into the core.

## Files in this skill

- `templates/adr-template.md` — the ADR scaffold (title-as-rule, Context, Decision,
  numbered Invariants, Enforcement, Consequences, References). Copy to
  `docs/adr/ADR-NNNN-<slug>.md` and keep it tiny.
- `templates/master-prd.md` — the Master PRD skeleton (Goal, anchored ADRs, scope,
  ordered plan, acceptance criteria = the dogfood script, verification plan, Done).
  Copy to `docs/prd/<feature>.md`; it's the source of truth across compaction.

## Notes / gotchas

- **Don't ask for one answer — ask for a couple of options.** A single answer skips
  the exploration that surfaces the better design.
- **Always demand a recommended answer + why.** That's what collapses alignment to a
  one-word "yes" and keeps the question list cheap.
- **Cap alignment at ~5 high-impact questions.** A wall of questions is a failure of
  scoping, not thoroughness.
- **Never summarize the codebase through sub-agents during reading.** The builder
  reads raw code and keeps it in context.
- **Never report "done" on code you haven't used.** Looks-correct ≠ works.
- **On a regression, redesign holistically.** Use full context to net all related
  breakage; don't trade one bug for two.
- **Keep ADRs and the Master PRD lean.** They're re-read on every future task —
  minimal tokens, maximum signal.
