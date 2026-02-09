# Replication Guide

How to independently verify predictions, slates, and enrichment metrics.

This is not a setup guide. It assumes you have access to the codebase and data directory. The goal is conceptual transparency: what would you need to check, and how?

---

## 1. Verify a Slate Was Not Modified

Every slate has a deterministic fingerprint stored in `slates.json` and embedded in `registry.parquet`. To verify a slate is authentic:

```bash
# Read the slate manifest
cat api/data/derived/repurposing/hypothesis_registry/slates.json | python3 -m json.tool
```

Each entry contains:

| Field | What to check |
|-------|--------------|
| `git_sha` | `git log --oneline -1 dceb02b` — confirm this commit exists and predates the slate |
| `best_model_hash` | `python3 -c "import hashlib; print(hashlib.md5(open('api/data/derived/repurposing/models/best_model.json','rb').read()).hexdigest()[:16])"` |
| `top_candidates_hash` | Same hash method on `top_candidates.parquet` |
| `top_candidates_size` | `stat -c %s api/data/derived/repurposing/top_candidates.parquet` |
| `top_candidates_mtime` | `stat -c %Y.%N api/data/derived/repurposing/top_candidates.parquet` |

If any hash differs from what's recorded, the slate was frozen against different data than what currently exists. This is expected if the model has been retrained since the freeze — the fingerprint records the state at freeze time, not the current state.

## 2. Rerun a Freeze

To reproduce a slate's predictions (assuming the data and model match the fingerprint):

```bash
python api/etl/scripts/freeze_hypothesis_slate.py \
  --diseases "type 2 diabetes,Glioblastoma,Ulcerative colitis,amyotrophic lateral sclerosis,Systemic lupus erythematosus,Idiopathic pulmonary fibrosis,triple negative breast cancer" \
  --top-k 50 \
  --min-score 0.3 \
  --description "Replication of cross-disease slate #1"
```

This creates a new slate with a new UUID. The predictions should be identical to the original if the data and model are unchanged. Compare:

```bash
python3 -c "
import duckdb
conn = duckdb.connect()
# Replace NEW_SLATE_ID with the UUID from the replication run
original = '469bb270-a423-4929-96bd-6eaadeb7986b'
replica  = 'NEW_SLATE_ID'
path = 'api/data/derived/repurposing/hypothesis_registry/registry.parquet'
diff = conn.execute(f\"\"\"
    SELECT o.disease_name, o.drug_name, o.rank as orig_rank, r.rank as new_rank,
           o.repurposing_score as orig_score, r.repurposing_score as new_score
    FROM read_parquet('{path}') o
    JOIN read_parquet('{path}') r
      ON LOWER(o.disease_name) = LOWER(r.disease_name)
     AND LOWER(o.drug_name) = LOWER(r.drug_name)
    WHERE o.slate_id = '{original}' AND r.slate_id = '{replica}'
      AND (o.rank != r.rank OR ABS(o.repurposing_score - r.repurposing_score) > 0.0001)
\"\"\").fetchall()
print(f'Differences: {len(diff)}')
for d in diff[:10]:
    print(f'  {d[0]} / {d[1]}: rank {d[2]}->{d[3]}, score {d[4]:.4f}->{d[5]:.4f}')
"
```

Zero differences means the slate is reproducible.

## 3. Independently Confirm Outcomes

Outcomes are detected by joining frozen predictions against external data sources. To verify:

### ChEMBL phase advances

```bash
python3 -c "
import duckdb, json
conn = duckdb.connect()
# Check a specific drug-disease pair against ChEMBL
rows = conn.execute(\"\"\"
    SELECT drug_name, max_phase_for_ind, efo_term, mesh_heading
    FROM read_parquet('api/data/sources/external/chembl/chembl_indications_full.parquet')
    WHERE LOWER(drug_name) = 'escitalopram'
      AND (LOWER(efo_term) LIKE '%amyotrophic%' OR LOWER(mesh_heading) LIKE '%amyotrophic%')
\"\"\").fetchall()
for r in rows:
    print(f'{r[0]}: max_phase={r[1]}, efo={r[2]}, mesh={r[3]}')
"
```

### CT.gov trials

```bash
python3 -c "
import duckdb
conn = duckdb.connect()
# Check a specific NCT ID from an outcome
rows = conn.execute(\"\"\"
    SELECT nct_id, phase, overall_status, start_date, primary_completion_date
    FROM read_parquet('api/data/trials.parquet')
    WHERE nct_id = 'NCT00965497'
\"\"\").fetchall()
for r in rows:
    print(f'{r[0]}: phase={r[1]}, status={r[2]}, start={r[3]}, completion={r[4]}')
"
```

Every outcome in `outcomes.parquet` includes an `outcome_detail` JSON field with source-specific identifiers (NCT IDs, ChEMBL terms, FDA application numbers). These are traceable back to the raw data.

### Rerun outcome ingestion

```bash
python api/etl/scripts/ingest_hypothesis_outcomes.py
```

This is idempotent. Running it again will find zero new outcomes (all existing outcomes are deduplicated on composite key). New outcomes only appear after upstream data refreshes (ChEMBL releases, CT.gov updates, FDA data pulls).

## 4. Recompute Enrichment Metrics

Performance metrics are not cached. They are computed live from `registry.parquet` and `outcomes.parquet`:

```bash
curl http://localhost:8000/api/repurposing/hypotheses/469bb270-a423-4929-96bd-6eaadeb7986b/performance \
  -H "Authorization: Bearer $TOKEN"
```

Or directly via Python:

```bash
python3 -c "
import sys; sys.path.insert(0, '.')
from api.services.repurposing_queries import get_slate_performance
result = get_slate_performance('469bb270-a423-4929-96bd-6eaadeb7986b')
import json; print(json.dumps(result, indent=2, default=str))
"
```

The computation is deterministic given the same registry and outcomes data. To verify the popularity-matched baseline independently:

1. Load `labeled_pairs.parquet`, count distinct diseases per drug (= indication breadth)
2. Assign drugs to breadth deciles (NTILE(10))
3. For each decile, compute the hit rate across all `top_candidates.parquet` pairs for the slate's diseases
4. Weight by the slate's decile distribution
5. Compare expected hits to observed hits

See [METRICS.md](METRICS.md) for the exact formulas.

## 5. Cross-Check Against Public Sources

The ultimate verification is checking predictions against public databases directly:

| Source | URL | What to check |
|--------|-----|---------------|
| ClinicalTrials.gov | https://clinicaltrials.gov/ | Search drug + disease, check trial phases and dates |
| ChEMBL | https://www.ebi.ac.uk/chembl/ | Search drug, check indication phases |
| FDA@FDA | https://www.accessdata.fda.gov/scripts/cder/daf/ | Search drug, check approval dates and indications |

If an outcome in our system does not match the public source, either our data is stale (expected — data refresh lag) or there is a matching error (report it).

## What You Cannot Replicate

- **The model training process** requires all upstream data sources (ChEMBL, OpenTargets, STRING, DrugCentral, etc.) and takes several hours. Replication of model training is possible but non-trivial.
- **Embedding generation** requires access to Amazon Bedrock (Titan v2). The pre-computed embeddings are included in the data directory.
- **Historical data states** are not versioned. If ChEMBL updates between two outcome ingestion runs, the intermediate state is lost. Outcomes track what we detected, not necessarily what existed at every point in time.
