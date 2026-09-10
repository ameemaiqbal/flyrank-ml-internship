# Ranking Volatility as an Early-Warning Signal for Content Decline

**FlyRank AI Internship — ML Track Capstone**
**Author:** Ameema Iqbal

---

## Abstract

This paper investigates whether short-term ranking position volatility can serve as an early-warning signal for content decline, ahead of visible impression loss. Using a 30-day sample from FlyRank's internship warehouse (June 2026, 99,181 content items across 46 clients), a leak-free Random Forest model using only `position_volatility` was validated against a client-grouped holdout split. The model concentrates true declining pages effectively at the top of a ranked list (Precision@20: 0.85, Precision@50: 0.68, both clearly above the 0.529 base rate) despite weak overall accuracy (0.516), making it suitable for review-prioritization, not page-level verdicts. Along the way, this work surfaced and corrected a confirmed data leak, an unfair baseline comparison, and a measurable disagreement between simple heuristics and model confidence affecting roughly 12% of content.

---

## Introduction / Problem Statement

**Research question:** Can short-term ranking (position) volatility serve as an early-warning signal for content decline, before impression loss becomes obvious?

**Decision it supports:** Which pages a content team should prioritize for manual review each week, given limited review capacity, before a page's traffic has already visibly dropped.

---

## Data

**Release:** `FlyRank/internship-warehouse` on Hugging Face.
**Tables:** `fact_content_daily_performance_sample.parquet` (latest full month, June 2026) and `dim_content.parquet` (content metadata).
**Date window:** 2026-06-01 to 2026-06-30, split into two 15-day halves for the impression-trend label.

**Excluded, with why:**
- `ga4_*` / channel-split `sessions_*`: only 3.6% row coverage
- `ai_*` referral columns: under 0.1% of rows have any AI referral
- `last_optimized_date` / `optimization_eligible_date`: confirmed leakage — 12.3% of dated rows fall after the prediction window
- `impression_drop_pct`: confirmed 100% agreement with the label, a direct restatement
- `client_hash_id` / `content_hash_id`: kept only as join/context keys, never as model features

All identifiers are pre-hashed by FlyRank before release. No raw client names, URLs, or queries appear anywhere in this analysis.

---

## Methodology

- **Assumptions:** ranking instability precedes visible traffic decline; a page's short-term position variance is measurable from a single 30-day window.
- **Features:** `position_volatility` (30-day standard deviation of daily average ranking position) — the sole feature in the final leak-free model.
- **Label:** `is_declining`, a rule-based proxy (`imp_last15 < 0.8 × imp_prev15`), not a human-verified outcome.
- **Baseline:** a hand-written rule combining normalized volatility and normalized impression drop.
- **Validation design:** `GroupShuffleSplit` by `client_hash_id` (75/25), chosen because random row-level splits let the same client appear in both train and test, inflating apparent performance.
- **Leakage checks:** three rounds of auditing found and removed `last_optimized_date`, `optimization_eligible_date`, and `impression_drop_pct`. The final feature was directly tested for label agreement (56.3%, near chance), confirming it is not a disguised restatement of the label.

---

## Results (vs Baseline)

| Metric | Baseline Rule | Random Forest (leak-free) | Base Rate |
|---|---|---|---|
| Precision@20 | 1.000* | 0.850 | — |
| Precision@50 | 1.000* | 0.680 | — |
| Accuracy | — | 0.516 | 0.529 |

*The baseline's perfect score is a leakage artifact (it still contains `impression_drop_pct`), not evidence of genuine superiority — this comparison is not apples-to-apples until the baseline is rebuilt leak-free.*

The honest, defensible claim: **the leak-free model beats the base rate at the top of the ranking** (Precision@20/50), even though overall accuracy does not exceed guessing the majority class. This matches the intended use case — prioritizing a limited review queue, not classifying every page.

---

## Limitations & Honest Framing

This work cannot claim:
- **Causation** — volatility correlates with decline in this sample; no causal mechanism is shown.
- **Generalization beyond this 30-day window** — untested across seasons or algorithm updates.
- **A validated baseline comparison** — the current baseline score is a leakage artifact.
- **Reliability for any single page** — the model is weak in the middle of the distribution; useful only for ranking/prioritization.
- **Full-client-base coverage** — only 46 of 104 total clients appear in this sample.
- **Perfect reproducibility** — repeated runs produced measurably different Precision@K values, likely from data-source row-order non-determinism.
- **AI-search or GA4-based insights** — both excluded early due to extreme data sparsity.

All claims above are observed, measured, directional, and decision-support — none are causal or universal.

---

## Ranked Recommendations

| Reason Code | Count | Action |
|---|---|---|
| HIGH_VOLATILITY_REVIEW | 26,741 | Investigate before it becomes a traffic loss |
| MODEL_TIER_DISAGREEMENT | 11,709 | Manual review — model and simple heuristic disagree |
| MODERATE_VOLATILITY_WATCH | 33,721 | Monitor, not urgent |
| STABLE_NO_ACTION | 27,010 | Deprioritize |

**Never automate:** publishing content changes, or deprioritizing/deleting `STABLE_NO_ACTION` pages without a separate quality check. A low volatility score means the *ranking* hasn't moved — it says nothing about content quality.

---

## Artifacts

![Reason code distribution](work/outputs/capstone_reason_code_chart.png)

![Volatility distribution by decline status](work/outputs/capstone_volatility_distribution.png)

Full recommendation queue and results table available in `work/outputs/capstone_recommendations.csv` and `work/outputs/capstone_results_table.csv`.

---

## Reproducibility

- **Repository:** [github.com/ameemaiqbal/flyrank-ml-internship](https://github.com/ameemaiqbal/flyrank-ml-internship)
- **Capstone notebook:** [work/notebooks/capstone.ipynb](https://github.com/ameemaiqbal/flyrank-ml-internship/blob/main/work/notebooks/capstone.ipynb)
- **All track notebooks:** [work/notebooks/](https://github.com/ameemaiqbal/flyrank-ml-internship/tree/main/work/notebooks)
- **Exported data/artifacts:** [work/outputs/](https://github.com/ameemaiqbal/flyrank-ml-internship/tree/main/work/outputs)

---

## Demo, Social Cut & Employer Summary

**5-minute demo outline:**
1. (30s) The question: can ranking volatility warn of decline before traffic drops?
2. (60s) Show the leak I caught — 100% "accuracy" that turned out to be a disguised label.
3. (90s) The honest result after fixing it: Precision@20 of 0.85, well above the 0.53 base rate.
4. (60s) The recommendation queue, including the 12% "disagreement" category I built to avoid overclaiming.
5. (60s) Limitations and what I'd do next with more data.

**Social post cut:**
> Spent my FlyRank ML internship building a content-decline early-warning model — and the best part wasn't the model, it was catching a data leak that gave me a fake 100% accuracy score. Rebuilt it honestly, validated across clients, and shipped a recommendation queue that flags its own uncertainty instead of hiding it.

**3-sentence employer-facing summary:**
Built and validated an early-warning classifier for content ranking decline using FlyRank's production-scale search data, including identifying and correcting a target-leakage bug that had produced a misleadingly perfect result. Designed a client-grouped validation methodology to ensure findings generalize beyond the training sample, and translated model output into an actionable, uncertainty-aware review queue for a non-technical audience. Comfortable working across the full ML lifecycle: data engineering (DuckDB/SQL at scale), leakage auditing, model validation, and stakeholder-ready communication.

---

## Acknowledgments & Data Credit

This work was completed as part of the FlyRank AI Internship, using anonymized data provided by [FlyRank](https://flyrank.ai). Thank you to the FlyRank ML track team for the dataset, tooling, and mentorship that made this project possible.
