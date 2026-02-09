# Metrics

Precise definitions for all evaluation metrics used in this system. Each metric includes its formula, interpretation, and known limitations.

## Retrospective Metrics

These are computed during model development using historical data. They establish a performance floor but do not prove prospective generalization.

### Hit@K (Leave-One-Out)

**What it measures**: If we hide one known treatment for a disease and re-rank all candidates, does the hidden drug appear in the top K?

**Procedure**:
1. For each disease with >= 10 labeled pairs, iterate over each positive (approved) drug
2. Remove that drug from the training set
3. Re-score all candidate drugs for that disease
4. Check if the removed drug appears in the top K predictions

**Formula**:
```
Hit@K = (number of diseases where hidden drug ranks <= K) / (total leave-one-out trials)
```

**Current value**: Hit@50 = 0.2525 (v5 ranker, 3,140 disease groups)

**Interpretation**: 25.25% of known treatments, when hidden, are recovered in the top 50. This is an optimistic estimate of prospective performance because hidden drugs were known positives with rich feature profiles.

**Limitations**: Does not test the system's ability to find truly novel candidates (drugs with no prior indication for the disease). Over-represents well-characterized drugs.

### NDCG@K

**What it measures**: Quality of the ranking order within each disease group, weighted by relevance grade.

**Formula**:
```
DCG@K  = sum_{i=1}^{K} (2^{rel_i} - 1) / log2(i + 1)
IDCG@K = DCG@K for the ideal (perfectly sorted) ranking
NDCG@K = DCG@K / IDCG@K
```

Where `rel_i` is the graded relevance label:
- Approved (label=4): gain = 2^4 - 1 = 15
- Phase 3 (label=3): gain = 2^3 - 1 = 7
- Phase 2 (label=2): gain = 2^2 - 1 = 3
- Phase 1 (label=1): gain = 2^1 - 1 = 1
- Negative (label=0): gain = 0

**Current value**: NDCG@50 = 0.9436 (validation set)

**Interpretation**: Within disease groups, the ranking places high-relevance drugs near the top. A value of 0.94 means the ranking is close to optimal within each group.

### AUC (Area Under ROC Curve)

**What it measures**: Global ability to separate positive from negative drug-disease pairs across all diseases.

**Formula**: Standard ROC AUC — probability that a randomly chosen positive pair scores higher than a randomly chosen negative pair.

**Current value**: Test AUC = 0.9686 (v5 ranker)

**Why it is not the primary metric**: AUC is computed globally across all diseases. Features that improve global discrimination (drug popularity, disease-level constants) can hurt within-disease ranking. A model with AUC 0.99 could still rank the wrong drugs at the top for any individual disease. Hit@50 and NDCG@50 are more operationally relevant.

### Temporal AUC

**What it measures**: Out-of-time generalization. Train the model on approvals before a cutoff year, predict approvals after.

**Procedure**:
1. Train on drug-disease pairs with approval year <= 2015
2. Test on pairs with approval year 2016-2024
3. Compute standard AUC on the test set

**Current value**: Temporal AUC = 0.9910 (per-year values all > 0.95)

**Interpretation**: The model's learned biological signals generalize across time. High temporal AUC does not guarantee zero data leakage (disease-level features may use full-dataset statistics), but it rules out gross temporal confounding.

## Prospective Metrics

These are computed on frozen slates after real-world outcomes are observed. They are the scientifically meaningful measurements.

### Enrichment vs. Random

**What it measures**: Are predictions in a given tier more likely to show real-world activity than a random selection from the same diseases?

**Formula**:
```
hit_rate(tier) = (pairs in tier with >= 1 high-signal outcome) / (total pairs in tier)

overall_hit_rate = (all pairs with >= 1 high-signal outcome) / (all pairs in slate)

enrichment_vs_random(tier) = hit_rate(tier) / overall_hit_rate
```

High-signal outcomes: `first_trial_seen`, `phase_advanced`, `fda_approved` (see [OUTCOME_TAXONOMY.md](OUTCOME_TAXONOMY.md)).

**Interpretation**: An enrichment of 1.5x for the high tier means top-10 predictions are 50% more likely to show activity than a randomly chosen prediction from the same slate. Values below 1.0 indicate the tier performs worse than random.

**Limitations**: Does not control for drug popularity. Well-characterized drugs appear more often in trials and are easier to rank highly. Use enrichment vs. popularity (below) as the primary metric.

### Enrichment vs. Popularity-Matched Baseline

**What it measures**: Are predictions more likely to show activity than drugs with equivalent indication breadth (number of existing indications)?

**Procedure**:
1. Compute indication breadth for each drug: count of distinct diseases in the labeled pairs dataset
2. Assign each drug to a breadth decile (NTILE(10) by indication count)
3. For each decile, compute the baseline hit rate across all scored candidates for the slate's diseases (not just slate predictions)
4. For each prediction in the slate, look up its decile's baseline hit rate
5. Compute the expected number of hits if the slate had average-popularity drugs: `sum(decile_baseline_rate * n_predictions_in_decile)`
6. Compare to observed hits

**Formula**:
```
expected_hit_rate = sum over deciles of (n_slate_predictions_in_decile * baseline_hit_rate_for_decile) / n_total_predictions

observed_hit_rate = n_slate_predictions_with_outcome / n_total_predictions

enrichment_vs_popularity = observed_hit_rate / expected_hit_rate
```

**Current value**: 1.70x (high tier, cross-disease slate #1)

**Interpretation**: High-tier predictions are 1.70 times more likely to show real-world activity than drugs with comparable indication breadth. This means the model is selecting the *right* popular drugs for each disease, not just popular drugs in general.

**Why this is the primary prospective metric**: It answers the hardest question: "Is the model doing something beyond picking well-known drugs?" A popularity-naive model that simply ranks drugs by number of existing indications would achieve enrichment_vs_popularity = 1.0.

### Precision Proxy

**What it measures**: Among all prediction-outcome pairs (not just high-signal), what fraction are high-signal?

**Formula**:
```
precision_proxy = (pairs with >= 1 high-signal outcome) / (pairs with any outcome)
```

**Interpretation**: If precision_proxy is high, most detected real-world activity is meaningful (trials, phase advances, approvals) rather than noise (status transitions). If low, the outcomes are dominated by administrative changes.

**Limitation**: This is a proxy, not true precision, because we cannot observe the full set of real-world events (only those captured in our data sources).

### Score Calibration

**What it measures**: Do higher-scoring predictions accumulate outcomes at a higher rate?

**Formula**:
```
mean_score_hits   = mean(repurposing_score) for pairs with >= 1 high-signal outcome
mean_score_misses = mean(repurposing_score) for pairs with no high-signal outcome
```

**Current value**: Hits = 0.917, Misses = 0.843 (cross-disease slate #1)

**Interpretation**: Predictions that eventually show real-world activity have higher model scores on average. This confirms the score is directionally correct as an ordering metric. It does not imply the score is a calibrated probability.

### Time-to-Event

**What it measures**: Median number of days between slate freeze and first high-signal outcome for each drug-disease pair.

**Formula**:
```
time_to_event(pair) = date(first high-signal outcome for pair) - date(slate created_at)

median_time_to_event = median(time_to_event) across all pairs with outcomes
```

**Interpretation**:
- **Negative values** mean the outcome event predates the slate freeze. These are retrospective signals detected during initial ingestion, not prospective predictions coming true.
- **Zero** means the outcome coincided with the freeze date (within the resolution of outcome_date).
- **Positive values** are genuine prospective outcomes. These are the scientifically meaningful measurements.

As a slate ages, the median time-to-event should shift positive. The rate of this shift, stratified by tier, indicates how quickly the model's predictions materialize into real-world activity.

**Limitation**: Time-to-event is bounded below by data refresh frequency. If ChEMBL updates quarterly, the minimum detectable positive time-to-event is ~90 days.
