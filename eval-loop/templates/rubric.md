---
# The threshold: nothing below this ships. Hold it.
threshold: 0.7
# How much a single case may drop vs baseline before it counts as a regression.
regression_epsilon: 0.05
# Criteria + weights. Should sum to 1.0; if they don't, score.py renormalizes them
# and prints a warning. A missing criterion in the judge's reply counts as 0 (it
# drags the score down, never inflates it). Edit these to encode YOUR taste — this
# file IS the standard.
criteria:
  - id: specific
    weight: 0.30
  - id: accessible
    weight: 0.20
  - id: replicable
    weight: 0.25
  - id: novel
    weight: 0.25
---

# Rubric: <target name>

You are grading a piece of output against the standard below. For each criterion,
return a score from 0.0 to 1.0 and a one-line reason. Be a strict, ego-free critic —
when in doubt, score lower. A polished-but-hollow piece is slop and should fail.

## Criteria

### specific  (weight 0.30)
Does it explain how to do something specific — an action the reader can take
tomorrow — rather than gesturing at a vibe?
- **1.0** — Contains at least one concrete, copy-pasteable step, template, or number.
- **0.0** — Abstract encouragement; nothing the reader could actually execute.

### accessible  (weight 0.20)
Can anyone in the intended audience follow it without inside-baseline jargon?
- **1.0** — Plain language; every term either common or defined in place.
- **0.0** — Jargon wall; assumes context the audience doesn't have.

### replicable  (weight 0.25)
Is it structured and step-by-step, something a reader could reproduce?
- **1.0** — Clear ordered steps or a named procedure.
- **0.0** — Inspirational but unstructured; no path to reproduce the result.

### novel  (weight 0.25)
Does the reader learn something they didn't know was possible?
- **1.0** — A genuinely non-obvious insight, method, or framing.
- **0.0** — Generic, could have been written by any account on the timeline.

## Meta-criterion (applies on top of all four)

Would someone bookmark this and come back to implement it later? If the honest
answer is no, the overall score should land below the threshold no matter how clean
the prose reads. Reflect this in the weakest applicable criterion.

---
<!--
PRODUCT / NON-CONTENT TARGETS: replace the criteria above with task-matched ones,
e.g. correctness (matches gold), format (valid JSON / schema holds), grounding (no
hallucinated facts/numbers), tone (on-brand), safety. Keep weights summing to 1.0
and give every criterion concrete 0.0 / 1.0 anchors. The more specific the anchor,
the more trustworthy the score.
-->
