# Content Decline Ranking - Final Model Development, Validation, and Action Report

**Author:** Dania Yasir  
**Project:** FlyRank ML Internship Capstone  
**Date:** 30 August 2026  
**Primary evidence:** Current `main` branch of `Dania-Yasir/flyrak-project`  

## Executive summary

This project asks whether search-performance signals available in the first half of a month can help rank which content pages should be reviewed first for possible future decline. The operational goal is prioritization under limited analyst capacity, not automatic content editing and not hard classification.

The final March 2026 modeling cohort contains **61,795 pseudonymized content pages across 34 clients**. Features use March 1-15 only. The outcome proxy is defined from March 16-31: a page is labeled as a decline case when average daily impressions are more than **20% lower** than in the feature window. The observed decline rate in the final cohort is **32.4%**.

The selected model is a **Random Forest - Full Signal** model using 21 pre-outcome features. Under the final 5-fold client-disjoint validation, it achieved **74.6% mean Precision@100 (SD 13.6%)**. The same-fold momentum-only Random Forest achieved **66.2% (SD 12.8%)**, while the Week-4 rule baseline achieved **35.2% (SD 6.4%)**. The measured uplift is therefore **+39.4 percentage points** over the rule baseline and a smaller, directional **+8.4 points** over the stronger momentum-only comparison.

The most important methodological finding is that random row validation was optimistic. The same full model scored **90.0% Precision@100** under random row cross-validation but **74.6%** when complete clients were held out. The client-disjoint result is the primary estimate because it better matches the intended use case.

The final recommendation is to use the model as a **human-review ranking system**: model prioritizes, human investigates, human decides. The work does not establish why a page declined, does not predict Google's ranking algorithm, and does not show that refreshing a page will causally improve performance.

## 1. Problem framing

Large content portfolios can contain far more pages than an analyst can inspect regularly. A useful system therefore needs to answer a practical question: **which pages should be reviewed first when review capacity is limited?**

The unit of analysis is one pseudonymized content page within one pseudonymized client. The output is a risk score and ranked review queue. A human analyst then investigates the highest-priority pages and decides whether to refresh, inspect search intent, review title/meta CTR, diagnose an anomaly, or take no immediate action.

The cost of a wrong call is asymmetric. A false positive consumes analyst time and may lead to unnecessary investigation. A false negative can leave a genuinely declining page lower in the queue. This is why ranking quality near the top of the queue is more useful than a generic accuracy number.

## 2. Data and information boundary

The analysis uses the approved **FlyRank ML Internship warehouse release, build `v20260703`**, accessed through DuckDB and Hugging Face. The warehouse's `fact_content_daily_performance` table contains approximately **78.8 million** daily performance rows, but the final experiment uses only the March 2026 partition.

The March aggregation produced **331,437 page-level candidates**. Of these, **68,288** had complete Google Search Console coverage for the March 1-15 feature window, and **61,795** also had complete coverage for the March 16-31 outcome window. The final cohort spans **34 pseudonymized clients**.

**Feature window:** March 1-15, 2026.  
**Outcome window:** March 16-31, 2026.  
**Observed decline rate:** 32.4%.

Only information available by the end of March 15 is allowed as model input. `client_id` and `content_id` are used for grouping and validation only. Future-window impressions, future averages, the decline proxy, and any target-derived fields are excluded from the feature set.

This coverage filter is a real limitation: the conclusions apply to the eligible population with complete search-data coverage, not automatically to every page in the warehouse.

## 3. Target definition

A page is labeled as a future decline case when:

> average daily impressions in March 16-31 are more than 20% lower than average daily impressions in March 1-15.

The target is a **future-impression-decline proxy**. It is useful for ranking evaluation but is not a universal definition of content quality, content decay, or the need for a refresh. It also does not measure whether an intervention would improve the page.

## 4. Baseline development

The Week-4 baseline was intentionally simple and transparent. A page enters the baseline review queue when:

1. its average first-half search position is 20 or better; and
2. its CTR is low relative to pages in the same search-position bucket.

The position buckets are Top 3, 4-10, 11-20, 21-50, and 50+. Matching pages are ranked by first-half impressions so that more visible opportunities appear earlier.

Earlier signal analysis showed why the rule is narrow rather than universal. Search volume alone was not monotonic with later decline. CTR was more informative when interpreted relative to search position, especially among pages already visible in the Top 20. This made the rule a useful operational baseline, but not a strong standalone predictor of future decline.

The original Week-4 queue achieved **39.0% Precision@100** when ranked once over the full eligible March population. That value is retained only as historical context. The correct model comparison uses the same client-disjoint folds for every approach; under that fair evaluation, the Week-4 rule achieved **35.2% Precision@100 (SD 6.4%)**.

## 5. Model design

The selected model is a **Random Forest classifier** because the project evidence indicates that useful signal comes from nonlinear interactions among recent momentum, trend shape, visibility, CTR, position, and client-relative context.

The final model uses **21 pre-outcome features** in four groups:

- **Momentum anchors (2):** impression momentum percentage and log impression momentum.
- **Current performance level (5):** log first-half impressions, log first-half clicks, first-half CTR, first-half average position, and first-half active rate.
- **Trend shape (10):** two impression step changes, acceleration, count of declining steps, recent-vs-prior impression trend, click trend, CTR movement, position movement, activity movement, and position-shape availability.
- **Client-relative context (4):** within-client percentiles for traffic level, CTR, position, and recent-vs-prior impression trend.

The final Random Forest configuration uses 200 trees, maximum depth 10, minimum leaf size 25, `sqrt` feature sampling, balanced subsampling for class weights, and random seed 42.

A **momentum-only Random Forest** using the two momentum anchors is retained as a stronger simplicity test. It helps answer whether the additional 19 contextual and trend-shape features add ranking value beyond recent impression movement alone.

## 6. Validation design and leakage controls

The final evaluation uses **5-fold client-disjoint grouped validation**. Complete clients are held out, so no client's pages appear in both training and validation within a fold.

This design was chosen after earlier development experiments showed that random row splits were too optimistic. In the final deterministic rerun, random row cross-validation produced **90.0% mean Precision@100 (SD 1.9%)**, while client-disjoint validation produced **74.6% (SD 13.6%)** - a drop of **15.4 percentage points**. On average, about **29.6 clients** appeared on both training and validation sides of the random-row split, which allowed the model to benefit from client-specific patterns it had already seen.

The grouped result is therefore the primary estimate. The lower score is not a failure of the model; it is a more honest estimate of ranking performance for client groups that were absent from training.

The final leakage audit checked all 21 model features. Forbidden overlap was empty, no feature name indicated future/outcome/target information, and the final feature matrix had no missing values. Client-relative percentiles are computed using pre-outcome data before future-coverage filtering.

## 7. Final results

All primary methods below are evaluated on the same target, the same client-disjoint folds, and the same Precision@100 metric.

| Method | Mean Precision@100 | SD |
|---|---:|---:|
| Random Forest - Full Signal | **74.6%** | 13.6% |
| Random Forest - Momentum Only | **66.2%** | 12.8% |
| Week-4 Rule Baseline | **35.2%** | 6.4% |

The full model improves mean Precision@100 by **39.4 percentage points** over the Week-4 rule under the fair same-fold comparison. The more informative comparison is against the momentum-only Random Forest: the full model is **8.4 percentage points** higher. This smaller gap suggests that recent momentum carries a large share of the signal, while the additional context and trend-shape features add useful but not universal incremental value.

A separate client-level check in the final validation notebook also showed the same directional ordering using macro-client Precision@10: **69.2%** for Full Signal, **58.8%** for Momentum Only, and **38.8%** for the Week-4 rule. This metric is intentionally reported separately because it is not the same as pooled fold-level Precision@100.

## 8. Ranking concentration and review capacity

The final out-of-fold queue contains **61,795 pages**. The ranking concentrates observed decline cases near the top:

- reviewing the top **10%** captures **20.1%** of observed decline cases;
- reviewing the top **20%** captures **34.6%**;
- reviewing the top **30%** captures **48.5%**.

The risk score also separates outcomes across the queue. The observed decline rate is about **9.1%** in the lowest-risk decile and **65.1%** in the highest-risk decile, compared with an overall cohort rate of **32.4%**.

These patterns support the system's intended use: not to make an automatic content decision, but to put a denser concentration of likely decline cases earlier in an analyst's review queue.

## 9. Client generalization

Client-disjoint validation makes cross-client heterogeneity visible rather than hiding it. In the final action-playbook evaluation, **24 of 34 clients** showed positive concentration of decline among their high-risk pages, and the **median client uplift was 18.1 percentage points** relative to each client's own overall decline rate.

This is useful evidence, but it is not uniform success. Ten clients did not show positive concentration under that summary. Fold-level Precision@100 also has a substantial **13.6% standard deviation**. The model should therefore be monitored by client and over time rather than treated as having one fixed production accuracy.

## 10. Error analysis and interpretation

Historical out-of-fold error analysis from model development remains useful for understanding what the model can and cannot see.

**High-ranked false positives** often already showed large negative impression momentum, repeated declining 5-day steps, or worsening search position by the prediction boundary. These pages looked decline-like by March 15 but did not cross the later -20% threshold. Plausible explanations include temporary volatility, partial recovery, or a warning pattern that did not persist long enough to satisfy the label.

**Low-ranked true declines** often had flat or positive first-half momentum, no clear sequence of declining 5-day steps, and sometimes improving position. These are structurally difficult cases because the later reversal was not yet clearly visible in the allowed feature window.

Feature importance is treated only as a model-behavior sanity check. Earlier diagnostic work suggested that recent-vs-prior impression trend, the latest 5-day impression step, client-relative trend context, traffic level, position, CTR, and momentum all contributed. These importances are associations inside the model, not causal explanations of why search performance changed.

## 11. Ranked action playbook

The model output is converted into a human-review policy using within-fold risk percentile:

- **Risk percentile >= 0.80:** active human review.
- **0.50 <= risk percentile < 0.80:** watchlist.
- **Risk percentile < 0.50:** monitor.

For high-risk pages, observable pre-outcome signals assign a review type:

| Observed pattern | Review action |
|---|---|
| Sustained decline pattern | `CONTENT_REFRESH_REVIEW` |
| Ranking slippage | `SERP_AND_INTENT_REVIEW` |
| Visible page with relatively low CTR | `TITLE_META_CTR_REVIEW` |
| Client-relative anomaly | `MANUAL_DIAGNOSTIC_REVIEW` |
| High risk without a clear heuristic explanation | `HUMAN_REVIEW_BEFORE_EDIT` |
| Medium risk | `WATCHLIST_NO_IMMEDIATE_EDIT` |
| Lower risk | `MONITOR_NO_IMMEDIATE_EDIT` |

These are investigation priorities, not automatic editing instructions. A `CONTENT_REFRESH_REVIEW` recommendation means that an analyst should examine whether a refresh is appropriate; it does not mean that the page should be automatically rewritten.

**Operating principle: Model prioritizes -> human investigates -> human decides.**

## 12. Limitations

1. **Proxy outcome.** The target measures future impression decline, not content quality and not the causal value of a refresh.
2. **Coverage selection.** Only 61,795 of 331,437 March page-level candidates enter the final modeling cohort.
3. **Single primary month.** March 2026 is the main experiment. No completed temporal holdout is part of the final evidence.
4. **Limited client count.** The modeling cohort contains 34 clients.
5. **Unequal client sizes.** Client-disjoint folds can still differ substantially in page count and outcome prevalence.
6. **Client heterogeneity.** Performance is not equally strong for every held-out client group.
7. **Portfolio-context dependency.** Client-relative features assume access to enough same-client pages to compute current portfolio context.
8. **No causal interpretation.** Model scores, feature importance, and reason codes describe useful associations for ranking. They do not establish why a page declined.
9. **No Google-algorithm claim.** The model does not predict Google's ranking algorithm.
10. **No intervention-effect claim.** The results do not show that refreshing, rewriting, or changing a high-risk page will improve later performance.

An April next-month holdout was considered during development, but it is not included as completed evidence in the final repo. Temporal and prospective validation remain appropriate next steps before stronger production claims.

## 13. Reproducibility

The final workflow is reproducible from the public repository using the approved gated warehouse access.

- Repository: `https://github.com/Dania-Yasir/flyrak-project`
- Data source: `hf://datasets/FlyRank/internship-warehouse`, build `v20260703`
- Core environment: Python, DuckDB, pandas, NumPy, scikit-learn, matplotlib, huggingface_hub
- Random seed: `42`
- Validation: `5` client-disjoint folds
- Feature window: March 1-15, 2026
- Outcome window: March 16-31, 2026
- Decline threshold: `-20%`

Key notebooks:

1. `work/notebooks/w03_data_contract.ipynb` - data contract and prediction boundary.
2. `work/notebooks/w04_baseline_score.ipynb` - transparent rule baseline.
3. `work/notebooks/w05_model_training!.ipynb` - learned-model development.
4. `work/notebooks/w06_validation_audit.ipynb` - honest split and leakage audit.
5. `work/notebooks/w07_action_playbook.ipynb` - final queue, paper metrics, and action policy.
6. `work/notebooks/capstone.ipynb` - concise capstone narrative.

The final deterministic rerun sorts the monthly model frame by `client_id` and `content_id` after aggregation before validation. This change makes the reported Random Forest results repeatable without changing the data definition, target, features, or model settings.

## 14. Final conclusion

The project establishes a defensible ranking workflow for prioritizing pages that may be at higher risk of future search-impression decline. The final Full-Signal Random Forest produces materially stronger ranking performance than the original Week-4 rule under the same client-disjoint validation design and a smaller, directional gain over a stronger momentum-only Random Forest.

The project also establishes an important validation lesson: random row splitting substantially overstates performance when client-specific patterns can appear on both sides of the split. Holding out complete clients reduces the headline score but produces a more credible estimate.

The result should therefore be presented as **measured ranking value with meaningful client-level heterogeneity**, not as a universal accuracy number. The appropriate operational use is a human-in-the-loop review queue, with temporal monitoring and prospective validation before stronger production claims.

## Evidence and provenance note

This final report treats the current repository as the source of truth for all headline numbers and final methodology. Earlier project reports were used only for useful development context such as baseline reasoning, validation lessons, feature interpretation, and error-analysis examples. Where an older value differed from the current deterministic rerun, the current repository value supersedes it.
