# Hypothesis Registry

## Why Freezing Predictions Matters

Retrospective evaluation answers: "Can the model recover what we already know?" Prospective evaluation answers: "Can the model tell us something we don't know yet?"

The difference is not technical. It is epistemic. Retrospective metrics can always be gamed, consciously or not, by tuning toward known outcomes. The only way to prove a prediction system works is to record its outputs before the world reveals the answers.

The hypothesis registry makes this possible. It freezes predictions at a point in time, stamps them with a deterministic fingerprint, and tracks what happens next. Predictions cannot be retroactively edited, added, or removed. Outcomes are appended, never overwritten.

This converts the repurposing system from an ML model into a scientific instrument with a track record.

## What a Slate Is

A **slate** is an immutable snapshot of predictions frozen at a specific moment. Each slate contains:

- A unique identifier (UUID)
- A timestamp (when frozen)
- A deterministic fingerprint (git SHA, model hash, data file hashes, parameters)
- A set of drug-disease predictions with scores, ranks, and confidence tiers
- Metadata about what was known at freeze time for each pair

A slate is the unit of prospective evaluation. All metrics are computed per-slate. Slates are never modified after creation.

### Fingerprint

Each slate records enough metadata to determine whether it could be reproduced:

| Field | Purpose |
|-------|---------|
| `git_sha` | Exact code version |
| `best_model_hash` | Hash of the model configuration file |
| `top_candidates_hash` | Hash of the scored candidates file |
| `top_candidates_size` | File size in bytes |
| `top_candidates_mtime` | File modification time |
| `parameters` | Exact freeze parameters (top_k, min_score, diseases) |

If any of these change between slates, the predictions are from a different system state.

## Why Falsification Criteria Are Required

Every prediction carries an auto-generated falsification criterion: a concrete, time-bounded statement of what would disprove it. This is not decoration. It forces the system to make specific, testable claims rather than vague assertions of "potential."

### Templates

| Tier | Known status | Criterion |
|------|-------------|-----------|
| High | Unknown | "{drug} enters Phase 2+ trial for {disease} within 3 years, OR receives FDA approval" |
| Moderate | Unknown | "{drug} enters Phase 1+ trial for {disease} within 5 years" |
| Exploratory | Unknown | "No clinical activity for {drug} in {disease} within 5 years falsifies this prediction" |
| Any | Approved | "Already approved; monitoring for label expansion or new formulation for {disease}" |
| Any | In trials | "Currently Phase {N}; advances to next phase within 3 years" |

High-tier predictions have shorter time horizons and stricter success criteria. This reflects the system's confidence: if a drug is ranked in the top 10 for a disease, something should happen within 3 years. If nothing does, the prediction was wrong.

## Why Outcomes Are Tracked Even When Negative

The registry tracks all detectable outcomes, not just successes:

- **Trial completions** confirm a hypothesis was tested
- **Trial terminations and withdrawals** indicate a hypothesis was tested and failed
- **Phase advances** indicate a hypothesis is gaining evidence
- **No activity** (the absence of outcomes) is itself informative after sufficient time

Tracking negative outcomes prevents survivorship bias. A system that only reports hits is not a scientific instrument.

## Example: Cross-Disease Slate #1

**Frozen**: 2026-02-09T21:02:56Z
**Model**: v5_rescue_features (1,039 features, LGBMRanker)
**Diseases**: 7 (type 2 diabetes, glioblastoma, ulcerative colitis, ALS, SLE, IPF, TNBC)
**Predictions**: 350 (top 50 per disease, min score 0.3)
**Fingerprint**: git `dceb02b`, model hash `c3703da0dc450a1f`

### Example prediction entry

```
disease_name:         Glioblastoma
drug_name:            pembrolizumab
repurposing_score:    0.917
rank:                 3
confidence_tier:      high
known_at_freeze:      {"in_labeled_pairs": true, "label": "phase2", "max_phase": 2.0}
falsification:        "Currently Phase 2; advances to next phase within 3 years"
```

This entry records that on 2026-02-09, the model ranked pembrolizumab #3 for glioblastoma with a score of 0.917, and that at freeze time ChEMBL already showed Phase 2 activity. The falsification criterion is specific: Phase 3 or higher within 3 years.

### Example outcome entry

```
disease_name:         Glioblastoma
drug_name:            pembrolizumab
outcome_type:         phase_advanced
outcome_source:       chembl
outcome_detail:       {"old_phase": 0, "new_phase": 2.0, "efo_term": "glioblastoma multiforme"}
observed_at:          2026-02-09T21:03:02Z
```

This outcome was detected during the initial ingestion run. It reflects ChEMBL data that existed at freeze time (a retrospective signal, not a prospective one). True prospective outcomes will accumulate over subsequent months as new ChEMBL releases, FDA approvals, and CT.gov registrations appear.

### Example falsification criterion (exploratory tier)

```
disease_name:         amyotrophic lateral sclerosis
drug_name:            tocilizumab
rank:                 47
confidence_tier:      moderate
falsification:        "tocilizumab enters Phase 1+ trial for amyotrophic lateral sclerosis within 5 years"
```

If no clinical trial for tocilizumab in ALS is registered by February 2031, this prediction is falsified. That is a concrete, checkable claim.

## Interpreting Time-to-Event

The registry computes median time-to-event for each slate: the number of days between slate creation and the first high-signal outcome for each drug-disease pair.

**Negative or zero time-to-event** values occur when the detected outcome (e.g., a trial start date) predates the slate freeze date. This happens because the initial outcome ingestion picks up historical events that already existed in the data. These are retrospective signals, not prospective predictions coming true.

**Positive time-to-event** values represent genuine prospective outcomes: events that occurred after the slate was frozen. These are the scientifically meaningful measurements.

As slates age, the proportion of positive time-to-event values will increase, and the median will shift from negative/zero toward positive. The rate of this shift, and how it varies by confidence tier, is the core metric of prospective performance.

## Data Schema

### Registry (`registry.parquet`)

| Column | Type | Description |
|--------|------|-------------|
| slate_id | VARCHAR | UUID identifying the freeze operation |
| created_at | VARCHAR | ISO timestamp of freeze |
| disease_name | VARCHAR | Original disease string |
| disease_canonical | VARCHAR | Lowercase, whitespace-normalized |
| disease_variant_group_id | VARCHAR | SHA-256 prefix for grouping variants |
| disease_ontology_id | VARCHAR | EFO/MONDO ID from labeled pairs (nullable) |
| drug_name | VARCHAR | Drug name |
| chembl_id | VARCHAR | Stable ChEMBL identifier (nullable) |
| repurposing_score | DOUBLE | Model score at freeze time |
| rank | INT32 | Within-disease rank (1 = top) |
| confidence_tier | VARCHAR | Rank-based: high/moderate/exploratory |
| confidence_tier_score | VARCHAR | Score-based: high/moderate/exploratory |
| model_version | VARCHAR | From best_model.json |
| model_config | VARCHAR (JSON) | Feature count, hyperparameters |
| known_at_freeze | VARCHAR (JSON) | What was known: label, max_phase, first_approval |
| evidence_summary_status | VARCHAR | "not_computed" (lazy) or "computed" |
| evidence_summary | VARCHAR (JSON) | Path counts by type (computed on demand) |
| falsification_criteria | VARCHAR | Auto-generated testable claim |

### Outcomes (`outcomes.parquet`)

| Column | Type | Description |
|--------|------|-------------|
| slate_id | VARCHAR | FK to registry |
| disease_name | VARCHAR | Disease |
| drug_name | VARCHAR | Drug |
| outcome_type | VARCHAR | first_trial_seen / phase_advanced / fda_approved / status_transition |
| outcome_date | VARCHAR | When the event occurred |
| outcome_source | VARCHAR | chembl / fda / ctgov |
| outcome_detail | VARCHAR (JSON) | Source-specific details |
| observed_at | VARCHAR | When we detected this |

See [OUTCOME_TAXONOMY.md](OUTCOME_TAXONOMY.md) for definitions of each outcome type.
