# Cross-Disease Slate — Q1 2026

Frozen: 2026-02-09 | Slate ID: `469bb270-a423-4929-96bd-6eaadeb7986b`

## Contents

| File | Description | Size |
|------|-------------|------|
| `registry.parquet` | 350 frozen predictions (drug, disease, score, rank, tier, known_at_freeze) | 16 KB |
| `outcomes.parquet` | 102 detected outcomes (trial starts, phase advances, status transitions) | 5 KB |
| `slates.json` | Slate manifest with fingerprint | 2 KB |

## Predictions by disease

| Disease | Predictions | High tier | Score range |
|---------|-------------|-----------|-------------|
| Type 2 diabetes | 50 | 10 | 0.903 - 0.989 |
| Glioblastoma | 50 | 10 | 0.870 - 0.991 |
| Ulcerative colitis | 50 | 10 | 0.911 - 0.991 |
| Amyotrophic lateral sclerosis | 50 | 10 | 0.467 - 0.971 |
| Systemic lupus erythematosus | 50 | 10 | 0.886 - 0.988 |
| Idiopathic pulmonary fibrosis | 50 | 10 | 0.616 - 0.939 |
| Triple negative breast cancer | 50 | 10 | 0.316 - 0.989 |

## Day-zero outcomes

| Type | Count |
|------|-------|
| phase_advanced | 98 |
| first_trial_seen | 1 |
| status_transition | 3 |

All outcomes are retrospective (detected from data existing at freeze time).

## How to read the data

```python
import duckdb
conn = duckdb.connect()

# All predictions for ALS
conn.execute("""
    SELECT drug_name, repurposing_score, rank, confidence_tier
    FROM read_parquet('registry.parquet')
    WHERE LOWER(disease_name) LIKE '%amyotrophic%'
    ORDER BY rank
""").fetchdf()

# All outcomes
conn.execute("""
    SELECT disease_name, drug_name, outcome_type, outcome_source, outcome_date
    FROM read_parquet('outcomes.parquet')
    ORDER BY disease_name, outcome_type
""").fetchdf()
```

## Fingerprint

| Field | Value |
|-------|-------|
| Git SHA | `dceb02bfba091c313e9ac5e53595e49856b7d905` |
| Model hash | `c3703da0dc450a1f` |
| Data hash | `279103d239742f46` |
| Model version | v5_rescue_features |

See [REPLICATION.md](../../docs/REPLICATION.md) for verification steps.
