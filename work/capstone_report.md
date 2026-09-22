# Capstone Report — <your lane>

- **Author:** Ali Mahmoud
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/AliMahmoud67/FlyRank_Starter
- **Date:** 2026-09-21

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract

Five sentences, written last, placed first: question → data → method → headline result →
what the output is for. This is the top of your deployed paper.

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

The notebook reads the FlyRank warehouse through DuckDB and uses pandas and scikit-learn for the feature preparation, model training, ranking, and evaluation.

The Random Forest uses:

- `n_estimators=200`
- `min_samples_leaf=10`
- `random_state=42`

The evaluation can be reproduced by opening the notebook from the GitHub repository in Google Colab, providing the read-only Hugging Face token through the `HF_TOKEN` Colab Secret, and running the notebook from top to bottom.

The final submission should record the exact package versions used in the final run so that the reported metrics can be reproduced from the repository.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

Data source: [FlyRank](https://flyrank.ai)
---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
