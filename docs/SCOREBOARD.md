# Scoreboard

Public, longitudinal tracking of prospective prediction performance. Updated quarterly after each outcome ingestion cycle.

This is the document that accumulates evidence over time. Everything else explains the methods. This shows the results.

---

## Active Slates

| Slate | Frozen | Diseases | Predictions | Model |
|-------|--------|----------|-------------|-------|
| [Q1 2026 Cross-Disease](../releases/2026_Q1_cross_disease_slate.md) | 2026-02-09 | 7 | 350 | v5 |

---

## Quarterly Results

### Q1 2026 — Baseline (day zero)

*First measurement. All outcomes are retrospective (pre-existing in data at freeze time).*

| Metric | All | High tier | Moderate tier |
|--------|-----|-----------|---------------|
| Predictions | 350 | 70 | 280 |
| High-signal outcomes | 60 | — | — |
| Enrichment vs. random | — | 1.52x | — |
| Enrichment vs. popularity | — | **1.70x** | — |
| Score calibration (hits) | 0.917 | — | — |
| Score calibration (misses) | 0.843 | — | — |
| Prospective outcomes (positive time-to-event) | 0 | 0 | 0 |

**Per-disease breakdown:**

| Disease | Predictions | Any outcome | High-signal | Notes |
|---------|-------------|-------------|-------------|-------|
| Type 2 diabetes | 50 | — | — | Crowded; hardest to beat popularity |
| Glioblastoma | 50 | — | — | |
| Ulcerative colitis | 50 | — | — | |
| ALS | 50 | 2 | 2 | Both moderate tier; [walkthrough](examples/ALS_slate_walkthrough.md) |
| SLE | 50 | — | — | |
| IPF | 50 | — | — | Low-activity disease |
| TNBC | 50 | — | — | |

---

### Q2 2026 — First update

*Target: ~May 2026, after ChEMBL 35 release + CT.gov refresh.*

| Metric | All | High tier | Moderate tier |
|--------|-----|-----------|---------------|
| Predictions | 350 | 70 | 280 |
| High-signal outcomes | — | — | — |
| Enrichment vs. random | — | — | — |
| Enrichment vs. popularity | — | — | — |
| Score calibration (hits) | — | — | — |
| Score calibration (misses) | — | — | — |
| Prospective outcomes | — | — | — |
| New slate frozen? | — | | |

---

### Q3 2026 — Second update

*Target: ~August 2026. First window for genuinely prospective signals.*

| Metric | All | High tier | Moderate tier |
|--------|-----|-----------|---------------|
| Predictions | 350 | 70 | 280 |
| High-signal outcomes | — | — | — |
| Enrichment vs. random | — | — | — |
| Enrichment vs. popularity | — | — | — |
| Prospective outcomes | — | — | — |

---

### Q4 2026 — Third update

| Metric | All | High tier | Moderate tier |
|--------|-----|-----------|---------------|
| Predictions | 350 | 70 | 280 |
| High-signal outcomes | — | — | — |
| Enrichment vs. random | — | — | — |
| Enrichment vs. popularity | — | — | — |
| Prospective outcomes | — | — | — |

---

### Q1 2027 — 12-month assessment

*Full evaluation against [12-month success bar](SYSTEM.md#f-12-month-success-bar).*

| Success criterion | Met? | Value |
|-------------------|------|-------|
| Enrichment vs. popularity >= 1.3x | — | — |
| Positive enrichment in >= 4/7 diseases | — | — |
| Score calibration holds (hits > misses) | — | — |
| At least 1 genuinely prospective outcome | — | — |

| Failure criterion | Triggered? | Value |
|-------------------|-----------|-------|
| Enrichment vs. popularity < 1.0x | — | — |
| Zero outcomes in low-activity diseases (ALS, IPF, SLE) | — | — |
| Score calibration inverts | — | — |
| > 30% high-tier predictions falsified | — | — |

**Overall assessment**: —

---

## Cumulative Trend

This table tracks the same slate over time to show whether evidence accumulates or degrades.

| Quarter | High-signal outcomes | Prospective outcomes | Enrichment vs. popularity | Score calibration gap |
|---------|---------------------|---------------------|--------------------------|----------------------|
| Q1 2026 (baseline) | 60 | 0 | 1.70x | +0.074 |
| Q2 2026 | — | — | — | — |
| Q3 2026 | — | — | — | — |
| Q4 2026 | — | — | — | — |
| Q1 2027 | — | — | — | — |

The critical column is **Prospective outcomes**. Everything before that is retrospective. The system proves itself when this number starts climbing and enrichment holds.

---

## How to read this scoreboard

**If enrichment vs. popularity stays above 1.3x and prospective outcomes accumulate**: The model is working. It finds signal beyond popularity.

**If enrichment drops toward 1.0x as prospective signals replace retrospective ones**: The retrospective result was an artifact. The model is no better than picking popular drugs.

**If prospective outcomes appear but enrichment inverts (moderate > high tier)**: The confidence tier assignment is wrong. The model scores well globally but the ranking within diseases is miscalibrated.

**If zero prospective outcomes appear after 12 months**: Either the data refresh pipeline is broken, or the model has not predicted anything the world hasn't already tried. Both are informative failures.

---

## Rules

1. **Never edit past quarters.** Each row is frozen when published. Corrections go in a footnote, not an overwrite.
2. **Never remove a slate.** If a slate's predictions look bad, that's data. Deleting it is scientific fraud.
3. **Publish misses alongside hits.** Every quarterly update includes both.
4. **Use the same metric definitions.** See [METRICS.md](METRICS.md). Do not change formulas retroactively.
5. **Fill in the blanks or write "no update."** Empty cells mean "not yet measured." If measured and null, write the value.
