# Known limitations

Ten findings from the original project's limitations register, which ran to 30 numbered
entries. Each was measured, not estimated. Teammate names, live infrastructure detail and
database identifiers have been removed.

The register existed because the system had no human review step between extraction and a
user-facing answer. Anything it got wrong went straight to a reader, so knowing precisely
what it got wrong was part of the deliverable.

---

## 1. The knowledge graph is small, and deliberately so

45 triples across three papers, from 578 extracted candidates.

| Paper | Extracted | Kept | Flagged: well-formedness | Flagged: grounding | Flagged: quote |
|---|---|---|---|---|---|
| Paper 1 | 220 | 16 | 39 | 195 | 70 |
| Paper 2 | 198 | 24 | 54 | 156 | 70 |
| Paper 3 | 160 | 5 | 26 | 140 | 127 |
| **Total** | **578** | **45** | | | |

A triple can be flagged by more than one gate, so the flag columns do not sum to the
rejected count.

This is the intended trade. The graph feeds a chatbot with no human review step, so a wrong
fact reaches an end user directly while a missing one only means the chatbot knows slightly
less. The gates are a hard gate: a flagged triple is dropped, not queued.

## 2. One paper contributes almost nothing, and the reason is its PDF

127 of Paper 3's 160 candidates fail the quote gate — 79%, against 32% and 35% for the other
two.

The quote gate checks that a triple's cited sentence appears verbatim on its own page of the
source PDF. A failure rate that far out of line indicates the PDF's text layer does not
match what the model read — most likely a two-column layout or a scan, where `pdfplumber`
returns text in an order that does not reconstruct into the sentences the model quoted.

This is a text-extraction problem, not a model problem, and nothing in the output
distinguishes the two. An operator seeing a 79% quote-gate failure has no signal telling
them to go and look at the PDF rather than at the prompt.

## 3. The graph is a star, not a graph

Of 67 distinct entities across the 45 triples, **59 appear exactly once**. One community
entity touches 13 triples; the next-largest hub is a bibliography citation, with 5.

A knowledge graph earns its name from traversal — "what connects X to Y". This shape cannot
do that. Almost everything either hangs off the one hub or is isolated.

Two causes:

- `insert_triple` does `MERGE (s:Entity {name: $subject})`, matching on the name string
  alone. Nothing reconciles entities, so two spellings of the same people are two nodes.
- Bibliography citations are extracted as entities. A reference is not a thing in the world.

Entity resolution should come before growing the graph. It gets harder with every paper
added.

**This turned out to be the root cause of two other defects** that looked independent: the
synthesizer padding short answers with off-topic evidence (#7), and no single relevance
threshold being able to separate "genuinely about this question" from "merely mentions the
community" for the sources panel. Both are this limitation surfacing downstream.

## 4. The predicate vocabulary is unbounded

32 distinct predicates across 45 triples; 26 used exactly once. Five describe language use
and none match each other: `LANGUAGE_USED`, `HAS_LANGUAGE_Maintenance`, `EXTENT_OF_USE`,
`IS_USED_IN`, `Emphasizes_Languages`.

`clean_relation` only uppercases and replaces non-alphanumerics, so whatever the model emits
becomes a relationship type. Each new paper adds roughly ten novel single-use types. At 23
papers that is around 250 types for around 350 edges, and no one can write a reliable
`MATCH ()-[r:USES_LANGUAGE]->()` against it.

The fix is either a controlled vocabulary in the extraction prompt, which only takes effect
on re-extraction (~1.2 h/paper), or a canonical mapping applied at load time in
`neo4j_loader/cleaner.py`.

## 5. The read path is a full scan with no index

The graph query matches `(s:Entity)-[r]->(o:Entity)` and tests `CONTAINS` against five
properties per relationship. There is no index and no full-text index.

Free at 45 triples and fine into the low thousands. It is the documented ceiling on this
read path: a few tens of thousands of relationships will need a full-text index and a
different query shape. A `FETCH_LIMIT` of 500 bounds the worst case, so an unexpectedly
large graph degrades rather than loading everything into memory.

## 6. The confidence framework's verdicts are unstable

The framework scores a paper for admission to the corpus. The same paper, scored 11 times:

```
 5,  5,  5, 37, 58, 59, 63, 65, 67, 80, 80
```

Against thresholds of 60 (manual review) and 75 (approve), that single paper spans REJECTED
through APPROVED. `temperature` is 0.1 with no seed, and the per-criterion score is
`max(scores)` across chunks rather than the average, so one favourable chunk can carry a
paper.

Worse: **an infrastructure failure is recorded as an editorial verdict.** When the LLM call
fails or its JSON will not parse, the fallback returns all four LLM-scored criteria at 0 —
85 of the available points — and the paper is filed as REJECTED. A network blip and a bad
paper are indistinguishable in the log.

The module is included in this repository because it is substantial work and the finding is
instructive, not because it should be relied on.

## 7. The synthesizer invents connections between real, unrelated facts

**Partly fixed.** A relevance gate makes the model emit `RELEVANT: yes` / `RELEVANT: no` as
its first line, and the code **discards everything after a "no"** rather than trusting the
model to stop writing. A missing or unreadable marker fails closed to a refusal. Design in
[`docs/design/`](docs/design/).

What prompted it, measured on the live site: a lineage question returned a **fabricated
citation attribution** — *"as stated in Pütz (1991) who identified family as one of the
domains of study"* — welding together two facts sharing only the keyword "family". The cited
work is a sociolinguistics framework about which *settings* a language is used in.
Misattributing a named academic source is worse than vague hedging, because it reads as
sourced.

An earlier instance was subtler: on a question with thin evidence, the model wrote
*"bamboo floors... suggests their religious practices may involve rituals"* and
*"bilingual... could indicate religious practices may be influenced by both languages."*
Neither claim exists in any evidence. It did not invent a fact; it invented a *connection*
between two real facts, and both read as trustworthy prose.

**What the gate does not fix.** It is all-or-nothing: a question with mostly good evidence
plus one irrelevant triple is still synthesized, and the model may still weave the
irrelevant one in. Per-triple classification is the next iteration.

**And retrieval scoring cannot substitute for it.** The fabricating question's top retrieval
score (22.05) was *higher* than two genuinely well-answered questions (16.14, 20.96),
because IDF weighting rewards keyword rarity, not topical relevance. No absolute threshold
separates them.

## 8. The relevance gate refuses questions it can answer

Measured on a live benchmark of eight questions: four answered and graph-grounded, three
refused, one timed out.

One of those refusals was wrong. Asked for the community's population, the system returned
`Population_Census_2022 -[POPULATION]-> 1,650,159` as the **top-ranked triple in its own
evidence** and then said it could not answer. The fact was in the corpus, retrieval found
it, retrieval ranked it first, and the gate refused anyway.

Precision-first is the right default for this system, but a gate that refuses its own
top-ranked evidence is not being cautious — it is broken.

## 9. End-to-end automation works, and yields almost nothing

Approving a paper triggers extraction automatically. The wiring was proven on the live
instance, not just in tests:

| Step | Result |
|---|---|
| Confidence framework | APPROVED, 83/100 (one chunk of eight hit a parse error and was skipped) |
| Extraction trigger | Automatic on approval |
| Run time | **73 minutes** |
| Raw triples | 98 |
| Survived the three gates | **1** |
| Loaded to the graph | 1 |

**The wiring works. The pipeline behind it does not.** A 1-in-98 survival rate is not the
gates being harsh — the rejected triples were visibly broken, with predicates containing
spaces and subjects that were sentence fragments. The gates did their job.

**The single survivor was also wrong.** So the honest summary of a 73-minute automated run
is: zero usable triples.

## 10. Credentials were committed, and the history was not rewritten

Three times over the project, database credentials reached version control: once in a
`.env` committed to the default branch, once as a real password inside an `.env.example`,
and once in a local file. All three instances were later decommissioned, so the exposure was
a process failure rather than a live incident — but the history was never rewritten, because
a coordinated history rewrite across six contributors was judged riskier than dead
credentials warranted.

Three things I would do differently: `.gitignore` the secret file in the first commit of the
repository rather than the fiftieth; never put a real value in a file whose name ends
`.example`; and have any account the system depends on owned by a shared project account
rather than an individual's, so access outlives the handover.

What was done: environment-based configuration with the secret file gitignored,
placeholders-only in the example file, and a code-level guard so that "inference stays
local" is enforced by the code rather than by a config value someone can change.

---

## Two defects in this portfolio copy specifically

- **`neo4j_loader/connection.py` constructs its driver at import time.** Importing the
  module without a `NEO4J_URI` in the environment raises `ConfigurationError`, so the loader
  tests need a placeholder URI set even though they never connect. The driver should be
  constructed lazily.
- **`rag.py` has top-level imports on six modules excluded from this copy** (admin ingest,
  extraction jobs, auth, text-to-speech, and two community-data modules). The file is here
  to be read, not imported. Six test files that import it were removed rather than shipped
  broken.
