# Ranking Content Review Opportunities with Search and Content Signals

- **Author:** Ali Mahmoud
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/AliMahmoud67/FlyRank_Starter
- **Date:** 2026-09-21

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract

This project asks whether search and content signals can be used to rank pages that are worth reviewing or refreshing so editors can prioritize their limited time. Using the FlyRank internship warehouse, February 2026 data was used as the feature window and March 2026 as the future outcome window, with five search and content features and 50,625 complete held-out test items. A time-aware Random Forest ranking model was compared with a transparent two-signal baseline using Precision@20 and Precision@50. The Random Forest achieved 75% Precision@20 and 78% Precision@50, compared with 65% and 66% for the baseline, with a 27.04% future-opportunity base rate. The resulting ranked queue is intended to help editors decide which pages to investigate first, while keeping the final refresh decision with human reviewers.

## 1. Problem framing

This project asks whether observable search and content signals can be used to rank pages that are worth reviewing or refreshing, so editors can prioritize their limited content-review time.

The unit of analysis is one content item for one client.

The output is a priority score, ranked position, action label, and reason code.

A FlyRank editor can use the ranked queue to decide which pages should be reviewed or refreshed first.

A false positive sends editor time toward a page that may not need attention. A false negative may cause a page that could benefit from review to be missed.

Data and ML can help because content performance depends on several signals at the same time, including search visibility, click-through performance, and content age. A data-driven approach can combine these signals consistently and help prioritize review work instead of relying only on manual selection.

## 2. Data safety

This project uses the FlyRank internship warehouse release hosted on Hugging Face.

The main tables used are:

- `fact_content_daily_performance` — daily search performance for each client and content item.
- `dim_content` — content-level information such as word count and content creation date.

The feature window is February 2026, containing 7,355,108 rows from 2026-02-01 through 2026-02-28.

The outcome window is March 2026, containing 9,843,178 rows from 2026-03-01 through 2026-03-31.

February data is used for information available at the decision point. March data is used only as the future outcome for validation and evaluation.

The main features used are:

- `gsc_impressions`
- `gsc_clicks`
- `gsc_avg_position`
- `word_count`
- `content_age_days`

The following fields are deliberately excluded:

- `trend_direction` — derived from trend information and can reveal the outcome.
- `trend_pct` — used to derive trend information and can therefore leak the outcome.
- `is_declining_label` — directly derived from the trend mechanism and is not an independent feature.
- March performance fields — unavailable at the February decision point and therefore future information.
- Pseudonymous client and content IDs — used only for grouping and joining, never as model features.

GSC availability is checked so missing search data is treated as unavailable data rather than zero performance.

The project uses pseudonymized data and does not include client names, domains, URLs, private search queries, credentials.

## 3. Baseline

The baseline is a transparent two-signal score designed to represent a simple editorial prioritization rule.

A content item receives:

- 1 point if it is at least 180 days old.
- 1 point if its February CTR is in the bottom 25% among pages with a similar February search-position group.

The resulting score ranges from 0 to 2. Higher scores indicate a stronger reason to review the page.

The final evaluation population contains 50,625 content items after removing rows with missing values in the model feature set. The future-opportunity base rate in this evaluation population is 27.04%.

The baseline was evaluated on the same 50,625 items and using the same ranking metrics as the Random Forest model.

| Metric | Baseline |
|---|---:|
| Precision@20 | 65.00% |
| Precision@50 | 66.00% |

The baseline is intentionally simple and explainable. It provides a transparent reference point for evaluating whether the learned model adds useful ranking signal.

## 4. Model / analysis

A Random Forest classifier was used to estimate the probability that a content item would become a future review opportunity.

The model uses five February features:

- `gsc_impressions`
- `gsc_clicks`
- `gsc_avg_position`
- `word_count`
- `content_age_days`

The model outputs a probability score for each page, which is used to rank the pages.

The target is a future content-opportunity proxy: a page is labeled as an opportunity when its March CTR is in the bottom 25% among pages with a similar February search-position group, using pages with at least 100 March impressions.

The model deliberately excludes March performance fields, `trend_direction`, `trend_pct`, `is_declining_label`, product decision flags, and pseudonymous IDs from the feature set.

## 5. Evaluation

A time-aware evaluation was used to approximate the real deployment situation.

Historical month pairs were used for training:

- August 2025 → September 2025
- September 2025 → October 2025
- October 2025 → November 2025
- November 2025 → December 2025
- December 2025 → January 2026
- January 2026 → February 2026

The final February 2026 → March 2026 period was held out as the future test period.

After removing rows with missing model features, the test set contained 50,625 content items. The future-opportunity base rate was 27.04%.

| Metric | Baseline | Random Forest |
|---|---:|---:|
| Precision@20 | 65.00% | 75.00% |
| Precision@50 | 66.00% | 78.00% |

The Random Forest produced higher precision at both review capacities.

In the model's top 20 ranked pages, 15 were future opportunities and 5 were not. Several false positives had very low or zero February clicks and relatively low impression counts, showing that sparse search observations can make some pages difficult to rank correctly.

The results are directional decision support for this dataset and time period, not a guarantee of future performance.

![Model vs Baseline](work/outputs/charts/precision_comparison.png)

## 6. Interpretation

The Random Forest feature-importance results were:

| Feature | Importance |
|---|---:|
| `gsc_clicks` | 0.3347 |
| `gsc_impressions` | 0.2872 |
| `word_count` | 0.1481 |
| `content_age_days` | 0.1160 |
| `gsc_avg_position` | 0.1141 |

The model relied most on clicks and impressions from the February feature window. Word count, content age, and average position contributed smaller amounts individually.

The error analysis showed that some high-scoring pages did not become future opportunities. Several of these pages had zero February clicks and relatively low impression counts, suggesting that limited search observations can make the future outcome harder to identify.

The feature-importance values describe associations used by the fitted model and should not be interpreted as causal effects.

## 7. Recommendation

The main output is a ranked review queue.

A FlyRank editor can use the model score to prioritize pages for review, starting with the highest-ranked content and then checking the page manually before taking action.

The recommended workflow is:

1. Review the highest-scoring pages first.
2. Check whether the page is still useful and up to date.
3. Inspect its search visibility and click performance.
4. Decide whether to refresh, leave unchanged, or investigate another cause of weak performance.

The model provides prioritization support rather than an automatic refresh decision. A high score does not guarantee that a page needs a refresh, and the editor should consider context that is not represented in the five model features.

The final notebook also produces a ranked recommendation table with a model score, action, reason code, and confidence label.


## 8. Reproducibility

The analysis is implemented in `work/notebooks/capstone.ipynb`.

The notebook reads the FlyRank warehouse through DuckDB and uses pandas and scikit-learn for feature preparation, model training, ranking, and evaluation.

To reproduce the analysis from a fresh clone:

```bash
git clone https://github.com/AliMahmoud67/FlyRank_Starter.git
cd FlyRank_Starter
```
### Data and evaluation

The notebook builds the historical training frame from month-to-next-month pairs:

- August 2025 → September 2025
- September 2025 → October 2025
- October 2025 → November 2025
- November 2025 → December 2025
- December 2025 → January 2026
- January 2026 → February 2026

The final held-out test period is:

- February 2026 → March 2026

The holdout frame is built directly in `work/notebooks/capstone.ipynb`. The model is trained only on the historical month pairs, while the March 2026 outcome is used only after the February predictions are produced.

### Model configuration

The Random Forest uses:

- `n_estimators = 200`
- `min_samples_leaf = 10`
- `random_state = 42`

The fixed random seed makes the model training reproducible for the same data, environment, and code.

### Environment

The final notebook run used:

- Python `3.13.15`
- pandas `2.2.3`
- scikit-learn `1.6.1`
- DuckDB `1.3.2`

### Evaluation receipt

The final evaluation metrics are saved in:

`work/outputs/capstone_metrics.json`

The file records:

- test population size: `50,625`
- base rate: `27.04%`
- Precision@20 for the baseline: `65.00%`
- Precision@20 for the Random Forest: `75.00%`
- Precision@50 for the baseline: `66.00%`
- Precision@50 for the Random Forest: `78.00%`

The model-vs-baseline comparison chart is saved in:

`work/outputs/charts/precision_comparison.png`

These files provide the committed receipts for the reported evaluation numbers.

### Reproducibility notes

The reported numbers should be regenerated by running the notebook from top to bottom using the same repository code, FlyRank warehouse release, and environment versions listed above.

The evaluation design is time-aware, so the final test outcome is kept after the training periods rather than randomly mixed into the training data.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

Data source: [FlyRank](https://flyrank.ai)
---
