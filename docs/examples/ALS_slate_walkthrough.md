# ALS Slate Walkthrough

What we predicted, why, what happened, and what would prove us wrong.

---

## Why ALS

Amyotrophic lateral sclerosis has ~624 registered clinical trials. That makes it a moderate-activity disease — enough trial infrastructure to generate detectable outcomes, but not so crowded that our predictions would be trivially obvious.

ALS also has very few approved treatments (riluzole, edaravone, tofersen, AMX0035). Most candidate therapies fail. If the model can identify drugs that eventually show clinical activity in ALS, that is a meaningful signal in a notoriously difficult therapeutic area.

## What we froze

On 2026-02-09, we froze the top 50 predictions for ALS from the v5 ranker (slate `469bb270`). Every prediction was scored, ranked, and assigned a confidence tier based on within-disease rank.

### Top 10 predictions (high tier)

| Rank | Drug | Score | Known at freeze |
|------|------|-------|----------------|
| 1 | Lidocaine | 0.9713 | Unlabeled |
| 2 | Zonisamide | 0.9648 | Unlabeled |
| 3 | Ketoconazole | 0.9606 | Unlabeled |
| 4 | Haloperidol | 0.9431 | Unlabeled |
| 5 | Risperidone | 0.8918 | Unlabeled |
| 6 | Quetiapine | 0.8826 | Unlabeled |
| 7 | Bupivacaine | 0.8781 | Unlabeled |
| 8 | Indomethacin | 0.8649 | Unlabeled |
| 9 | Aripiprazole | 0.8497 | Unlabeled |
| 10 | Ropivacaine | 0.8235 | Unlabeled |

All 10 are unlabeled — no known clinical activity for ALS at freeze time. None are approved ALS drugs. None are currently in ALS trials (as far as our data shows).

This is important: the high-tier predictions for ALS are genuinely novel hypotheses, not confirmations of existing knowledge.

### What the model sees

The top 10 cluster into recognizable drug classes:

- **Local anesthetics** (lidocaine, bupivacaine, ropivacaine): These target sodium channels. Sodium channel dysfunction is implicated in ALS motor neuron hyperexcitability. The model is picking up a target-level signal, not a clinical one.
- **Antipsychotics** (haloperidol, risperidone, quetiapine, aripiprazole): Multiple receptor targets (D2, 5-HT2A, etc.) with broad neuromodulatory effects. These drugs have extensive safety and pharmacological profiles, giving the model rich feature vectors.
- **Anticonvulsant** (zonisamide): Neuroprotective properties, sodium/calcium channel modulation. Already studied in Parkinson's disease.
- **Anti-inflammatory** (indomethacin, ketoconazole): Neuroinflammation is a recognized ALS pathway.

The model does not "know" these mechanisms. It ranks drugs based on biological signature similarity to the ALS disease profile. The mechanistic coherence of the top 10 is emergent, not engineered.

### What the model does not see

- Whether these drugs cross the blood-brain barrier at therapeutic concentrations
- Whether the relevant targets are expressed in motor neurons (vs. peripheral tissue)
- Whether the doses required for neuroprotection are safe in ALS patients
- Commercial or regulatory feasibility

The model reduces biological search space. It cannot substitute for pharmacology.

## What happened (as of 2026-02-09)

Two outcomes were detected during initial ingestion:

### Outcome 1: Escitalopram (rank 26, moderate tier)

```
outcome_type:   phase_advanced
outcome_source: ctgov
outcome_detail: NCT00965497, Phase 3, COMPLETED, started 2009-07-01
```

A completed Phase 3 trial testing escitalopram for ALS-related symptoms (pseudobulbar affect). The trial started in 2009, well before our freeze date. This is a **retrospective signal** — the model ranked escitalopram at position 26, and historical data confirms clinical activity.

This is informative but not prospective. It tells us the model's ranking aligns with historical clinical interest, which is a necessary (but not sufficient) condition.

### Outcome 2: Trazodone (rank 13, moderate tier)

```
outcome_type:   phase_advanced
outcome_source: ctgov
outcome_detail: NCT04302870, Phase 2/3, RECRUITING, started 2020-02-27
```

A Phase 2/3 trial of trazodone for ALS, started in 2020. Again a retrospective signal (the trial predates the freeze), but notable: trazodone has shown neuroprotective effects in preclinical ALS models via unfolded protein response modulation, and the model ranked it 13th.

### What has not happened

No outcomes for any of the top 10 (high-tier) predictions. Zero. As of the initial ingestion, none of the high-tier drugs (lidocaine, zonisamide, ketoconazole, haloperidol, risperidone, quetiapine, bupivacaine, indomethacin, aripiprazole, ropivacaine) show any clinical trial activity for ALS.

This is not surprising at day zero. These are novel predictions. The whole point is that they have not been tested yet. But it means we have nothing to celebrate and nothing to claim. The high tier is an open bet.

## What we expect over 12 months

**Realistic expectation**: 1-3 of the 50 predictions will show a new trial registration or phase advance for ALS within 12 months. This is based on the base rate: across all 7 diseases in the slate, ~17% of predictions had some outcome at day zero (mostly retrospective). The prospective rate will be lower.

**What would be encouraging**: Any high-tier drug (rank 1-10) entering a Phase 1+ ALS trial within 12 months. Even one hit in the top 10 for ALS would be notable given the disease's low hit rate.

**What would be concerning**: Zero high-signal outcomes across all 50 ALS predictions after 12 months, AND similar nulls for other low-activity diseases in the slate (IPF, SLE). One disease failing is expected. All low-activity diseases failing would suggest the model only works for crowded therapeutic areas.

## What would falsify these predictions

Each prediction carries an explicit falsification criterion:

**High tier** (rank 1-10):
> "{drug} enters Phase 2+ trial for amyotrophic lateral sclerosis within 3 years, OR receives FDA approval"

If by February 2029 none of the top 10 drugs has entered a Phase 2 or higher ALS trial, these predictions are falsified.

**Moderate tier** (rank 11-50):
> "{drug} enters Phase 1+ trial for amyotrophic lateral sclerosis within 5 years"

Five years is generous because moderate-tier predictions are weaker claims. But the clock is running.

**System-level falsification for ALS**: If after 12 months the ALS predictions show enrichment vs. popularity-matched baseline below 1.0x, the model has no useful signal for ALS specifically. This does not invalidate the model for other diseases, but it would mean ALS is outside the system's competence.

## What this walkthrough demonstrates

1. **The predictions are recorded and immutable.** We cannot retroactively add drugs that turned out to work or remove drugs that didn't.

2. **The model makes surprising predictions.** Lidocaine for ALS is not obvious. It may be wrong. That's the point — conservative predictions that only recapitulate known treatments are useless.

3. **Day-zero results are honest.** Two moderate-tier hits from retrospective data, zero high-tier outcomes. No spin.

4. **Time will tell.** The value of this walkthrough increases with every quarterly data refresh. In 12 months, we will either have evidence that the model finds real signal in a hard disease, or evidence that it doesn't. Both results are publishable.

---

*Slate ID: `469bb270-a423-4929-96bd-6eaadeb7986b` | Disease: amyotrophic lateral sclerosis | Frozen: 2026-02-09 | Model: v5_rescue_features*
