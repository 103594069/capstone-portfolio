# Operational scripts

Entry points and checks that sit outside the main pipeline. Run them with `python -m` from
the repository root, so the modules they import resolve.

| File | What it does |
|---|---|
| `run_verification_pipeline.py` | Runs the three gates against an existing extraction CSV, without re-extracting. Each gate writes its own flags CSV naming why every rejected row was rejected |
| `run_page_pipeline.py` | Extraction over a single page, for iterating without paying for a whole paper |
| `load_production_graph.py` | Loads a reviewed CSV set into Neo4j, stamping every relationship with a shared run id |
| `smoke_test.py` | Post-deploy check: configuration present, database reachable, inference endpoint answering |
| `dry_run_check.py` | Validation and subject correction over a CSV with no database write |
| `check_subject_specificity.py` | Audits the dependency-parse subject correction against a CSV, to see what it changed and what it left alone |

```bash
python -m scripts.run_verification_pipeline output/<paper>_kg.csv \
    --wellformedness-check --grounding-check --quote-check --pdf papers/<paper>.pdf
```

None of these run as published: the repository has no corpus, no model and no database.
