# Master PRD: <feature name>

> The single source of truth for this build. It survives context compaction —
> the executing agent re-reads THIS, not the chat scrollback, to know what
> "done" means. Keep it current as the plan changes.

## Goal

One sentence: what is true when this is done that is not true now.

## Anchored decisions

- ADR-NNNN: <the law this feature must obey>  (docs/adr/ADR-NNNN-*.md)
- <any other ADR this touches>

## In scope

- <what this feature includes>

## Out of scope

- <explicitly NOT doing — guards against scope creep mid-run>

## Plan

Ordered, so the one agent can blast through without re-deciding mid-flight.

1. <step>
2. <step>
3. <step>

## Acceptance criteria (the dogfood script)

Concrete, user-perspective scenarios. Each is something you will actually DO in
the running app — not a unit test. "Done" = every one of these passes in real use.

- [ ] <as a user, I do X → I observe Y>
- [ ] <edge case: I do A with bad input → I see graceful Z>

## Verification plan

How acceptance will be proven, end-to-end:

- Launch: `run` skill / <command>
- Drive: cua-driver (native GUI) | Playwright MCP (web) | CLI invocation
- Judge: `verify` skill confirms observed behavior matches each criterion above
- Gates before "done": `/code-review`, `/simplify`, and `/security-review` if it
  touches auth, secrets, or untrusted input

## Done

All acceptance criteria pass via real-use dogfooding, related breakage netted and
fixed holistically, gates clean, ADR(s) still honored.
