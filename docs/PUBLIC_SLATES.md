# Public Slates

Track record of frozen prediction slates. Each entry records the state at freeze time and accumulates outcomes over time. This document is updated after each outcome ingestion run.

---

## Slate #0: TNBC Pilot

| Field | Value |
|-------|-------|
| Slate ID | `302b34f7-1053-483a-bf12-c68109990cea` |
| Frozen | 2026-02-09T20:57:03Z |
| Model | v5_rescue_features (1,039 features, LGBMRanker) |
| Diseases | 1: triple negative breast cancer |
| Predictions | 20 (top 20, min score 0.3) |
| Git SHA | `dceb02b` |
| Model hash | `c3703da0dc450a1f` |
| Data hash | `279103d239742f46` |

**Purpose**: Initial proof-of-concept. TNBC was chosen because it is an active research area with frequent trial registrations, making it likely to accumulate outcomes quickly. This slate is not used for cross-disease performance claims.

### Outcome Summary (as of 2026-02-09)

| Outcome Type | Count |
|-------------|-------|
| first_trial_seen | 3 |
| phase_advanced | 2 |
| status_transition | 3 |
| **Total** | **8** |

**Note**: All outcomes detected during initial ingestion reflect pre-existing data, not prospective predictions materializing. True prospective outcomes will be identifiable by positive time-to-event values in future ingestion runs.

---

## Slate #1: Cross-Disease Prospective

| Field | Value |
|-------|-------|
| Slate ID | `469bb270-a423-4929-96bd-6eaadeb7986b` |
| Frozen | 2026-02-09T21:02:56Z |
| Model | v5_rescue_features (1,039 features, LGBMRanker) |
| Diseases | 7 (see below) |
| Predictions | 350 (top 50 per disease, min score 0.3) |
| Git SHA | `dceb02b` |
| Model hash | `c3703da0dc450a1f` |
| Data hash | `279103d239742f46` |

**Diseases included**:

| Disease | Trial Activity | Rationale |
|---------|---------------|-----------|
| Type 2 diabetes | High (2,413 trials) | Crowded therapeutic area |
| Glioblastoma | Moderate (1,247 trials) | High unmet need, active research |
| Ulcerative colitis | Moderate (891 trials) | Autoimmune with emerging biologics |
| Amyotrophic lateral sclerosis | Moderate (624 trials) | Neurodegenerative, few treatments |
| Systemic lupus erythematosus | Moderate (712 trials) | Complex autoimmune |
| Idiopathic pulmonary fibrosis | Low (438 trials) | Rare, limited options |
| Triple negative breast cancer | High (1,892 trials) | Active oncology pipeline |

**Purpose**: First scientifically meaningful slate. Spans high, moderate, and low trial-activity regimes to test whether the model generalizes across disease contexts.

### Outcome Summary (as of 2026-02-09)

| Outcome Type | Count |
|-------------|-------|
| first_trial_seen | 21 |
| phase_advanced | 39 |
| status_transition | 42 |
| **Total** | **102** |

### Performance Metrics (as of 2026-02-09)

| Metric | Value |
|--------|-------|
| High-tier enrichment vs. random | 1.52x |
| High-tier enrichment vs. popularity-matched | **1.70x** |
| Score calibration (mean score hits) | 0.917 |
| Score calibration (mean score misses) | 0.843 |
| Mean indication breadth (hits) | 176.6 |
| Mean indication breadth (misses) | 107.9 |

**Interpretation**: High-tier predictions are 1.70x more likely to show real-world clinical activity than drugs with equivalent indication breadth. The model selects the right popular drugs for each disease, not just popular drugs in general. Mean hit breadth > mean miss breadth confirms the model picks genuinely active drugs rather than being fooled by popularity.

**Caveat**: Day-zero measurement. All detected outcomes are from pre-existing data overlap (retrospective signals). The 1.70x enrichment is real — these drugs do have more clinical activity — but the temporal direction is backward. True prospective validation requires re-running outcome ingestion after new ChEMBL/FDA/CT.gov data releases (target: quarterly).

---

## Planned Slates

### Slate #2: Cross-Disease Follow-Up

| Field | Target |
|-------|--------|
| Planned freeze | ~May 2026 (~90 days after Slate #1) |
| Model | v5_rescue_features (same, unless retrained) |
| Diseases | Same 7 as Slate #1, possibly expanded |
| Purpose | Measure temporal enrichment accumulation; compare day-0 vs day-90 outcomes |

Slate #2 will use the same model as Slate #1. If the model is retrained before then, both slates will be tracked independently. Comparing same-model slates at different freeze dates tests whether the system's predictions are durable or sensitive to data drift.

---

## Reading This Document

- **Outcome counts will increase** after each ingestion run. The total at any point represents all events detected through the most recent data refresh.
- **Performance metrics will change** as the denominator (pairs with any outcome) and numerator (pairs with high-signal outcomes) evolve.
- **Positive time-to-event values** appearing in future updates are the scientifically important signal. They represent events that occurred after the slate was frozen.
- **No retroactive edits** to frozen slates. If predictions change due to model updates, a new slate is frozen. Old slates continue tracking outcomes against their original predictions.
