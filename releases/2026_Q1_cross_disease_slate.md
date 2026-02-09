# Exmplr Prospective Slate — Q1 2026

*Frozen: 2026-02-09 | Model: v5_rescue_features | Slate ID: `469bb270`*

---

## What this is

On February 9, 2026, we froze 350 drug repurposing predictions across 7 diseases. Each prediction is a hypothesis: this drug may show future clinical activity for this disease. The predictions are ranked, timestamped, and immutable.

We will track these predictions against real-world outcomes (new clinical trials, phase advances, FDA approvals) and publish the results quarterly. We will not edit, add, or remove predictions after the fact.

This is the first public slate. It establishes a baseline. Most of the value will come from what happens over the next 12 months.

## Diseases

| Disease | Why included |
|---------|-------------|
| Type 2 diabetes | High trial activity (~2,400 trials). Crowded. Hard to beat popularity. |
| Glioblastoma | Moderate activity (~1,200 trials). High unmet need. |
| Ulcerative colitis | Moderate activity (~900 trials). Emerging biologics. |
| Amyotrophic lateral sclerosis | Moderate activity (~620 trials). Very few approved treatments. |
| Systemic lupus erythematosus | Moderate activity (~710 trials). Complex autoimmune. |
| Idiopathic pulmonary fibrosis | Low activity (~440 trials). Limited therapeutic options. |
| Triple negative breast cancer | High activity (~1,900 trials). Active oncology pipeline. |

The diseases span high, moderate, and low clinical trial activity. This tests whether the model generalizes or only works where data is abundant.

## Predictions

50 predictions per disease, 350 total. Each prediction includes:
- A repurposing score (relative ranking, not a probability)
- A within-disease rank
- A confidence tier (high: rank 1-10, moderate: rank 11-50)
- A falsification criterion (what would disprove this prediction, with a time horizon)
- A record of what was known at freeze time (existing approvals, trial phases)

The full prediction set is available in `registry.parquet`. See [HYPOTHESIS_REGISTRY.md](../HYPOTHESIS_REGISTRY.md) for schema details.

## Day-zero outcomes

We ran outcome detection immediately after freezing. This picks up historical signals — events that already existed in ChEMBL, FDA, and ClinicalTrials.gov at the time of the freeze.

| Outcome type | Count | Signal level |
|-------------|-------|-------------|
| first_trial_seen | 21 | High |
| phase_advanced | 39 | High |
| status_transition | 42 | Low |
| **Total** | **102** | |

These are retrospective signals, not prospective predictions coming true. They confirm the model's ranking aligns with historical clinical activity. They do not prove the model can predict the future.

## Day-zero performance

| Metric | Value |
|--------|-------|
| High-tier hit rate | — |
| Enrichment vs. random (high tier) | 1.52x |
| Enrichment vs. popularity-matched baseline (high tier) | **1.70x** |
| Score calibration: mean score (hits) | 0.917 |
| Score calibration: mean score (misses) | 0.843 |
| Mean indication breadth (hits) | 176.6 |
| Mean indication breadth (misses) | 107.9 |

**What the 1.70x means**: The model's top-10 predictions per disease are 1.70 times more likely to have real-world clinical activity than drugs with equivalent indication breadth (number of existing approved indications). This controls for the trivial strategy of picking well-known drugs.

**What the 1.70x does not mean**: It does not mean the model is 70% better at finding cures. It means, at day zero with retrospective data, the model selects drugs that are more clinically active than their popularity alone would predict.

## What happens next

- **Q2 2026 (~May)**: Rerun outcome ingestion after ChEMBL 35 release and CT.gov refresh. Publish updated outcome counts and metrics. Freeze Slate #2 for the same diseases.
- **Q3 2026 (~August)**: Second outcome update. First measurement of genuinely prospective signals (positive time-to-event values).
- **Q1 2027 (12-month mark)**: Full assessment against the [12-month success bar](../SYSTEM.md#f-12-month-success-bar). Publish whether the system meets, misses, or fails each criterion.

## What we do not claim

- We do not claim these drugs will be approved for these diseases
- We do not claim the 1.70x enrichment will hold prospectively
- We do not claim this system replaces clinical judgment, mechanism-of-action studies, or safety evaluation
- We do not claim superiority over any other system (we have not compared)

We claim only that these predictions are frozen, falsifiable, and will be tracked honestly over time.

## How to verify

See [REPLICATION.md](../REPLICATION.md) for step-by-step verification of slate fingerprints, outcome data, and enrichment calculations.

## Detailed case study

See [ALS walkthrough](../examples/ALS_slate_walkthrough.md) for a full, honest analysis of what the model predicted for ALS, what happened, and what would falsify the predictions.

---

*Model: LightGBM Ranker (LambdaRank), 1,039 features, LOO Hit@50 = 0.2525*
*Data sources: ChEMBL, OpenTargets, STRING, ClinicalTrials.gov, FDA, DrugCentral, BindingDB*
*Methodology: [SYSTEM.md](../SYSTEM.md) | Metrics: [METRICS.md](../METRICS.md) | Outcome definitions: [OUTCOME_TAXONOMY.md](../OUTCOME_TAXONOMY.md)*
