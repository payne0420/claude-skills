# ADR-NNNN: <the law, stated as a rule>

Status:        Accepted   <!-- Proposed | Accepted | Superseded by ADR-NNNN | Deprecated -->
Date:          YYYY-MM-DD
Last-reviewed: YYYY-MM-DD

## Context

Why this law exists. What the system looked like before it, and what breaks
without it. Keep this to the few sentences that make the rule feel inevitable.

## Decision

The rule itself, stated precisely and plainly. A reader should be able to obey
it from this paragraph alone.

## Invariants

The brief checkable clauses a reviewer verifies. Number them so a reviewer
(human or agent) can cite them precisely as `ADR-NNNN §N`.

1. <clause one — a single must-hold truth>
2. <clause two>
3. <clause three>

## Enforcement

State which arm enforces each invariant. Map every invariant to exactly one of:

- **automated** — a lint rule / type / test that fails the build every commit
  (e.g. `eslint <plugin>/<rule-name>`)
- **semantic** — caught only by a reviewer reading intent; no static rule
  possible (a human or an agent review pass)
- **formal** — a machine-checked spec (e.g. `specs/<name>.tla`)

<!-- The example commands above are illustrative — swap in this project's own
     enforcement tools. The point is that each invariant names who keeps it. -->

## Consequences

What this law costs the team, and what it buys. Name the tradeoff honestly so a
future reader understands why it was worth it — including what goes wrong if the
rule is NOT followed.

## References

- Code: `path/to/file.ts` (`symbol`) — core service agents must understand / import from
- Related: ADR-NNNN (supersedes / depends on), `docs/prd/<feature>.md`
