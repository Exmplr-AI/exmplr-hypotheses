# System Methods

## A. What This System Addresses

This system ranks drug-disease hypotheses under uncertainty. Given a disease, it produces an ordered list of existing drugs most likely to show future clinical activity for that indication, based on multi-source biological evidence.

It is a search-space reduction tool. It does not claim cures, predict approval, or replace clinical judgment. It narrows ~11,000 drugs to a ranked shortlist of 50, so that human experts, lab teams, or trial sponsors can allocate attention more efficiently.

The system was developed retrospectively (trained on known drug-indication approvals through 2024) and is now operating prospectively: predictions are frozen before outcomes are observed, and tracked against real-world events over time.

## B. What the System Outputs

**Ranked hypotheses.** For each disease, the system scores every candidate drug and ranks them. The output is a ranked list, not a probability estimate. A score of 0.85 means "ranked higher than most candidates for this disease," not "85% chance of approval."

**Confidence tiers.** Predictions are binned by within-disease rank:
- **High** (rank 1-10): strongest signal for this disease
- **Moderate** (rank 11-50): plausible candidates worth monitoring
- **Exploratory** (rank 51+): weak or speculative

Tiers are rank-based, not score-based. Score distributions vary across diseases and model versions; rank is stable.

**Frozen slates.** Predictions can be frozen into timestamped, immutable snapshots ("slates") with deterministic fingerprints. Each slate records the model version, data hashes, and git state at freeze time. Slates cannot be retroactively modified.

**Outcome tracking.** Frozen predictions are compared against ChEMBL, FDA, and ClinicalTrials.gov to detect real-world events: new trials started, phase advances, approvals, and terminal status changes. Outcomes are appended, never edited.

All outputs are hypotheses, not claims.

## C. Model Overview

### Inputs

Each drug is represented by a 512-dimensional biological signature constructed from 10 sub-signatures:

| Component | Source | Captures |
|-----------|--------|----------|
| Target binding | ChEMBL, DrugCentral, BindingDB, DTC, UniProt | Which genes the drug acts on |
| PPI neighborhood | STRING (score >= 700) | Network context of drug targets |
| Mechanism | ChEMBL mechanisms | Action type (inhibitor, agonist, etc.) |
| Safety / AE | Clinical trial adverse events | Toxicity profile |
| Outcome | Clinical trial endpoints | What the drug affects clinically |
| Semantic | Amazon Titan text embedding | Drug description similarity |
| Molecular | RDKit descriptors | Physicochemical properties |
| FDA regulatory | FDA submission metadata | Regulatory pathway features |
| Druggability | Tractability scores | Target class feasibility |
| Real-world evidence | Synthae population data | Observed utilization patterns |

Each disease is similarly represented by a 512-dimensional signature built from gene-disease associations, GWAS hits, and clinical phenotype data.

Drug and disease signatures are projected into a shared embedding space via joint PCA (3,072 raw dimensions reduced to 256 shared + 256 entity-specific). Cosine similarity in this space provides a biological plausibility baseline (~0.72 AUC).

### Core model

LightGBM Ranker (v5). LambdaRank objective optimizing NDCG@50 within disease groups. 1,039 features including:
- Shared-space cosine similarity
- Per-target and per-PPI overlap counts
- Disease-disease collaborative filtering (max/mean similarity to diseases with known treatments)
- Disease crowding normalization (log-density of known positives)
- Normalized indication breadth (drug popularity adjusted for disease support)

### What "score" means

The repurposing score is a relative ordering metric, not a probability. Within a single disease, higher scores indicate stronger evidence for clinical relevance. Scores are not comparable across diseases. A score of 0.6 for a rare disease with 5 known treatments is not equivalent to 0.6 for type 2 diabetes with 200 known treatments.

### Why Hit@50

Hit@50 measures: "If we hide one known approved treatment and re-rank, does it appear in the top 50?" This is the most operationally relevant metric because it simulates the actual use case: given a disease, would this system surface a real treatment in its shortlist?

Current Hit@50: **0.2525** (leave-one-out across 3,140 disease groups with >= 10 labeled pairs). This means 25.25% of known treatments, when hidden, reappear in the top 50 predictions. The remaining 74.75% are ranked lower, often because the hidden drug has a unique mechanism not shared by any other treatment for that disease.

Hit@50 is an upper bound on prospective performance because it evaluates against known positives. Prospective hit rates will be lower and are the real test.

## D. Evaluation Philosophy

### Retrospective metrics establish a floor

| Metric | Value | What it tests |
|--------|-------|---------------|
| LOO Hit@50 | 0.2525 | Can the model recover hidden treatments? |
| NDCG@50 | 0.9436 | Is the within-disease ranking well-ordered? |
| Test AUC | 0.9686 | Can the model separate positives from negatives? |
| Temporal AUC | 0.9910 | Train pre-2015, predict 2016-2024 approvals |

Retrospective metrics are necessary but not sufficient. They confirm the model has learned real biological signal but cannot prove it generalizes to truly novel predictions.

### AUC alone is insufficient

AUC measures global discrimination but ignores within-disease ranking. Features that improve AUC (drug popularity, disease-level constants) can hurt Hit@50 because they are useless for ranking within a disease group. This system uses a LambdaRank ranker that optimizes NDCG within groups, not a classifier that optimizes global AUC.

### Prospective metrics are the real test

The hypothesis registry tracks predictions frozen before outcomes are observed. Key prospective metrics:

- **Enrichment vs. random**: Are top-ranked predictions more likely to show real-world activity than a random selection from the same disease?
- **Enrichment vs. popularity**: Are top-ranked predictions more likely to show activity than drugs with equivalent indication breadth? This controls for the trivial strategy of picking well-known drugs.
- **Time-to-event**: How quickly do outcomes appear for top-ranked vs. lower-ranked predictions?
- **Score calibration**: Do higher-scoring predictions accumulate outcomes faster than lower-scoring ones?

Popularity-matched baselines are required for any enrichment claim. Without them, high hit rates may simply reflect that the model picks popular drugs.

### Current prospective result

Cross-disease slate #1 (7 diseases, 350 predictions, frozen 2026-02-09):
- High-tier enrichment vs. popularity-matched baseline: **1.70x**
- Score calibration: mean score for hits = 0.917, misses = 0.843
- This result is preliminary (day-zero outcomes from retrospective data overlap). True prospective signal requires 6-12 months of aging.

## E. Failure Modes

### Crowded diseases

Diseases with many known treatments (> 200 labeled positives) concentrate prediction errors. The model must distinguish subtle differences among many plausible candidates. Misses in crowded diseases are often within 0.01 score margin of the top-50 cutoff. This is the dominant failure mode, not cold-start (sparse diseases).

### Unlabeled positives

The negative set includes drugs that are genuinely effective but not yet approved or trialed for a given disease. These "unlabeled positives" receive low labels during training, teaching the model to suppress them. Any attempt to mine hard negatives from high-scoring candidates worsens this problem because the highest-scoring negatives are disproportionately real repurposing opportunities.

### Popularity bias

Drugs with many existing indications are easier to rank highly because they have richer feature profiles. Without explicit correction, the model favors well-characterized drugs over novel candidates. The v5 model partially addresses this with `norm_indication_breadth` (popularity normalized by disease support), and the evaluation framework uses popularity-matched baselines. But the bias is structural and cannot be fully eliminated without new data types.

### Data lag

ChEMBL, ClinicalTrials.gov, and FDA data are updated on different schedules (quarterly to annually). A drug may enter Phase 2 trials months before the data appears in our pipeline. This creates both false negatives (missed outcomes) and a time-to-event floor (outcomes cannot be detected faster than data refresh cycles).

### Retrospective leakage

The temporal validation split (train pre-2015, test 2016-2024) guards against data leakage, but subtle forms persist: disease-level features computed from the full dataset, drug signatures built from all available data regardless of approval date. The temporal AUC of 0.9910 is reassuringly high but not a guarantee of zero leakage. The hypothesis registry, by freezing predictions before outcomes, is the definitive test.

### What this system cannot do

- It cannot predict which specific trial will succeed
- It cannot estimate probability of FDA approval
- It cannot replace mechanism-of-action studies
- It cannot account for commercial or regulatory considerations
- It cannot detect safety signals that don't appear in historical data

## F. 12-Month Success Bar

These are the criteria by which this system should be judged after 12 months of prospective operation. They are written before the results are known.

### The system works if

After 12 months and at least 2 quarterly outcome ingestion cycles:

1. **High-tier enrichment vs. popularity-matched baseline >= 1.3x** across the full cross-disease slate. This means the model's top-10 predictions are at least 30% more likely to show real-world activity than drugs with equivalent indication breadth. The day-zero value is 1.70x; some regression is expected as retrospective signals wash out and prospective signals accumulate.

2. **Positive enrichment in >= 4 of 7 diseases**. The model should generalize across disease contexts. Failing in 1-2 diseases is acceptable (some diseases may be inherently harder or data-poor). Failing in 4+ means the model only works for a narrow disease subset.

3. **Score calibration holds prospectively**. Mean repurposing score for predictions with high-signal outcomes should exceed mean score for predictions without outcomes. If this relationship inverts or disappears, the score is not a useful ordering metric.

4. **At least 1 genuinely prospective high-signal outcome**. At least one drug-disease pair from the slate should show a new trial, phase advance, or approval that was not present in the data at freeze time (positive time-to-event). Zero prospective outcomes after 12 months means the model has not predicted anything new.

### The system does not work if

Any of the following after 12 months:

1. **Enrichment vs. popularity < 1.0x** across the slate. This means the model performs worse than picking drugs by popularity alone. The model has negative value.

2. **Zero high-signal outcomes across all low-activity diseases** (ALS, IPF, SLE). If the model only shows activity for T2D and TNBC (which have thousands of trials), it is reflecting clinical trial density, not biological insight.

3. **Score calibration inverts**. Predictions without outcomes have higher mean scores than predictions with outcomes. The scoring function is anti-correlated with real-world activity.

4. **Falsification criteria met for > 30% of high-tier predictions** within their stated time horizon. If more than a third of top-10 predictions are falsified, the model's confidence assignment is miscalibrated.

### What the system owes the world after 12 months

- An updated PUBLIC_SLATES.md with prospective outcome counts
- An honest assessment of which criteria were met and which were not
- If the system does not work: a clear statement saying so, with the data to prove it

This is a commitment to transparency, not optimism.
