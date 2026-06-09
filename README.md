# claude-skills

A collection of [Claude Code](https://claude.com/claude-code) skills — reusable,
model-driven workflows that extend the agent. Each folder is a self-contained skill
(a `SKILL.md` plus any templates/scripts) that Claude Code loads on demand.

A guiding principle across these: **engine-agnostic.** Where a skill can delegate
work to another coding agent, it offers a neutral menu (Codex / Cursor / opencode /
a raw endpoint) and lets Claude pick per task — nothing is hardcoded to one vendor.

## Skills

### Build workflows
| Skill | What it does |
|---|---|
| [`scope-to-ship`](scope-to-ship/) | Ship a feature with **one aligned agent**: discuss + scope → write a tiny ADR → execute against a Master PRD → dogfood end-to-end. The anti-fan-out methodology. |
| [`ultracode-external-agents`](ultracode-external-agents/) | Author ultracode workflows whose `agent()` steps run on an **external CLI** (Codex / Cursor / opencode) instead of Claude subagents — bridge any step onto another model. |
| [`loopify`](loopify/) | Design, scaffold, and **audit autonomous agent loops**: qualify a recurring task (and refuse when it doesn't fit), build the minimum viable loop (automation + standing context + state file + objective gate), dry-run, then schedule. |

### Delegate to another coding agent
| Skill | What it does |
|---|---|
| [`codex-exec`](codex-exec/) | Drive OpenAI's **Codex** CLI headlessly (`codex exec`) — independent implementer, reviewer, or second opinion behind a hard OS sandbox. |
| [`cursor-agent`](cursor-agent/) | Drive **Cursor**'s CLI agent headlessly (`cursor-agent -p`) — fast repo-aware edits/reviews, with git-worktree isolation. |
| [`opencode`](opencode/) | Drive the **opencode** CLI (`opencode run`) — a model-agnostic second agent; pin any provider/model. |
| [`llm-endpoint`](llm-endpoint/) | Call a configured OpenAI-/Anthropic-compatible HTTP endpoint as a **lightweight model backend** (a raw completion, not an agent). |

### Quality & docs
| Skill | What it does |
|---|---|
| [`eval-loop`](eval-loop/) | Stand up an **eval loop**: score AI output against a rubric, gate what's below the line, turn every failure into a permanent test. |
| [`find-docs`](find-docs/) | Fetch **current** library / framework / SDK / CLI docs via Context7 — instead of trusting training-data API surface. |

### Usage tracking
| Skill | What it does |
|---|---|
| [`harness-usage`](harness-usage/) | Read usage / rate-limits / cost across **other** coding harnesses (Codex, Cursor, opencode, Gemini, …) via CodexBar. |
| [`track-usage`](track-usage/) | Track **this** Claude Code session's own 5-hour / weekly / context usage and when each resets. |

## Install

Skills live in a `skills/` directory that Claude Code scans — either your personal
`~/.claude/skills/` or a project's `.claude/skills/`. Clone this repo and link (or
copy) the skills you want into place.

```bash
git clone https://github.com/payne0420/claude-skills ~/Github/claude-skills

# enable one skill for yourself (symlink so `git pull` keeps it current):
ln -s ~/Github/claude-skills/scope-to-ship ~/.claude/skills/scope-to-ship

# …or enable all of them:
for d in ~/Github/claude-skills/*/; do
  [ -f "$d/SKILL.md" ] && ln -sfn "$d" ~/.claude/skills/"$(basename "$d")"
done
```

Then invoke a skill in Claude Code by name (e.g. `/scope-to-ship`) or just describe
what you want — Claude routes to the matching skill via its `description`.

## Notes

- Some skills wrap external CLIs (`codex`, `cursor-agent`, `opencode`, `ctx7`,
  `codexbar`). Those tools must be installed and authenticated separately; each
  skill checks for its dependency and tells you if it's missing.
- `llm-endpoint` reads its base URL + API key from a **local** file
  (`~/.config/llm-endpoint/env`) that is never committed — see
  [`llm-endpoint/config.example.sh`](llm-endpoint/config.example.sh).

## License

[MIT](LICENSE).
