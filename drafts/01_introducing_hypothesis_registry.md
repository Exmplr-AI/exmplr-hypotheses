# We Froze 350 AI Drug Predictions. Now We Wait.

*Why we're publishing our hypotheses before we know if they're right.*

---

Most AI drug discovery announcements follow a pattern: train a model, show impressive retrospective numbers, imply the future will be similar, raise money.

We're doing something different. We froze 350 drug repurposing predictions across 7 diseases, published them with timestamps and falsification criteria, and committed to tracking what happens — including when nothing happens.

This post explains why, what we actually did, and what it would take to prove us wrong.

---

## The problem with retrospective validation

Every AI drug repurposing paper reports some version of: "Our model recovered X% of known drug-disease relationships." The numbers are often impressive. They are also, by themselves, scientifically insufficient.

Retrospective evaluation tells you whether a model can recover what's already known. It cannot tell you whether it can find something new. Worse, it's susceptible to subtle data leakage, cherry-picked test sets, and post-hoc metric selection — none of which require dishonesty, just the ordinary incentives of wanting your system to look good.

The only way to know whether a prediction system works is to record its predictions before the world reveals the answers.

## What we built

Exmplr is a drug repurposing system that ranks existing drugs by their likelihood of showing future clinical activity for a given disease. It uses biological signatures from 10 data sources (gene targets, protein interactions, safety profiles, clinical outcomes, molecular properties, and others), combined through a learning-to-rank model trained on ~30,000 labeled drug-disease pairs.

The retrospective numbers are strong: leave-one-out Hit@50 of 0.2525 (hide one known treatment, check if it reappears in the top 50), temporal AUC of 0.9910 (train on pre-2015 approvals, predict 2016-2024). Full methodology is published in our [SYSTEM.md](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/SYSTEM.md).

But retrospective numbers are table stakes. What matters is what comes next.

## What we froze

On February 9, 2026, we froze the top 50 predictions for each of 7 diseases:

| Disease | Why |
|---------|-----|
| Type 2 diabetes | High trial activity. Crowded. Hard to beat popularity. |
| Glioblastoma | Moderate activity. High unmet need. |
| Ulcerative colitis | Moderate activity. Emerging biologics. |
| ALS | Moderate activity. Very few approved treatments. |
| Systemic lupus erythematosus | Moderate activity. Complex autoimmune. |
| Idiopathic pulmonary fibrosis | Low activity. Limited options. |
| Triple negative breast cancer | High activity. Active pipeline. |

We deliberately chose diseases spanning high, moderate, and low clinical trial activity. If the model only works for type 2 diabetes (where thousands of drugs are being tested), that's not useful. If it works for IPF (where almost nothing is happening), that's interesting.

Each prediction carries:
- A repurposing score (relative ranking, not a probability)
- A confidence tier (top 10 = high, top 50 = moderate)
- A record of what was known at freeze time
- A falsification criterion: a specific, time-bounded statement of what would disprove it

For example, one of our high-tier ALS predictions:

> **Zonisamide** (rank 2, score 0.965): "Enters Phase 2+ trial for ALS within 3 years, OR receives FDA approval"

If nothing happens by February 2029, this prediction is falsified. That's a concrete claim with a deadline.

The full dataset — 350 predictions with scores, ranks, tiers, and falsification criteria — is published as Parquet files in our [public repository](https://github.com/Exmplr-AI/exmplr-hypotheses/tree/main/slates/2026-Q1). Anyone with DuckDB can query it.

## What we found at day zero

Immediately after freezing, we compared predictions against existing data in ChEMBL, ClinicalTrials.gov, and FDA records. This is not prospective validation — it's checking whether the model's ranking aligns with historical clinical activity.

102 outcomes were detected: 60 high-signal events (phase advances, new trial registrations) and 42 low-signal events (trial completions and terminations).

The key metric: **1.70x enrichment vs. popularity-matched baseline** for high-tier predictions.

What this means: we compared our top-10 predictions against drugs with the same number of existing approved indications. A drug with 50 indications is more likely to appear in a new trial than a drug with 3, regardless of any model. After controlling for this, our high-tier predictions are still 1.70 times more likely to show real-world clinical activity.

What this does not mean: it does not mean we're 70% better at finding cures. It means, with retrospective data, the model selects drugs that are more clinically active than their popularity alone would predict.

## What we don't know yet

The honest answer is: almost everything that matters.

The 1.70x enrichment is real, but it's measured against historical data that the model may have indirectly seen during training. The true test is whether new events — trials registered, phases advanced, drugs approved after February 2026 — continue to favor our predictions over random or popularity-matched baselines.

For ALS specifically, the most interesting disease in our slate, the high-tier predictions are all novel. Lidocaine, zonisamide, ketoconazole, haloperidol — none have known ALS clinical activity. The model ranks them highly because of target-level and network-level biological similarity to ALS disease profiles. Whether that biological similarity translates to clinical activity is an open question. Right now, the answer is: zero outcomes for the top 10. We published a [full ALS walkthrough](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/examples/ALS_slate_walkthrough.md) explaining what the model sees and what it doesn't.

## Why we're publishing misses

Most AI systems report hits. We report everything.

Our [outcome taxonomy](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/OUTCOME_TAXONOMY.md) defines exactly what counts as a high-signal outcome and what doesn't. Our [scoreboard](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/SCOREBOARD.md) has blank cells for Q2, Q3, and Q4 2026. They will be filled in with whatever the data shows — including zeros.

We defined our [12-month success bar](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/SYSTEM.md#f-12-month-success-bar) before seeing any prospective results:

**The system works if** (after 12 months):
- Enrichment vs. popularity stays above 1.3x
- At least 4 of 7 diseases show positive enrichment
- Score calibration holds (higher scores → more outcomes)
- At least 1 genuinely prospective outcome appears

**The system does not work if**:
- Enrichment drops below 1.0x (worse than picking by popularity)
- Zero outcomes in low-activity diseases (ALS, IPF, SLE)
- Score calibration inverts

We wrote these criteria down, committed them to version control, and published them. We cannot move the goalposts.

## What happens next

Every quarter, we will:
1. Refresh outcome data from ChEMBL, ClinicalTrials.gov, and FDA
2. Update the scoreboard with new outcome counts and enrichment metrics
3. Publish a short update — including failure analyses for diseases with no activity
4. Freeze a new slate (same diseases, same model unless retrained)

The next update is targeted for May 2026, after the ChEMBL 35 release.

We will not:
- Edit past predictions
- Delete slates that look bad
- Redefine what counts as a success
- Claim victory before the evidence supports it

## Why this matters beyond Exmplr

The drug repurposing field has a credibility problem. Dozens of papers and companies claim AI-powered drug discovery, but almost none publish prospective, falsifiable predictions with defined success criteria and public failure tracking.

This isn't because the technology doesn't work. It's because the incentive structure rewards impressive retrospective numbers over honest prospective measurement. Freezing predictions is uncomfortable — it means you can be publicly wrong.

We think being publicly wrong, when it happens, is more valuable than being retrospectively right. It's how the field learns what works and what doesn't. And if the predictions hold up, the evidence will be stronger for having been accumulated honestly.

The data is public. The methodology is documented. The clock is running.

---

*Full repository: [github.com/Exmplr-AI/exmplr-hypotheses](https://github.com/Exmplr-AI/exmplr-hypotheses)*

*Methodology: [SYSTEM.md](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/SYSTEM.md) | Metrics: [METRICS.md](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/METRICS.md) | Outcome definitions: [OUTCOME_TAXONOMY.md](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/OUTCOME_TAXONOMY.md) | Replication: [REPLICATION.md](https://github.com/Exmplr-AI/exmplr-hypotheses/blob/main/docs/REPLICATION.md)*
