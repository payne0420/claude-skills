---
name: harness-usage
description: >-
  Read and track usage / rate-limits / cost across the OTHER coding harnesses on
  this machine — Codex (OpenAI), Claude (account), Cursor, opencode, Gemini, and
  ~40 more providers — primarily via the **CodexBar CLI** (`codexbar usage
  --format json`), which fetches live per-provider rate-limit % + reset times in
  one command. Falls back to CodexBar's on-disk JSON (offline) and per-harness
  native sources.

  Use when the user asks "how much have I used on Codex / my weekly Codex limit",
  "what's my Cursor / opencode / Gemini / zai usage", "compare usage across
  harnesses", "how much has opencode cost me", "when does my Codex window reset",
  or "what does CodexBar track". For THIS Claude Code session's own
  5h/weekly/context usage, prefer the companion `track-usage` skill.

  Triggers: "codex usage", "cursor usage", "opencode cost", "gemini quota",
  "codexbar usage", "how much of my codex weekly limit", "cross-harness usage".
---

# Coding-harness usage

Best tool by far is the **`codexbar` CLI** (shipped by the CodexBar menu-bar app,
on PATH at `/opt/homebrew/bin/codexbar`). It fetches usage for every enabled
provider in one command and prints clean JSON. Use it first.

> **Secrets rule:** usage JSON is safe (percentages/resets/credits). But the
> credential files beside it (`~/.codexbar/config.json`, `~/.codex/auth.json`,
> `~/.local/share/opencode/auth.json`, `factory-session.json`) hold tokens —
> never read or print their values.

---

## 1. `codexbar usage` — primary cross-harness reader (live, all providers)

```bash
codexbar usage --provider all --format json          # every enabled provider
codexbar usage --provider codex --format json --pretty
codexbar usage --provider claude --format json
codexbar usage --provider cursor --format json       # yes — CodexBar fetches Cursor too
codexbar usage --status                              # human pace view (see note)
codexbar usage --provider codex --all-accounts --format json
```

It fetches from each provider's web/OAuth/API surface (source shown per result),
so it **does not need the GUI running** — but it **does make network calls** and
needs the stored cookies/creds. It honors your in-app provider toggles
(disabled providers come back with `null` windows).

**JSON shape** — an array, one object per provider:
```jsonc
[{ "provider":"codex", "source":"openai-web",
   "usage": {
     "loginMethod":"Pro 5x",
     "primary":   { "usedPercent":6,  "windowMinutes":300,   "resetDescription":"Resets 4:06 AM" },         // 5-hour
     "secondary": { "usedPercent":33, "windowMinutes":10080, "resetsAt":"2026-06-07T15:47:37Z" },           // weekly (7d)
     "tertiary": null,
     "extraRateWindows":[ ... ],
     "updatedAt":"2026-06-02T00:00:37Z" },
   // codex also adds: "openaiDashboard":{accountPlan, creditsRemaining, codeReviewLimit, primaryLimit, secondaryLimit}, "credits":{remaining}
}]
```
- `usedPercent` = **percent USED** (0–100, may be float). `primary`=5h, `secondary`=weekly, `tertiary` varies (e.g. Claude Opus / Codex extra).
- `resetsAt` = ISO-8601 **UTC**; `resetDescription` = human string (local tz).
- ⚠ **`--status` shows percent LEFT** (inverse) plus a burn-rate/"runs out in" estimate — don't mix it up with JSON's `usedPercent`.

**Agent one-liners:**
```bash
# Codex: plan + 5h + weekly (used %) + weekly reset
codexbar usage --provider codex --format json | jq -c '.[0].usage | {plan:.loginMethod, fivehr:.primary.usedPercent, weekly:.secondary.usedPercent, weekly_reset:.secondary.resetsAt}'

# All providers, compact summary (skip ones with no data):
codexbar usage --provider all --format json | jq -r '.[] | select(.usage.primary.usedPercent!=null) | "\(.provider): 5h=\(.usage.primary.usedPercent)%  weekly=\(.usage.secondary.usedPercent // "–")%  (resets \(.usage.secondary.resetsAt // .usage.primary.resetsAt // .usage.primary.resetDescription // "?"))"'
```
Enabled here include: codex, claude, cursor, opencodego, zai, minimax, mistral,
mimo, gemini (of 48 configured). `gemini` via CodexBar is unreliable (free tier
returned `usedPercent:100` / epoch-0 reset — treat as "unknown", prefer §5).

---

## 2. CodexBar on-disk JSON — offline fallback (no network, ~60s stale)

When you want **no network call** (or the CLI is unavailable), read what the
running app already fetched. Two parent dirs (`CodexBar`≡`codexbar` are the same
dir on the case-insensitive FS; `com.steipete.codexbar` is **different**):

```bash
# current Codex + Claude usage (last entry of each window is live):
python3 - <<'PY'
import json,os
H=os.path.expanduser("~/Library/Application Support/com.steipete.codexbar/history")
for prov in ("claude","codex"):
    p=f"{H}/{prov}.json"
    if not os.path.exists(p): continue
    d=json.load(open(p)); acct=d["accounts"][d["preferredAccountKey"]]
    print(f"== {prov} ==")
    for w in acct:
        e=(w.get("entries") or [{}])[-1]
        print(f"  {w['name']:8} used={e.get('usedPercent')}%  resets={e.get('resetsAt')}  captured={e.get('capturedAt')}")
PY
cat "$HOME/Library/Application Support/com.steipete.codexbar/openai-dashboard.json"   # codex plan/credits
cat "$HOME/Library/Application Support/CodexBar/codex-account-snapshots.json"          # multi-acct + Spark windows
tail -1 "$HOME/Library/Application Support/CodexBar/usage-history.jsonl"               # long time-series (codex weekly)
```
- Updates only **while CodexBar runs** — check the in-file `capturedAt`/`updatedAt` for staleness.
- ⚠ `codex-account-snapshots.json` `resetsAt` is **Core-Foundation epoch** (seconds since 2001-01-01); add `978307200` for unix. (`history/*.json` use ISO-8601.)
- Percent + reset only — no requests-left/$ figure here.

---

## 3. Codex CLI — direct fallback (no CodexBar)

No `codex usage` subcommand. The server `rate_limits` block is recorded into the
session rollout on each model turn (so it's only as fresh as your last turn):

```bash
f=$(find ~/.codex/sessions -name '*.jsonl' -exec grep -l 'rate_limits' {} + 2>/dev/null | xargs ls -t 2>/dev/null | head -1)
grep 'rate_limits' "$f" | tail -1 | python3 -c 'import sys,json,datetime as dt;rl=json.load(sys.stdin)["payload"]["rate_limits"];[print(k,"used="+str(w["used_percent"])+"%","resets="+dt.datetime.fromtimestamp(w["resets_at"],dt.UTC).isoformat()) for k,w in (("primary",rl.get("primary")),("secondary",rl.get("secondary"))) if w]'
```
`primary`=5h, `secondary`=weekly, `resets_at`=unix UTC. Force-refresh with
`codex exec 'hi' -s read-only` then re-read. CodexBar (§1/§2) is fresher.

## 4. Cursor — use CodexBar; the CLI has no usage

`codexbar usage --provider cursor --format json` **does** return live Cursor
usage (period-usage %, reset). The `cursor-agent` CLI itself exposes **no** usage
— only tier/auth: `cursor-agent about --format json` (`subscriptionTier`),
`cursor-agent status --format json` (`isAuthenticated`). (`~/.cursor/ai-tracking/
ai-code-tracking.db` is AI-authorship analytics, NOT quota.)

## 5. opencode — cost + tokens (not rate-limits)

opencode tracks consumption locally (no rate-limit %):
```bash
opencode stats                                       # human (also --days N, --models, --project "")
opencode db "SELECT ROUND(SUM(cost),4) cost_usd, SUM(tokens_input) input, SUM(tokens_output) output, COUNT(*) sessions FROM session" --format json
opencode db path                                     # → ~/.local/share/opencode/opencode.db (WAL)
```
`cost` is opencode's estimate from model pricing, not a billed amount. (The
`opencodego` provider in CodexBar is a different, hosted plan with its own % — not
this local cost.)

## 6. Gemini CLI — free tier, server-side quota (historical tokens only)

No local quota counter; free-tier RPD/RPM is revealed only by a `429`. Historical
per-session token usage (read-only, no network):
```bash
LATEST=$(ls -t ~/.gemini/tmp/*/chats/session-*.json 2>/dev/null | head -1)
python3 - "$LATEST" <<'PY'
import sys,json
d=json.load(open(sys.argv[1])); agg={'input':0,'output':0,'cached':0,'total':0}; reqs=0
for m in d.get('messages',[]):
    t=m.get('tokens')
    if t: reqs+=1; [agg.__setitem__(k,agg[k]+t.get(k,0)) for k in agg]
print('session',d.get('sessionId'),'requests',reqs,'tokens',agg)
PY
```
Live per-call stats: `gemini -p "hi" --output-format json` → `.stats` (⚠ spends a request).

## 7. Claude Code (this session) → use `track-usage`

For the running Claude Code session's own 5h/weekly/context %, the `track-usage`
skill reads the live statusline snapshot. CodexBar's `claude` provider (§1/§2) is
the **account-level** Claude usage (Max/Pro) — same windows, different vantage.

---

## Summary

| Harness | Live rate-limit % | Best source |
|---|---|---|
| **Codex** | ✅ | `codexbar usage --provider codex --format json` |
| **Claude (account)** | ✅ | `codexbar usage --provider claude` |
| **Claude Code (session)** | ✅ | `track-usage` skill |
| **Cursor** | ✅ (via CodexBar only) | `codexbar usage --provider cursor` |
| **opencode** | ➖ cost+tokens, no limits | `opencode db ... --format json` |
| **Gemini** | ❌ server-side (429 only) | `~/.gemini/tmp/*/chats/*.json` (history) |
| **40+ others** (zai, minimax, mistral, copilot, …) | ✅ if enabled | `codexbar usage --provider <name> --format json` |

Rule of thumb: **`codexbar usage --format json` first** (one command, all
providers, live), CodexBar on-disk files when offline, `track-usage` for the
current Claude Code session, `opencode db` for opencode cost.
