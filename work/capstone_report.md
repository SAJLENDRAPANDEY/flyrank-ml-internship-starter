# Capstone Report — Refresh Priority Scoring

- **Author:** Sajlendra Pandey
- **Lane:** Search Intelligence — Content Refresh Prioritization
- **Repo:** https://github.com/SAJLENDRAPANDEY/flyrank-ml-internship-starter
- **Date:** August 2026

## 1. Problem framing

### Decision

This project supports the decision of **which pages a FlyRank content or SEO team should review first for a possible content refresh**.

The unit of analysis is an **individual client webpage**. The output is a **ranked priority score and reason-coded refresh queue**. A human editor can use the ranking to decide which pages should be reviewed first rather than manually sorting thousands of pages.

The main research question is:

> Which webpages are most likely to experience declining search performance and should be prioritized for content refresh?

The target is a directional decline signal: a page is labelled as a decline when its June 2026 impressions fall by more than 20% compared with its May 2026 baseline.

The cost of a wrong recommendation is asymmetric. Refreshing a healthy page wastes editorial time, while missing a genuinely declining page can allow search visibility and potential business opportunities to deteriorate.

Machine learning is useful because several signals interact — including impressions, users, sessions, CTR, average position, update age, search volume, competition, content type, and search intent. A model can learn combinations of these signals and produce a repeatable ranking instead of relying only on a manually written rule.

---

## 2. Data safety

### Data used

The analysis used the FlyRank ML Internship Warehouse through DuckDB over Parquet:

- `fact_content_daily_performance` — daily Search Console and GA4 metrics per page, client, and day.
- `dim_content` — content metadata including update date, creation date, content type, search volume, competition, and search intent.

The analysis covered **May 1 – June 30, 2026**. May was used to build the 30-day feature window and June was used only to derive the decline label.

The raw May + June fact data contained **23,381,448 rows**. After aggregation and joining, the initial modeling dataset contained **382,069 page-level rows**. Rows with unrecoverable update dates were removed, leaving **334,704 rows** in the clean modeling dataset.

### Features used

The final model used these 10 features:

```text
impressions_30d
users_30d
sessions_30d
ctr
avg_position
days_since_last_update
search_volume
competition
content_type
main_intent
```

### Deliberately excluded data

The analysis deliberately excluded:

- Client names
- Domains
- Raw URLs
- Credentials
- Client-identifying information
- Label-derived fields such as `trend_direction` and `trend_pct`
- Pseudonymous identifiers such as `client_hash_id` and `content_hash_id` as model features

`client_hash_id` was used only for grouping train/test data so that pages from the same client did not appear in both training and testing.

No client-identifying details appear in the report or intended public outputs.

### Leakage risks considered

The June performance data was used only to create the target label and was not used as an input feature.

The label was defined as:

```text
decline_label = 1
if June impressions fell more than 20% versus the May baseline
else 0
```

A major data-quality issue was also identified: approximately 30% of the original update-age values were negative. Because a page cannot have an update date after the reference date, these values were treated as invalid. Rows with unrecoverable update dates were dropped instead of being guessed or imputed.

---

## 3. Baseline

The project first created a transparent rule-based priority score so that the trained models had a simple benchmark.

The baseline adds:

- **+2** for meaningful visibility: `impressions_30d >= 24`
- **+3** when CTR falls in the opportunity band: `0.05 <= ctr < 0.20`
- **+2** for staleness: `days_since_last_update >= 90`

The baseline is intentionally simple and explainable. It represents the kind of rule that a content team could implement without training a model.

The same Precision@K metric was used for the baseline and model comparisons.

### Clean held-out comparison

| K | Baseline | Random Forest | Improvement |
|---:|---:|---:|---:|
| 10 | **0.90** | 0.60 | -0.30 |
| 20 | **0.85** | 0.60 | -0.25 |
| 50 | **0.78** | 0.64 | -0.14 |
| 100 | 0.72 | **0.80** | +0.08 |

The overall decline rate in the clean held-out test set was approximately **0.523**, so Precision@K is interpreted against the task base rate rather than in isolation.

---

## 4. Model / analysis

### Target

The target is a binary directional decline label:

> `decline_label = 1` when June impressions decrease by more than 20% compared with the page's May baseline.

This is a decision-support label, not a causal explanation of why performance changed.

### Models tested

Two supervised models were tested:

1. Logistic Regression
2. Random Forest

Both used the same preprocessing approach:

- Numeric features: median imputation
- Categorical features: most-frequent imputation
- Categorical encoding: one-hot encoding
- Logistic Regression additionally used standard scaling for numeric features
- Random Forest used the numeric features without scaling

The Random Forest configuration used for the final cleaned-data experiment was:

```text
n_estimators = 300
max_depth = 12
min_samples_leaf = 10
class_weight = "balanced"
random_state = 42
n_jobs = -1
```

### Validation design

The project used client-grouped validation because pages belonging to the same client can share similar patterns.

Two validation passes were deliberately retained:

1. **5-fold `StratifiedGroupKFold`** on the dataset as first built, before the update-date issue was identified.
2. **Client-grouped 80/20 `GroupShuffleSplit`** on the cleaned dataset after the data-quality fix.

The grouped split produced **zero client overlap** between train and test.

### Initial cross-validation result

On the dataset as first built, before the date-quality correction:

| K | Baseline | Logistic Regression | Random Forest |
|---:|---:|---:|---:|
| 10 | 0.700 | 0.640 | **0.920** |
| 20 | 0.680 | 0.680 | **0.950** |
| 50 | 0.748 | 0.704 | **0.936** |
| 100 | 0.746 | 0.704 | **0.906** |

These results initially suggested a strong Random Forest advantage.

However, the update-date quality problem was then identified and the experiment was repeated on cleaned data. The cleaned-data result was substantially weaker, especially at the top of the ranking.

---

## 5. Evaluation

### Final clean-data split

The final clean-data modeling dataset contained **334,704 rows**.

The final held-out split used:

```text
Train: 252,543
Test : 82,161
Train clients: 50
Test clients : 13
Client overlap: 0
```

The held-out test decline rate was approximately:

```text
0.523
```

### Precision@K

The final Random Forest achieved:

| K | Baseline | Random Forest | Improvement |
|---:|---:|---:|---:|
| 10 | **0.90** | 0.60 | -0.30 |
| 20 | **0.85** | 0.60 | -0.25 |
| 50 | **0.78** | 0.64 | -0.14 |
| 100 | 0.72 | **0.80** | +0.08 |

The result is important operationally: at the very top of the queue, where a content team would normally begin, the simple baseline performed better. The Random Forest only became better at K=100.

### Error / ranking analysis

The model's top-ranked results sometimes assigned high probability to pages with extremely low visibility. This led to an explicit business rule in the production queue:

```text
pages with impressions_30d < 100 → MONITOR
```

This prevents a page with almost no meaningful search visibility from being recommended for immediate refresh merely because its model probability is high.

The ranked output therefore combines model probability with an eligibility filter and a human-readable action.

---

## 6. Interpretation

### Main finding

The strongest finding is not that the Random Forest beats the baseline. On the clean held-out data, it does not at K=10, K=20, or K=50.

The main finding is that **visible-but-underperforming pages are more important to the model than update age alone**.

### Feature importance

Random Forest feature importance on `dataset_clean` was approximately:

| Feature | Importance |
|---|---:|
| `impressions_30d` | 0.565 |
| `avg_position` | 0.247 |
| `days_since_last_update` | 0.052 |
| `ctr` | 0.037 |
| `sessions_30d` | 0.032 |
| `users_30d` | 0.032 |
| `main_intent` / other categorical features | lower |

`impressions_30d` and `avg_position` together account for more than 80% of the reported impurity-based feature importance.

In plain terms, the model behaves mostly like a **visible-but-underperforming page detector**, rather than a pure content-decay detector.

### Data-quality finding

The update-age feature initially contained a substantial number of negative values. The project did not hide this problem. Instead, the affected rows were investigated and the final model was re-evaluated on a cleaned dataset.

The initial cross-validation results were much stronger than the clean-data results. This reversal shows why data-quality validation is important before interpreting model performance.

### Negative result

The model did **not** demonstrate a reliable improvement over the transparent baseline at the top of the ranking.

That is a valid result. It suggests that a more complex model is not automatically better for this decision and that the baseline may be preferable when the editorial team needs a simple, explainable top-priority queue.

---

## 7. Recommendation

### Ranked action queue

The final production-style output contains **18,479 eligible held-out pages**.

| Action | Pages | Recommended human action |
|---|---:|---|
| `HIGH_PRIORITY_REFRESH` | 3,638 | Review and refresh this sprint |
| `REVIEW` | 7,968 | Review in the next editorial cycle |
| `MONITOR` | 6,873 | Continue monitoring and reassess next month |

The eligibility filter requires:

```text
impressions_30d >= 100
```

The queue also contains reason codes so an editor can understand why a page was flagged.

The most common trigger reported in the final queue was:

```text
VISIBLE_LOW_CTR_LOW_POSITION
```

with **7,978 pages**.

### How an editor should use the output

A FlyRank editor should:

1. Start with the `HIGH_PRIORITY_REFRESH` pages.
2. Check the reason code and the page's current impressions, CTR, and position.
3. Prioritize visible pages with weak CTR and/or weak ranking position.
4. Review stale pages as supporting evidence rather than treating age alone as proof of decline.
5. Use `REVIEW` pages for the next editorial cycle.
6. Keep `MONITOR` pages out of the immediate refresh workload unless other business context justifies review.

### Confidence and limits

The output is a **decision-support ranking**, not an automatic verdict.

The decline label can be influenced by seasonality, changes in demand, news cycles, promotion changes, or other factors. Therefore, a high priority score should trigger human review rather than automatic content modification.

Nothing in this analysis establishes a causal relationship with Google's ranking algorithm.

The model also has limited evidence for low-traffic pages because the production queue intentionally filters out pages with fewer than 100 impressions in the 30-day window.

---

## 8. Reproducibility

### Repository structure

The final work is organized as:

```text
work/
├── notebooks/
│   └── capstone_content_refresh.ipynb
├── outputs/
│   ├── final_refresh_recommendations.csv
│   └── top20_refresh_recommendations.csv
├── capstone_report.md
└── README.md
```

### Environment

The notebook uses Python 3 and the project environment includes:

```text
pandas
numpy
scikit-learn
duckdb
huggingface_hub
matplotlib
```

The repository `requirements.txt` should be used for the environment where available.

### Random seed

The analysis uses:

```text
random_state = 42
```

for the relevant splits and model training.

### Fresh-clone workflow

From a fresh clone of the repository:

```bash
git clone https://github.com/SAJLENDRAPANDEY/flyrank-ml-internship-starter.git
cd flyrank-ml-internship-starter
pip install -r requirements.txt
```

Then open:

```text
work/notebooks/capstone_content_refresh.ipynb
```

and run the notebook from top to bottom with a valid `HF_TOKEN` configured for access to the gated FlyRank dataset.

The notebook writes:

```text
work/outputs/final_refresh_recommendations.csv
work/outputs/top20_refresh_recommendations.csv
```

The output CSV files are committed separately to the repository.

---

## Claims checklist

- **Observed / measured:** dataset sizes, feature distributions, model rankings, Precision@K, feature importance, and final queue counts are reported as measured outputs.
- **Directional:** the decline label represents a directional month-over-month impression decline.
- **Decision-support:** the ranking is intended to support human editorial prioritization.
- **No causal claims:** the analysis does not claim that content age or any other feature causes search-performance decline.
- **No Google-algorithm claims:** the model does not predict Google's algorithm.
- **Data safety:** no client names, domains, raw URLs, credentials, or other client-identifying details are included in the report or intended public outputs.
- **Negative results retained:** the clean-data evaluation is reported even though it weakens the initial model result.
- **Base rate reported:** the final held-out decline rate is approximately 0.523 and should be considered when interpreting Precision@K.

---

## Acknowledgments & Data Credit

Built on the **FlyRank ML Internship dataset**.

No client names, domains, raw URLs, credentials, or raw exports appear in this report or its underlying notebook. All findings are reported as observed, directional, decision-support signals.
