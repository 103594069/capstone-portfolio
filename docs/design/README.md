# Design documents

Four design documents were written over the project, totalling 3,312 lines, all
sole-authored by me. Two are included here; the other two are described below.

The pattern in all four: a "context and goals" section naming the observed failure that
motivated the work, a design section, a testing section, a risks section, and a
"follow-up work (not this change)" section that keeps scope honest.

## Included

| File | Lines | What |
|---|---|---|
| [`synthesis-relevance-gate-design.md`](synthesis-relevance-gate-design.md) | 225 | Design spec. Stopping the chatbot from fabricating connections between real, unrelated facts, after a live answer misattributed a named academic source. Records the decision *not* to fix it in the prompt, because two prompt-only attempts had already failed |
| [`synthesis-relevance-gate-plan.md`](synthesis-relevance-gate-plan.md) | 643 | The implementation plan written from that spec: file-by-file changes, the parser's behaviour on a missing marker, the tests to write first, and the rollback switch |

These two are included because together they are the shortest complete trace from a
measured live failure to a shipped fix, and short enough to actually be read.

## Not included

| Document | Lines | Why it is not here |
|---|---|---|
| Extraction & verification design spec (2026-09-21) | 509 | Specifies the strict-prompt trial, the two-step verification configuration, and the evidence/direction/readability checks that became the three gates. Several sections quote the source corpus directly |
| Extraction & verification harness implementation plan (2026-09-21) | 1,935 | A working document rather than a reading document — the full build-out of the eval harness, prompt versioning and gate wiring |

Code in this repository references the excluded spec by section number (for example
`§6.2` in `verification/wellformedness_check.py`). Those references are left in place
because they are part of the record of how the code was specified, even though the document
they point at is not published here.

## Note on the plan documents

The plans contain commands and host references from the live system. Host addresses have
been replaced with placeholders. A reference to a now-retired deployment has been replaced
with `<api-host>`.
