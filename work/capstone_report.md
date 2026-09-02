# Structured Content Performance Archetypes

**Author:** ahmed8809  
**Lane:** Performance archetypes / unsupervised analysis  
**Repository:** https://github.com/ahmed8809/FlyRank_intern  
**Analysis snapshot:** March 2026

## Abstract

This case study addresses a FlyRank content-operations problem: how analysts can triage a large content inventory for human review without assuming that every stale page is a good refresh candidate. Using the March 2026 FlyRank internship warehouse snapshot, we aggregate content-level observations and apply K-Means clustering to impressions, clicks, click-through rate, average search position, and content staleness. A client-grouped holdout separates 26 training clients from 7 unseen validation clients, and the selected two-cluster solution reaches a validation silhouette of 0.6875. On the same held-out rows, K-Means surfaces 81 of 12,688 items (0.64%) as a stale/low-demand archetype, whereas the stricter stale-and-visible baseline surfaces none for refresh. The result is a descriptive decision-support framework for prioritizing human review in the FlyRank content problem—not evidence of causal refresh impact, semantic content type, or business value.

## Introduction / Problem statement

This case study starts from a practical FlyRank content-operations problem: analysts need to turn a large search-performance inventory into a manageable review queue and decide where human attention should begin. A simple rule can identify pages that are both old and visible, but that rule can miss low-volume pages whose strongest signal is staleness rather than exposure—and staleness by itself is not proof that a refresh will help.

This analysis asks:

> **What performance archetypes exist across the content inventory?**

The unit of analysis is a client-content pair aggregated over March 2026. The output is an interpretable cluster assignment plus a human-review prioritization framework for the FlyRank content problem. A wrong call can waste editorial effort or cause a useful page to be changed unnecessarily, so the output is intentionally decision support rather than an automated action or a prediction of refresh success.

## Data

The broader FlyRank internship warehouse contains approximately **79 million daily production search-performance records**. This paper uses a restricted **March 2026 snapshot** rather than the entire warehouse.

**Tables used**
- `fact_content_daily_performance`
- `dim_content`

The fact table supplies daily search-performance observations. The content dimension supplies `content_updated_date`, from which `stale_days` is derived relative to the latest March observation for each client-content pair.

The analysis produced **27,886 content observations across 33 clients**. The client-grouped holdout contains **15,198 training observations from 26 clients** and **12,688 validation observations from 7 clients**, with zero client overlap.

**Excluded from the public artifact**
- client names or identifying business information;
- raw URLs;
- article titles;
- private search queries;
- proprietary text.

Hashed identifiers are used internally for grouping/validation only and are not model features.

## Methodology

### Features

The clustering matrix contains:
1. March GSC impressions
2. March GSC clicks
3. GSC click-through rate
4. average search position
5. content staleness in days

Impressions and clicks are transformed with `log1p` to reduce the effect of strong right skew. All clustering features are standardized using parameters fitted on training clients only. `client_hash_id` and `content_hash_id` are never used as features.

### Model

The primary method is **K-Means clustering**. It fits the research question because there is no ground-truth target: the objective is to discover recurring groups of observed performance patterns.

Candidate values from `k=2` through `k=6` were evaluated with training-set silhouette. `k=2` produced the strongest training silhouette in the stored run (**0.5114**), so the final model uses:
- `k = 2`
- `random_state = 42`
- `n_init = 20`

Cluster names are assigned from training profiles rather than arbitrary numeric cluster IDs.

### Validation and leakage checks

A `GroupShuffleSplit` with `test_size=0.20` and `random_state=42` holds out entire clients. The validation model is applied to unseen clients without refitting the scaler or K-Means model.

The stored run reports:
- training silhouette: **0.5114**
- validation silhouette: **0.6875**
- client overlap: **0**

The analysis uses only the March 2026 snapshot and does not use future performance outcomes. Because this is unsupervised clustering, there is no prediction label and therefore no conventional target-leakage pathway; nevertheless, identifiers and future outcomes are excluded deliberately.

## Results

### Two recurring archetypes

The held-out validation set contains:
- **Visible / Established:** 12,607 items (**99.36%**)
- **Stale Low-Demand:** 81 items (**0.64%**)

The validation profiles show a large separation in staleness. Median stale days are approximately **34 days** for Visible / Established and **230 days** for Stale Low-Demand. Median March impressions are **319** versus **4**, respectively.

The validation silhouette of **0.6875** indicates reasonably separated structure under the chosen representation, but it should not be read as predictive accuracy.

### Model vs baseline on the same split

The Week-4 baseline is a transparent operational rule: a page is selected for refresh only when it is **at least 90 days stale and has at least 1,000 March impressions**.

Both the model and baseline are evaluated on the same 12,688 held-out rows. Because the clustering task has no ground-truth outcome, accuracy, precision, AUC, or lift would not be a valid comparison. Instead, the comparison is operational coverage.

In the stored run:
- K-Means surfaced **81 / 12,688 (0.64%)** as Stale Low-Demand.
- The strict baseline surfaced **0 / 12,688 (0%)** for refresh.
- The difference shows that the learned archetype lens and the strict rule identify different review populations.

This is **not** evidence that K-Means is more accurate or that any refresh will improve performance. It is evidence that the two decision-support frameworks have different coverage on the same held-out clients.

## Limitations & honest framing

1. **Descriptive, not causal.** The clusters describe observed performance patterns; they do not show that refreshing content causes better performance.
2. **Not semantic clustering.** No article text is used, so clusters do not represent topic, meaning, or content quality.
3. **Rare archetype.** The Stale Low-Demand group contains only 81 validation observations, so its stability deserves monitoring.
4. **Limited historical window.** The main model uses a March 2026 snapshot rather than a long longitudinal history.
5. **Exposure is not business value.** Search impressions do not directly measure revenue, conversions, strategic importance, or editorial value.
6. **Missing context.** The model cannot judge factual accuracy, search-intent alignment, content quality, or whether an editorial update is appropriate.
7. **Cluster uncertainty.** Observations near cluster boundaries should be reviewed manually.
8. **Validation scope.** Client-grouped validation is stronger than a random page split, but it does not guarantee stability on future clients or future months.
9. **Baseline strictness.** The stale-and-visible baseline selected no held-out pages in the stored run, so it provides a useful contrast but a weak coverage benchmark for this particular validation composition.

## Ranked recommendations

### 1. Review stale pages with meaningful visibility

Start with pages at least 90 days stale that still receive measurable search exposure. These are plausible refresh candidates because they combine age with evidence of search visibility.

### 2. Diagnose very stale, low-demand pages before investing heavily

Use the Stale Low-Demand archetype as a diagnostic queue, not an automatic refresh list. Check topic demand, strategic value, and editorial context before investing.

### 3. Maintain established pages unless another signal indicates a problem

The Visible / Established archetype generally has stronger observed exposure and lower staleness. Monitor these pages rather than automatically rewriting them.

### 4. Manually inspect uncertain assignments

Pages near the learned cluster boundary should be reviewed by a person rather than treated as definitive archetype members.

### 5. Revalidate over time

Rerun the clustering and action rules on later snapshots. Track distribution drift, cluster stability, missingness, and human-review overrides.

**All recommendations are decision support, not automated production actions.**

## Reproducibility

The analysis is implemented in the public repository:

- Repository: https://github.com/ahmed8809/FlyRank_intern
- Capstone notebook: https://github.com/ahmed8809/FlyRank_intern/blob/main/work/notebooks/capstone.ipynb
- Baseline notebook: https://github.com/ahmed8809/FlyRank_intern/blob/main/work/notebooks/w04_baseline_score.ipynb
- Model notebook: https://github.com/ahmed8809/FlyRank_intern/blob/main/work/notebooks/w05_model.ipynb
- Validation audit: https://github.com/ahmed8809/FlyRank_intern/blob/main/work/notebooks/w06_validation_audit.ipynb
- Action playbook: https://github.com/ahmed8809/FlyRank_intern/blob/main/work/notebooks/w07_action_playbook.ipynb

Reproduction settings: March 2026 snapshot; K-Means with `k=2`, `random_state=42`, `n_init=20`; client-grouped validation; `log1p` on impressions/clicks; training-only standardization.

## Acknowledgments & data credit

Built on the **FlyRank ML Internship dataset**. Data is used according to the internship's data-use rules and is presented here in a public-safe, anonymized form.

**Data source:** https://flyrank.ai

---
