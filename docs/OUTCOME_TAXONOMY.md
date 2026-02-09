# Outcome Taxonomy

This document defines what counts as an outcome when evaluating frozen predictions against real-world data. These definitions are fixed. Changing them retroactively would invalidate all prior slate evaluations.

## High-Signal Outcomes

These three outcome types are used in enrichment and performance calculations. They represent unambiguous evidence that a drug-disease hypothesis received real-world clinical attention or validation.

### `first_trial_seen`

**Definition**: The first ClinicalTrials.gov registration where the drug appears as an intervention and the disease appears as a condition, with a `start_date` after the slate freeze date.

**Why it matters**: A new trial registration is the strongest prospective signal available. Someone with resources decided this drug-disease combination was worth testing.

**Deduplication**: Only one `first_trial_seen` outcome is recorded per (slate, disease, drug) triple, regardless of how many matching trials exist. This prevents diseases with many overlapping trials from inflating hit counts.

**Matching**: Exact case-insensitive match on drug intervention name and disease condition name. This is conservative and will miss trials using brand names or variant disease spellings. Under-counting is preferred to over-counting.

### `phase_advanced`

**Definition**: The maximum clinical phase recorded for a drug-disease pair has increased since the slate was frozen.

**Sources**: ChEMBL `max_phase_for_ind` field, or ClinicalTrials.gov trial phase exceeding the `known_at_freeze.max_phase` value.

**Why it matters**: Phase advancement means a drug-disease hypothesis passed a clinical milestone. Moving from Phase 1 to Phase 2 is a real decision backed by data.

**Edge case**: A drug may show phase advancement in ChEMBL without a corresponding new trial in CT.gov (or vice versa), because the databases have different coverage and update schedules. Both sources are tracked independently.

### `fda_approved`

**Definition**: An FDA submission for the drug with a `submission_date` after the slate freeze date, OR a ChEMBL `max_phase_for_ind` reaching 4 (post-market).

**Why it matters**: FDA approval is the gold standard outcome. It is also the rarest and slowest to appear.

**Distinction from label expansion**: If the drug was already approved for a different indication, a new approval for the predicted disease still counts. However, if the drug was already approved for the predicted disease at freeze time (`known_at_freeze.label == "approved"`), this outcome is expected, not surprising. The performance metrics handle this by separating predictions where the drug-disease pair was already known from those where it was novel.

## Low-Signal Outcomes

### `status_transition`

**Definition**: A ClinicalTrials.gov trial matching the drug-disease pair transitions to a terminal status: COMPLETED or TERMINATED/WITHDRAWN.

**Why it is tracked but not used in enrichment**: Status transitions are informative context but unreliable as success/failure signals:

- **COMPLETED** does not mean the trial succeeded. Many completed trials show no efficacy.
- **TERMINATED** does not always mean the drug failed. Trials terminate for funding, enrollment, sponsor decisions, or strategic reasons unrelated to efficacy.
- **WITHDRAWN** can mean the trial never started (regulatory, logistics) rather than a scientific failure.

Status transitions are included in outcome tracking for completeness and will become more useful as the system matures and develops more sophisticated trial-level outcome parsing (e.g., integrating results data).

## Excluded Signals

The following CT.gov statuses are intentionally excluded from outcome tracking:

### SUSPENDED

Suspension is a temporary administrative state. Trials frequently move from SUSPENDED back to ACTIVE or to COMPLETED. Tracking it would generate noise: a trial could appear as "suspended" in one ingestion run and "completed" in the next, creating contradictory outcomes for the same pair.

### NOT_YET_RECRUITING / RECRUITING / ACTIVE_NOT_RECRUITING / ENROLLING_BY_INVITATION

These intermediate statuses change frequently and do not represent discrete events. A trial moving from RECRUITING to ACTIVE_NOT_RECRUITING is not a meaningful outcome. Only the final state matters.

### Protocol amendments and updates

CT.gov records are amended constantly (contact changes, site additions, date adjustments). These are administrative, not scientific, events.

## Why FDA Label Expansion Is Not a Separate Type

An FDA approval for a new indication is recorded as `fda_approved` regardless of whether the drug had prior approvals for other indications. We do not distinguish "first approval" from "label expansion" because:

1. From the prediction system's perspective, the question is the same: "Will this drug show clinical activity for this disease?"
2. The regulatory distinction (new NDA vs. sNDA) is a commercial/regulatory artifact, not a scientific one.
3. The `known_at_freeze` field already captures whether the drug had prior approvals, allowing downstream analysis to stratify if needed.

## Summary Table

| Outcome Type | Signal Level | Used in Enrichment | Sources |
|-------------|-------------|-------------------|---------|
| `first_trial_seen` | High | Yes | CT.gov |
| `phase_advanced` | High | Yes | ChEMBL, CT.gov |
| `fda_approved` | High | Yes | FDA, ChEMBL |
| `status_transition` | Low | No | CT.gov |

## Preventing Metric Gaming

These definitions exist to prevent retroactive redefinition of success. If a future model version produces different predictions, it should be frozen into a new slate and evaluated against the same outcome definitions.

Do not:
- Add new outcome types that retroactively inflate hit rates for existing slates
- Change the high-signal / low-signal classification after observing results
- Count the same real-world event under multiple outcome types for the same pair
- Relax matching criteria (e.g., fuzzy name matching) to increase hit counts

The deduplication and matching rules are conservative by design. It is better to undercount real outcomes than to overcount ambiguous ones.
