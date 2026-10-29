# Prompt experiments

The harnesses behind the 17 logged extraction runs in the main README's evaluation section.
They run extraction against a paper and write a CSV, without touching the database, so a
prompt change can be scored before it goes anywhere near the production path.

| File | What it does |
|---|---|
| `test_extraction_prompt_config.py` | Runs one named prompt from `prompts/` against one page. The harness used for the A/B/C comparison. Unlike the others it always preserves the model's `SENTENCE REF` line into a `sentence_ref` column, which the grounding and quote gates both read |
| `test_extraction_variants.py` | Runs a prompt variant against a page and writes a comparable CSV. Used for the four-round baseline-versus-variant rounds |
| `test_extraction.py` | Runs the current baseline prompt, for a reference extraction to compare against |
| `run_full_paper_variantA.py` | Runs variant A across a whole paper rather than a single page |

Despite the `test_` prefix these are **manual harnesses, not unit tests** — `pytest` collects
nothing from them. The automated suite is in `tests/`.

Run from the repository root:

```bash
python -m experiments.test_extraction_prompt_config papers/<paper>.pdf 3 \
    --prompt extraction_strict_tacit_v1
```

Then score the resulting CSV:

```bash
python -m eval.run_eval --csv output/<the_csv> --label "strict tacit, page 3"
```

## What these measured

Four configurations, four rounds each. The headline finding was not which prompt won — it
was that the same prompt, on the same page, at `temperature` 0.1, produced 8, 7, 13 and then
1 true positives across four runs. Any single-run prompt comparison on this pipeline is
noise. The main README has the full table.
