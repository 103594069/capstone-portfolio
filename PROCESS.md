# How this was built

A record of how I worked on this project with Claude Code. The numbers and artefacts here
are all checkable against the original repository's history.

## The measurements

| | |
|---|---|
| My commits on the project | 163 |
| Of those, carrying a Claude co-author trailer | **134 (82%)** |
| Model versions used | Opus 5 (77), Sonnet 5 (49), Opus 5.5 (9), Haiku 4.5 (8) |
| Design documents written | 4, totalling 3,312 lines, all sole-authored |
| Unit tests in the original suite | 561 passing, 9 failing (stale assertions in a language test for answer languages that had been dropped — not included here) |
| Unit tests in this portfolio subset | 176 passing, 0 failing |

## The loop

Design spec → implementation plan → tests → code → measure → revise the design.

Two of the four design documents ship in [`docs/design/`](docs/design/). The other two are
listed in [`docs/design/README.md`](docs/design/README.md) with their line counts.

The parts of this that actually mattered:

**Specs came before code, and they recorded the measurement that motivated them.** The
documents are not retrospective write-ups. They open with a "context and goals" section
naming the observed failure, and the code carries comments pointing back at them —
`# WHY THIS EXISTS: measured live 2026-10-08, synthesis fabricated a citation...`. A year
from now the reason a gate exists is still in the file.

**Measurements were allowed to kill my own work.** Three times:

1. **The LLM verification pass.** I designed, specced and built a second-stage LLM pass to
   verify extracted triples. Measured against the deterministic gates alone, it showed no
   reduction in hallucination rate and cost 8+ hours of CPU time per paper. I made it
   opt-in, then removed it — commit `e4dc3da`, *"deterministic gates alone already hit 0%
   hallucination"*. The prompt text stayed recoverable in git history.
2. **The constrained prompt.** A prompt variant intended to stop predicate duplication made
   it reproducibly **worse** (Variant B in the README's evaluation table: 0–2 true positives
   against the baseline's 2–11). Reported as a negative result rather than quietly dropped.
3. **Prompt-only fixes for the fabricated citation.** Two attempts to fix it by instructing
   the model failed on live checks. That is what justified moving the control into code —
   the relevance gate parses the model's first line and discards everything after a
   `RELEVANT: no` rather than trusting the model to stop writing.

**Single LLM measurements were treated as untrustworthy.** Every prompt configuration was
run four times. Variant C scored 8, 7, 13 and 1 true positives on identical input with
`temperature` at 0.1. One run would have produced a confident and wrong conclusion. This is
the practice I would carry into any LLM work: a single number from a generative pipeline is
an anecdote.

**A worked example, end to end.** The relevance gate, which is the shortest complete trace:

| Step | Artefact |
|---|---|
| Failure observed on the live system | Chatbot answered a lineage question with *"as stated in Pütz (1991) who identified family as one of the domains of study"* — a real citation, misattributed, welding together two facts sharing only the word "family" |
| Why it mattered more than a vague answer | A misattributed named academic source reads as sourced, so it is more dangerous than hedging |
| Design spec | [`docs/design/synthesis-relevance-gate-design.md`](docs/design/synthesis-relevance-gate-design.md) — 225 lines |
| Implementation plan | [`docs/design/synthesis-relevance-gate-plan.md`](docs/design/synthesis-relevance-gate-plan.md) — 643 lines |
| Decision recorded in the spec | Do not fix this in the prompt. Two prompt attempts had already failed. Parse the model's output and discard in code |
| Implementation | `rag.py` — the gate makes the model emit `RELEVANT: yes|no` as its first line; a missing or unreadable marker fails closed to a refusal |
| Verified live | The fabricating question now refuses honestly; three previously-correct questions still answer, which was the over-refusal risk the change introduced |
| Limitation recorded, not hidden | All-or-nothing: mostly-good evidence plus one irrelevant triple is still synthesized. Per-triple classification is the next iteration |

**Commit messages carry the finding, not the diff.** A sample from the original history:

```
e4dc3da  Make the LLM verify pass opt-in, not default -- deterministic gates
         alone already hit 0% hallucination
b95afd8  Rank retrieval on the question's own words instead of a list the graph
         never matched
5cb610f  Stop "live" false-matching "livelihood" in retrieval scoring
10705e6  Judge a triple by the question's topic, not the community's name
d74d36f  Stop the rollback sweeping orphans it did not create
a7be324  Bound citation length inside grounding_check: a sentence_ref must be a
         sentence, not a paragraph
```

## What I would say about the tooling

The useful part was not generation speed. It was that the workflow made it cheap to write a
spec before writing code, cheap to run an experiment four times instead of once, and cheap
to write up a negative result properly. Those are the three things that most improved the
engineering, and all three are habits rather than features.

The limitations document in the original repository ran to 30 numbered entries with
measurements attached. Maintaining something like that by hand, alongside the implementation,
is the part I would not have done without the tooling — and it is the part that made the
project's real state legible to the team and to whoever inherits it.
