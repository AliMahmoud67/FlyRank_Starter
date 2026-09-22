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

The project uses pseudonymized data and does not include client names, domains, URLs, private search queries, credentials

## 3. Baseline

The baseline is a transparent two-signal score designed to represent a simple editorial prioritization rule.

A content item receives:

- 1 point if it is at least 180 days old.
- 1 point if its February CTR is in the bottom 25% among pages with a similar February search-position group.

The resulting score ranges from 0 to 2. Higher scores indicate a stronger reason to review the page.

The final evaluation population contains 50,625 content items after removing rows with missing values in the model feature set. The future-opportunity base rate in this evaluation population is 27.04%.

The baseline was evaluated on the same 50,625 items and with the same ranking metrics used for the Random Forest model.

| Metric | Baseline |
|---|---:|
| Precision@20 | 40.00% |
| Precision@50 | 44.00% |

The baseline is intentionally simple and explainable. It provides a transparent reference point for evaluating whether the learned model adds useful ranking signal.

## 4. Model / analysis

Your method and why it fits the lane. The exact feature list (and what you left out on
purpose). The target or proxy definition, in one sentence.

## 5. Evaluation

Your split (grouped by client? time-aware?) and why. Metrics, model vs baseline **on the same
split**. What the errors look like — a short error analysis beats a big metric table.

## 6. Interpretation

What the model/clusters actually found. Feature importances or cluster profiles in plain
words. Surprises and negative results — a well-understood "no effect" is a valid result.

## 7. Recommendation

The ranked actions or decisions your output supports, and how a FlyRank editor would use them
tomorrow. State your confidence and the limits explicitly.

## 8. Reproducibility

The exact commands to re-run everything from a fresh clone, your random seeds, and your
environment (`pip freeze` highlights or `requirements.txt` deltas). If you claim a sealed or
holdout evaluation, two things must be committed: the cell/script that builds the sealed
frame, and the metrics file it produced — "evaluated once, blind" should be checkable from
your repo, not taken on faith.

## 9. Acknowledgments & data credit

One short section at the bottom of the deployed paper: "Built on the FlyRank ML Internship
dataset" **linking to https://flyrank.ai**. Crediting your data source is standard research
practice — and it's on the capstone's required-section list, so a paper without it isn't done.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
