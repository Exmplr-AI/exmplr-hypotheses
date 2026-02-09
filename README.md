# Exmplr Hypotheses

A prospective hypothesis registry for evaluating AI-generated drug repurposing predictions over time.

Predictions are frozen before outcomes are observed, tracked against real-world clinical trial registrations, phase advances, and regulatory approvals, and published with full methodology and failure analysis.

Quick start page: [`index.html`](index.html)

## Web page

A minimal public landing page is available at [`index.html`](index.html).  
GitHub Pages deployment is automated via [`.github/workflows/deploy-pages.yml`](.github/workflows/deploy-pages.yml).
The page includes an in-browser disease query tool backed by [`slates/2026-Q1/query_data.json`](slates/2026-Q1/query_data.json).

To publish:
1. In GitHub repository settings, set **Pages** source to **GitHub Actions**.
2. Push to `main`.
3. The workflow deploys the site and reports the live URL.

## Current status

| Slate | Frozen | Diseases | Predictions | Enrichment vs. popularity | Status |
|-------|--------|----------|-------------|--------------------------|--------|
| [Q1 2026](slates/2026-Q1/) | 2026-02-09 | 7 | 350 | 1.70x (high tier) | Aging |

Next update: ~May 2026 (after ChEMBL 35 + CT.gov refresh).

## What this repository contains

**Frozen predictions** — Immutable snapshots of ranked drug-disease hypotheses with scores, confidence tiers, and falsification criteria. Stored as Parquet files in [`slates/`](slates/).

**Tracked outcomes** — Real-world events (new trials, phase advances, approvals, failures) detected by comparing frozen predictions against ChEMBL, ClinicalTrials.gov, and FDA data. Updated quarterly.

**Methodology** — How the system works, how it is evaluated, what it cannot do, and what would prove it wrong. See [`docs/`](docs/).

## Documentation

| Document | Purpose |
|----------|---------|
| [SYSTEM.md](docs/SYSTEM.md) | Methods, model overview, failure modes, 12-month success bar |
| [SCOREBOARD.md](docs/SCOREBOARD.md) | Quarterly results — the living record |
| [HYPOTHESIS_REGISTRY.md](docs/HYPOTHESIS_REGISTRY.md) | Why predictions are frozen and how they are tracked |
| [METRICS.md](docs/METRICS.md) | Precise definitions and formulas for all evaluation metrics |
| [OUTCOME_TAXONOMY.md](docs/OUTCOME_TAXONOMY.md) | What counts as an outcome and what does not |
| [REPLICATION.md](docs/REPLICATION.md) | How to independently verify results |

## Case studies

- [ALS walkthrough](docs/examples/ALS_slate_walkthrough.md) — What the model predicted, what happened, and what would falsify it

## Releases

- [Q1 2026 — Cross-disease slate](releases/2026_Q1_cross_disease_slate.md)

## Key principles

**Predictions are hypotheses, not claims.** Scores represent relative ranking within a disease, not probability of approval.

**Frozen means frozen.** Predictions are never edited, added, or removed after the freeze date. Outcomes are appended, never overwritten.

**Failures are published.** Misses, negative outcomes, and falsified predictions are first-class data. They are tracked and reported alongside successes.

**Popularity is controlled for.** Enrichment metrics compare predictions against drugs with equivalent indication breadth, not against random. This prevents the trivial strategy of picking well-known drugs.

## Introduction

- [**We Froze 350 AI Drug Predictions. Now We Wait.**](drafts/01_introducing_hypothesis_registry.md) — Why we're publishing hypotheses before we know if they're right.

## Reading the data

Run DuckDB from the repository root. Paths below are relative to the repo root.

```python
import duckdb

conn = duckdb.connect()

# List all predictions for a disease
conn.execute("""
    SELECT drug_name, repurposing_score, rank, confidence_tier, falsification_criteria
    FROM read_parquet('slates/2026-Q1/registry.parquet')
    WHERE LOWER(disease_name) LIKE '%glioblastoma%'
    ORDER BY rank
""").fetchdf()

# List all outcomes
conn.execute("""
    SELECT disease_name, drug_name, outcome_type, outcome_date, outcome_source
    FROM read_parquet('slates/2026-Q1/outcomes.parquet')
    ORDER BY disease_name
""").fetchdf()
```

Requires [DuckDB](https://duckdb.org/) (Python: `pip install duckdb`).

## What this system does not claim

- It does not claim these drugs will be approved
- It does not claim superiority over any other system
- It does not replace clinical judgment or safety evaluation
- It does not predict which specific trials will succeed

It claims only that these predictions are frozen, falsifiable, and tracked honestly over time.

## License

Data and documentation are provided for research and evaluation purposes.
