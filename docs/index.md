---
title: "Refresh Priority Scoring: Predicting Which Pages Need Content Refresh Next"
---

# Refresh Priority Scoring: Predicting Which Pages Need Content Refresh Next

**Sajlendra Pandey**
Machine Learning Engineer Intern — FlyRank AI
**Lane:** Refresh / Content Opportunity Scoring
**Program:** FlyRank ML Internship — Search Intelligence Capstone

---

## Abstract

Content and SEO teams cannot manually review every page every month. This project asks which pages show signals associated with search-performance decline and should therefore be prioritized for human content review. Using FlyRank's ML Internship warehouse, 30-day search and analytics signals were combined with content metadata and a directional month-over-month decline label. A Random Forest classifier was evaluated against a transparent rule-based baseline using a client-grouped held-out split and Precision@K. On the cleaned holdout split, the Random Forest did not outperform the baseline at K=10, K=20, or K=50, but achieved Precision@100 of 0.80 versus 0.72 for the baseline. The final output is a ranked, reason-coded decision-support queue for human editorial review.

---

## 1. Introduction / Problem Statement

FlyRank's content and SEO teams manage large numbers of pages and cannot manually review every page at the same frequency.

The decision supported by this project is:

> Which webpages should be prioritized for human content-refresh review?

The unit of analysis is a content page for a client.

The model produces a priority score and a ranked recommendation. A human editor then decides whether a page should actually be refreshed.

The system is **decision-support**, not an automatic content-update system.

---

## 2. Data

The project uses the FlyRank ML Internship Warehouse.

The main sources were:

- `fact_content_daily_performance`
- `dim_content`

The final modeling window used May 2026 as the feature period and June 2026 for the directional outcome label.

The cleaned modeling dataset contained:

**334,704 rows**

A total of **47,365 rows** were removed because their update-date information could not be reliably reconstructed — roughly 30% of the dataset at that stage. That correction is described in full in Section 3, because it changed the final result and is part of the story, not a footnote.

The public paper does not expose client names, domains, raw URLs, private queries, credentials, or raw warehouse exports.

---

## 3. Methodology

### Target

The decline label represents a directional month-over-month search-performance decline, based on the change in impressions between May and June (a drop of more than 20% counts as a decline).

A decline is not treated as proof of a ranking change or as a causal outcome.

### Features

The final model used:

- `impressions_30d`
- `users_30d`
- `sessions_30d`
- `ctr`
- `avg_position`
- `days_since_last_update`
- `search_volume`
- `competition`
- `content_type`
- `main_intent`

Client and content identifiers were not used as predictive features.

Future or label-derived fields were excluded to reduce leakage risk.

### Model

The primary final model was a:

**Random Forest Classifier**

with:

```text
n_estimators = 300
max_depth = 12
min_samples_leaf = 20
class_weight = balanced
random_state = 42
n_jobs = -1
```

Numeric features were median-imputed and scaled. Categorical features (`content_type`, `main_intent`) were imputed with the most frequent value and one-hot encoded, with unknown categories handled gracefully at prediction time.

A Logistic Regression model, using the same preprocessing, was also trained as a second point of comparison alongside the rule-based baseline.

### Data quality fix

A sanity check on `days_since_last_update` turned up negative values for close to a third of rows — a page cannot have been "updated" after the reference date used to build the features. This was traced to inconsistent update-date values in the source metadata rather than a real signal. Every row whose corrected age still came out negative was dropped instead of imputed, which is what produced the 334,704-row clean dataset described above.

### Validation

The evaluation used a **client-grouped train/test split**, so that a client seen during training is never scored during testing. One split produced:

```text
Train: 252,543 rows
Test : 82,161 rows

Train clients: 50
Test clients : 13
Client overlap: 0
```

A 5-fold `StratifiedGroupKFold` cross-validation, grouped the same way, was also run earlier in the project — before the update-age bug was caught — and is reported alongside the cleaned result below, because the two disagree in an informative way.

---

## 4. Results

### Cross-validated Precision@K — before the date-quality fix

| K | Baseline | Logistic Regression | Random Forest |
|---|---|---|---|
| 10 | 0.700 | 0.640 | **0.920** |
| 20 | 0.680 | 0.680 | **0.950** |
| 50 | 0.748 | 0.704 | **0.936** |
| 100 | 0.746 | 0.704 | **0.906** |

Taken at face value, this looks like a clear win for the model — roughly 25 points of precision over the baseline at every K.

### Cleaned holdout split — after the fix

| K | Baseline | Random Forest | Improvement |
|---|---|---|---|
| 10 | **0.90** | 0.60 | −0.30 |
| 20 | **0.85** | 0.60 | −0.25 |
| 50 | **0.78** | 0.64 | −0.14 |
| 100 | 0.72 | **0.80** | +0.08 |

At the top of the list — where a content team would actually start — the simple rule-based baseline wins. The Random Forest only pulls ahead once the list opens up to the top 100 pages.

A meaningful part of the model's apparent edge in the first table was riding on the update-date bug, not a genuine pattern. Both tables are reported here deliberately, because the gap between them is the most useful finding this project produced.

### Feature importance

On the cleaned data, the Random Forest leaned almost entirely on two signals:

| Feature | Importance |
|---|---|
| `impressions_30d` | 0.565 |
| `avg_position` | 0.247 |
| `days_since_last_update` | 0.052 |
| `ctr` | 0.037 |
| `sessions_30d` | 0.032 |
| `users_30d` | 0.032 |
| `search_volume` | 0.006 |
| `competition` | 0.005 |

Visibility and search position together account for over 80% of the model's decision weight. `days_since_last_update` — the signal this lane is named after — contributes only about 5%. In plain terms, this model behaves more like a "visible but underperforming page" detector than a true content-decay detector.

---

## 5. Limitations & Honest Framing

- The decline label is directional, not causal. A drop of more than 20% in impressions can come from seasonality, a one-off news cycle, or a client pausing promotion — not only from a change in search ranking.
- A real data-quality bug shaped the early cross-validation results. The cleaned, single-split comparison is the more trustworthy read.
- Cross-validation standard deviations were large (roughly ±0.35 on Precision@10 for the baseline), so five folds is not a large amount of evidence on its own.
- The model relies mostly on visibility and position rather than the staleness signal it was built around.
- The production queue only includes pages with `impressions_30d ≥ 100`; genuinely low-traffic pages that may still matter to a client are excluded by design.
- Every finding here is decision-support for a human reviewer, not a verdict on any individual page.

---

## 6. Ranked Recommendations — Action Playbook

The final model scores only the held-out test split — pages it never saw during training. Eligible pages (`impressions_30d ≥ 100`) are assigned an action and a reason code.

| Action | Pages | Meaning |
|---|---|---|
| `HIGH_PRIORITY_REFRESH` | 3,638 | Visible and underperforming on CTR/position, and/or stale — refresh title, meta, and content this sprint |
| `REVIEW` | 7,968 | Worth a look next editorial cycle, not urgent |
| `MONITOR` | 6,873 | Currently fine — re-check next month |

Total eligible queue: **18,479** of 82,161 held-out pages.

The single most common reason a page gets flagged is `VISIBLE_LOW_CTR_LOW_POSITION` (7,978 pages) — a page that's getting impressions but underperforming on both click-through rate and ranking position.

---

## 7. Reproducibility

- **Notebook:** `work/notebooks/capstone.ipynb` in the project repository — runs top to bottom without errors given a valid warehouse access token.
- **Random seed:** `random_state = 42` throughout, for every split and every model.
- **Outputs:** the recommendation CSVs are generated locally by running the notebook and are not committed to the repository; the top-20 shortlist is reproduced above as a table so it's visible without re-running anything.
- **Environment:** Python 3, `pandas`, `numpy`, `scikit-learn`, `duckdb`, `huggingface_hub`, `matplotlib`.

---

## 8. Acknowledgments & Data Credit

Built on the **FlyRank ML Internship dataset**. [flyrank.ai](https://flyrank.ai)

No client names, domains, raw URLs, credentials, or raw exports appear anywhere in this paper or its underlying notebook. Every finding above is reported as observed, directional, decision-support signal — not as proof of how Google's ranking algorithm works.
