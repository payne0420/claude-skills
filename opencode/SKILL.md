---
name: opencode
description: >-
  Drive the opencode CLI non-interactively via `opencode run` to delegate coding
  work to a second, independent agent. Use this skill when the user wants a second
  opinion on a design or bug, an independent implementation plan, an independent
  code review of a diff or GitHub PR, a self-contained implementation task run
  headlessly, parallel/background work on a sub-task, or cross-checking your own
  work against another model (opencode is model-agnostic and can run many
  providers). Also use when the user explicitly mentions "opencode", "opencode
  run", or asks to "have opencode plan / look at / build / review" something.

  Triggers: "ask opencode", "get a second opinion from opencode", "have opencode
  plan this", "have opencode review this", "run opencode on X", "delegate this to
  opencode", "opencode run ...".
---

# opencode (headless)

`opencode run` runs the opencode agent **non-interactively** (headless): one
message in, the agent works autonomously, output to stdout. opencode is
**model-agnostic** — it can drive Claude, GPT, and many open models through
configured providers — which makes it a flexible independent implementer,
reviewer, or second opinion. Treat it like delegating to a capable peer in
another terminal.

Verify it is available and authenticated before relying on it:

```bash
opencode --version              # e.g. 1.15.13
opencode auth list              # shows configured providers/credentials
```

If a provider isn't set up, run `opencode auth login`. If unavailable, tell the
user — don't silently do the work yourself without saying so.

## Core invocation

```bash
# Message can be a positional or via --prompt; ALWAYS pin the model explicitly
opencode run "summarize what this repo does" -m opencode-go/deepseek-v4-pro

# Read-only analysis with the built-in plan agent
opencode run "explain the auth flow" --agent plan -m openai/gpt-5.5
```

**Pin the model with `-m provider/model`.** Without it, opencode falls back to its
configured default, which may be a free/router model rather than the one you want.
List options with `opencode models`.

⚠️ **Model ids are machine-specific — there is no universal id.** The configured
providers/prefixes differ per machine (e.g. `anthropic/claude-opus-4-8` only works
if you've set up an Anthropic provider — it does **not** exist by default). Always
`opencode models` and pin one that's actually listed. The examples below use
`opencode-go/deepseek-v4-pro` (a strong reasoning model) and `openai/gpt-5.5`;
substitute whatever your `opencode models` shows.

## Safety & permissions — agents are the control

opencode's guardrails are **agents**, each with a permission policy. Built-ins:

| `--agent` | Behavior | Use for |
|-----------|----------|---------|
| `plan` | **hard read-only** — write tools are unavailable; it refuses to edit ("you'd need to exit plan mode") | review, analysis, planning, second opinions |
| `build` (default) | full write + shell; default policy `allow *` | implementation, refactors, fixing tests |
| `explore` / `general` | scoped subagents | targeted search / delegated lookups |

How permissions actually behave in headless `run` (verified):

- **`plan` is genuinely read-only** — unlike cursor's soft `--plan`, opencode's plan
  agent has no write tools and will not edit even when pushed.
- **`build` writes in the workspace without any extra flag** — its default policy is
  `allow *`, so in-workspace edits/commands run with no prompt and no stalling. You
  do **not** need `--dangerously-skip-permissions` for ordinary edits.
- **Actions an agent marks `ask` are auto-REJECTED in headless** (it does not hang).
  E.g. writing outside the workspace logs `permission requested: external_directory
  (/tmp/*); auto-rejecting` and the write fails. `--dangerously-skip-permissions`
  flips those auto-rejects into auto-approvals — so use it only when the task must do
  something outside the default allow set (write outside the workspace, etc.), and
  tell the user.

```bash
# Ordinary in-workspace implementation — no skip-permissions needed
opencode run "fix the failing test in src/parser.ts" -m opencode-go/deepseek-v4-pro --agent build

# Only when it must touch paths outside the workspace:
opencode run "update the shared config in ../infra" -m opencode-go/deepseek-v4-pro \
  --agent build --dangerously-skip-permissions
```

## Key flags

| Flag | Purpose |
|------|---------|
| `-m, --model <provider/model>` | pick the model (e.g. `opencode-go/deepseek-v4-pro`, `openai/gpt-5.5`) |
| `--agent <name>` | `plan` (read-only) \| `build` (default) \| `explore` \| `general` |
| `--variant <level>` | provider reasoning effort: `minimal` \| `high` \| `max` |
| `--thinking` | show thinking blocks |
| `--format <fmt>` | `default` (formatted text) \| `json` (raw NDJSON events) |
| `-c, --continue` / `-s, --session <id>` | continue the last / a specific session |
| `--fork` | fork the session when continuing (branch off without mutating it) |
| `-f, --file <files...>` | attach file(s) to the message |
| `--dir <path>` | working directory |
| `--title <t>` | name the session |
| `--share` | create a shareable link for the session |
| `--dangerously-skip-permissions` | auto-approve non-denied permissions (unattended) |

## Workflows

### 1. Second opinion / pair programming
Get an independent take, read-only, with the `plan` agent.

```bash
opencode run "We're choosing optimistic locking vs a queue for X. Trade-offs?" \
  --agent plan -m openai/gpt-5.5

# Independent debugging hypothesis, attaching the relevant files
opencode run "What race condition could cause this flaky test?" --agent plan \
  -m opencode-go/deepseek-v4-pro -f src/worker.ts -f test/worker.test.ts
```

Pick a *different* provider than your own to get genuine model diversity, then
reconcile the two answers for the user — flagging where they agree or diverge.

### 2. Planning
The `plan` agent is read-only by design: it investigates and proposes a plan but
makes no edits.

```bash
opencode run "Plan adding OAuth login: files to touch, ordered steps, risks, tests." \
  --agent plan -m opencode-go/deepseek-v4-pro --variant high
```

Plan→execute handoff: capture the plan, then run a fresh `build` pass (don't reuse
the read-only session to write):

```bash
opencode run "Implement this plan step by step: $(cat plan.md)" \
  --agent build -m opencode-go/deepseek-v4-pro
```

### 3. Code review (diff or GitHub PR)
Use the hard-read-only `plan` agent and let it inspect the diff itself (it can run
`git diff`):

```bash
opencode run "Review the uncommitted changes for correctness bugs and risky edge cases. Run git diff yourself. Reference file:line and quote each buggy line." \
  --agent plan -m opencode-go/deepseek-v4-pro
```

Verified: on a repo with a planted operator bug + an unguarded division,
`deepseek-v4-pro` in the plan agent ran `git diff`, reported both, and correctly
flagged which one the diff introduced vs. pre-existing.

For a GitHub PR, opencode can check it out and run against it:

```bash
opencode pr 123                 # fetch + checkout PR #123, then run opencode on it
opencode github                 # manage the GitHub agent (automated PR review/triage)
```

Relay findings; offer to fix, but don't auto-apply unless asked.

### 4. Delegated implementation (headless)
Hand off a self-contained task with the `build` agent.

```bash
opencode run "Add input validation to the /signup handler and a unit test for it." \
  -m opencode-go/deepseek-v4-pro --agent build
```

In-workspace edits need no extra flag (build's default policy is `allow *`). Then
review what it changed (`git diff`) before trusting it.

### 5. Structured / scriptable output
`--format json` emits NDJSON events. The final answer is in `text` parts; this
extraction is verified working:

```bash
opencode run "one-line summary of this repo" --agent plan -m openai/gpt-5.5 --format json \
  | jq -rs 'map(select(.type=="text").part.text) | join("")'
```

For full session data after a run, use `opencode export [sessionID]` (JSON) and
`opencode stats` for token/cost totals.

### 6. Multi-turn (continue / session)
Continue a prior session with its context retained.

```bash
opencode run "now also add the migration" --continue
opencode run "address the review comments" --session ses_abc123
opencode run "try an alternative approach" --continue --fork   # branch without mutating
```

### 7. Parallel / background work & server mode
For long delegated tasks, launch in the background (in this harness,
`run_in_background: true`) and capture stdout. opencode can also run as a headless
HTTP server that multiple `run` invocations attach to:

```bash
opencode serve --port 4096                       # headless server
opencode run "do the big refactor" --attach http://localhost:4096 -m opencode-go/deepseek-v4-pro
```

### 8. Headless parallel fan-out (one agent spawns many)

opencode supports **two** fan-out modes — both verified robust under load (16
isolated children in a single wave = 16/16; 32 children in overlapping waves,
peak ~17 in-flight = 32/32; **no concurrency ceiling found, ~100% completion**).

**Mode A — shell-out children (isolated writes).** A `build` parent (its `allow *`
policy lets it run `opencode run` as a shell command — **no
`--dangerously-skip-permissions` needed**) spawns child `opencode run` processes.
opencode has **no native `-w`/worktree flag**, so isolate writes with a **manual
git worktree per child + `--dir`**:

```bash
for task in A B C; do
  git worktree add -q -b "wt-$task" "/tmp/wt-$task" HEAD
  ( opencode run "<self-contained prompt — stdin is ignored, inline ALL context>" \
      --agent build -m opencode-go/<model> --dir "/tmp/wt-$task" > "out-$task.log" 2>&1 ) &
done
wait   # MANDATORY (parent exit orphans children) — AND give each child a hard-kill timeout
# verify each child actually made its change (exit 0 ≠ work done), then merge worktrees
```

Each child is a separate OS process and session; nesting is unrestricted (no
recursion guard — children just inherit `OPENCODE=1`, `OPENCODE_PROCESS_ROLE`,
`OPENCODE_RUN_ID`). Use distinct worktrees: two children writing the **same file**
in the same tree can silently lose an edit.

**Mode B — native subagents (`task` tool).** Set
`OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=1` and opencode exposes a **`task`**
tool that spawns **truly parallel in-process subagents** (verified: two tasks ran
concurrently) — the clean analog to a UI "spawn subagents" button, no process
management:

```bash
OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=1 \
  opencode run "Use the task tool to do these in parallel: <subtask 1>; <subtask 2>." \
  --agent build -m opencode-go/<model>
```

Subagent types: `explore` (read-only) and `general` (write-capable) — both verified.
Native subagents **share the parent's workspace** (no `--dir` isolation), so give
each one a **disjoint file/area** to avoid collisions. Prefer Mode B for parallel
read/research/analysis; prefer Mode A when children must make **isolated** edits to
the same repo.

**Watchdog every child regardless of mode.** opencode does **not** self-recover
from a lost backend request — a rare transient stall leaves the process alive but
idle (0-byte output, no connection, no DB writes) indefinitely. A per-child
hard-kill timeout is the safety net; don't assume a hung child will ever return.

## Output / event format

- `--format default` → formatted text (shows the active `agent · model`, then the
  answer).
- `--format json` → NDJSON event stream: `step_start`, `text` (with `part.text`),
  `step_finish` (carrying `tokens` and `cost`), plus tool events when the agent
  acts. Parse `text` parts for the answer (see workflow 5).

## Monitoring a running instance (liveness)

Tell a backgrounded / orchestrator-owned run's state apart — finished, still
working, or wedged — without killing anything. All checks are **read-only**.

```bash
pgrep -fl "opencode run"                        # alive? (wrapper shells may also match)
ls -l out.txt                                   # redirected stdout — `> agent · model` header appears within seconds
stat -f '%z %Sm' ~/.local/share/opencode/opencode.db-wal   # global session DB, written at message boundaries
lsof -p <pid> -a -i | grep ESTABLISHED          # in-flight request
```

- **Finished**: exit 0, stdout holds the header + answer (verified: header within
  seconds, answer by exit). Hard failure → non-zero exit (verified: bad model →
  exit 1, `Error: Model not found` in the output).
- **Still working**: process alive + `opencode.db-wal` mtime advancing. The wal is
  **global** (any concurrent opencode invocation also freshens it — verified) and
  written only at message boundaries, so it goes quiet during a long tool call or a
  slow model turn — a few quiet minutes are normal.
- **Wedged** (observed live on a flaky network): process alive 10+ min, stdout
  still 0 bytes (the first response never arrived), db-wal untouched since startup,
  and the once-held ESTABLISHED connection gone with no replacement — the request
  was lost and opencode never recovered. The UTC-named per-invocation files in
  `~/.local/share/opencode/log/` are **not** a liveness signal — logging stops
  seconds after launch; a new file there only tells you a run *started*.

## Context files

opencode reads `AGENTS.md` for project/user conventions and project config from
`opencode.json` / `.opencode/`. Custom agents (with their own model + permissions)
can be defined there or via `opencode agent` — useful for a reusable read-only
"reviewer" agent. For a one-off run, put rules directly in the prompt.

## Gotchas

- **Pin the model** (`-m provider/model`) — the default may be a free/router model,
  not the one you intend. `opencode models` lists them.
- **Agent = permission policy.** `plan` is hard read-only (no write tools); `build`
  (default `allow *`) writes in-workspace with no flag and no stalling.
- **Headless auto-rejects `ask` permissions** (it doesn't hang). Anything outside the
  default allow set — e.g. writing outside the workspace (`external_directory`) — is
  auto-rejected and fails. Add `--dangerously-skip-permissions` to auto-approve those
  instead, and tell the user.
- **Review before trusting writes** — inspect `git diff` after a `build` run.
- **Model-agnostic is the strength** — for a true second opinion, run a different
  provider than your own and reconcile the two views for the user.
- **Don't leak secrets** into prompts sent to an external model/provider.
- **Parallel fan-out** (workflow 8): shell-out children need a manual `git worktree`
  + `--dir` (no native `-w`); native `task` subagents
  (`OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=1`) run parallel but share the
  workspace. opencode **won't self-recover a hung request** — always watchdog each
  child with a hard-kill timeout.
