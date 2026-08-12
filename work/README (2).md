# work/ — your space

Everything you build lives here: lane experiments, notebooks, figures, outputs, and your capstone report. The rest of the repo is the shared reference; this folder is yours.

## Rules of the road

1. **Copy, don't edit.** Need to change the pipeline? Copy the script here (e.g. `work/scripts/03_train_model_v2.py`) or adjust the feature lists in `scripts/ml_utils.py`. The reference pipeline in `scripts/` stays pristine — it's the baseline you compare against, and reviewers expect to find it unchanged.

2. **No datasets in git.** Raw datasets, including the recommendation CSVs this project generates, are not committed to the repository. Small summary tables belong in the report as markdown, not as raw data files.

3. **Stay reproducible.** Fix your random seeds and note them in your report. Someone with a fresh clone should be able to re-run your work from your instructions alone.

4. **Public-safety language.** Everything here may end up public with your submission: observed / measured / directional / decision-support — no client-identifying details, no causal claims without a design (see `DATA_USE.md`).

---

# Project Overview

## Lane

**Search Intelligence — Content Refresh Prioritization**

## Research Question

**Which webpages are most likely to experience declining search performance and should be prioritized for content refresh?**

## Decision Supported

This project supports an editorial decision:

> Which webpages should the content/SEO team review first for a possible content refresh?

The system produces a ranked list of webpages with a priority score and recommended action.

## Unit of Analysis

The primary unit of analysis is an individual webpage/content record.

## Expected Output

The final system produces:

- a decline-risk / priority score
- a rank for each webpage
- an editorial action
- supporting search-performance signals

The main actions are:

- `HIGH_PRIORITY_REFRESH`
- `REVIEW`
- `MONITOR`

---

# Problem Framing

Search performance can change because of multiple interacting signals such as visibility, average search position, CTR, traffic/users, and content freshness.

A simple rule-based approach can identify obvious cases, but a machine-learning model can combine several signals and produce a ranked prioritization score.

The purpose of this project is **not** to predict Google's ranking algorithm.

The purpose is to provide **decision support for content refresh prioritization**.

A human editor should review the recommendations before taking action.

---

# Data

The analysis uses FlyRank search/content performance data accessed through the provided data environment.

The modelling workflow uses features such as:

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

The following types of fields were not used as predictive model features:

- client identifiers
- content identifiers
- label-derived fields
- fields that could directly reveal the target
- other identifier-like fields

Client and content IDs are used only where necessary for grouping or identifying recommendation outputs.

No client-identifying information is intentionally included in the final report.

---

# Data Quality and Leakage Checks

Several data-quality checks were performed before the final modelling experiment.

The update-age calculation was checked and corrected because the initial calculation produced negative values for records whose update date was later than the reference date (roughly 30% of rows were affected before the fix).

The corrected update-age field was then used as:

```text
days_since_last_update
```

The final cleaned modelling dataset was created after removing invalid records according to the defined data-quality checks.

Label leakage was also considered.

Fields derived directly from the target or future outcome were excluded from the predictive feature set.

Client identifiers were not used as model features.

---

# Baseline

A transparent rule-based baseline was created before evaluating the machine-learning model.

The baseline assigns points based on observable search-performance signals including:

* recent impressions
* CTR range
* update age

The baseline provides a simple comparison point for determining whether the machine-learning model adds useful ranking signal.

The baseline was evaluated using the same Precision@K framework used for the model.

---

# Machine Learning Method

The final model used in the clean client-grouped experiment was a:

**Random Forest Classifier**

Configuration (as run in `notebooks/capstone.ipynb`):

```text
n_estimators = 300
max_depth = 12
min_samples_leaf = 20
class_weight = balanced
random_state = 42
n_jobs = -1
```

Categorical variables were processed using:

```text
SimpleImputer(strategy="most_frequent")
OneHotEncoder(handle_unknown="ignore")
```

Numeric variables were processed using median imputation and scaling.

The model outputs a probability of the positive decline class.

That probability is then used as the:

```text
priority_score
```

for ranking pages.

---

# Final Feature Set

The main modelling features were:

### Numeric

```text
impressions_30d
users_30d
sessions_30d
ctr
avg_position
days_since_last_update
search_volume
competition
```

### Categorical

```text
content_type
main_intent
```

Identifiers were excluded from the predictive feature set.

---

# Validation Strategy

The final evaluation used a **client-grouped train/test split**.

The purpose was to prevent the same client from appearing in both training and test data.

One final split produced:

```text
Train: 252543 rows
Test : 82161 rows

Train clients: 50
Test clients : 13
Client overlap: 0
```

The zero client overlap provides a stronger test of whether the model generalizes beyond clients observed during training.

---

# Evaluation Metric

The primary ranking metric is:

**Precision@K**

The model was evaluated at:

```text
K = 10
K = 20
K = 50
K = 100
```

Precision@K measures the proportion of actual decline cases among the top K ranked recommendations.

The overall decline rate is also considered when interpreting the results.

---

# Final Model Results

The final Random Forest experiment, on the cleaned client-grouped holdout split, produced:

|   K | Baseline | Random Forest | Improvement |
| --: | -------: | ------------: | ----------: |
|  10 |     0.90 |          0.60 |       -0.30 |
|  20 |     0.85 |          0.60 |       -0.25 |
|  50 |     0.78 |          0.64 |       -0.14 |
| 100 |     0.72 |          0.80 |       +0.08 |

For context, an earlier 5-fold grouped cross-validation pass — run before the update-age bug was caught — showed the Random Forest well ahead of the baseline at every K (e.g. Precision@20: 0.95 vs 0.68). That comparison is reported in `capstone_report.md` alongside this cleaned one, because the gap between the two is itself a finding, not something to quietly drop.

## Interpretation

The Random Forest did **not** outperform the transparent baseline at K=10, K=20, or K=50, on the cleaned holdout split.

At K=100, the Random Forest achieved:

```text
Precision@100 = 0.80
```

compared with:

```text
Baseline Precision@100 = 0.72
```

This is an improvement of:

```text
+0.08
```

Therefore, the final result should be interpreted as a **mixed model result**:

* the baseline is stronger for very small editorial queues
* the Random Forest becomes more useful when the review pool is expanded to approximately 100 pages

No causal claim is made from these results.

---

# Feature Importance

The Random Forest feature importance analysis (on the cleaned holdout model) identified the following strongest signals:

| Feature                  | Importance |
| ------------------------ | ---------: |
| `impressions_30d`        |   0.565367 |
| `avg_position`           |   0.247332 |
| `days_since_last_update` |   0.052336 |
| `ctr`                    |   0.036878 |
| `sessions_30d`           |   0.032474 |
| `users_30d`              |   0.032428 |
| `search_volume`          |   0.005518 |
| `competition`            |   0.005156 |

The strongest signals were therefore:

1. recent impressions
2. average search position
3. update age
4. CTR
5. sessions
6. users

Recent search visibility and search position dominated the Random Forest's decisions in this experiment — `days_since_last_update`, the signal this lane is named after, contributed only about 5% of the model's weight.

Feature importance should be interpreted as model behavior, not as causal importance.

---

# Recommendation Logic

The final ranking assigns each webpage a `priority_score`.

The score is then converted into an editorial action.

### HIGH_PRIORITY_REFRESH

Used for pages with a high predicted decline probability and meaningful search visibility.

These pages should be considered first for editorial review.

### REVIEW

Used for pages with moderate predicted decline probability.

These pages should receive additional manual review before deciding whether a refresh is appropriate.

### MONITOR

Used for pages where immediate refresh is not recommended by the decision logic.

These pages can remain under observation.

Very low-visibility pages are not automatically treated as high-value refresh opportunities even when their model score is high — pages below the `impressions_30d >= 100` floor are excluded from the eligible queue by design.

---

# Example Recommendation Output

The final recommendation output contains fields such as:

```text
rank
content_hash_id
priority_score
action
reason_code
days_since_last_update
impressions_30d
users_30d
ctr
avg_position
```

Example high-priority records from the final ranking showed meaningful visibility together with relatively weak CTR and/or poor average position.

These characteristics make them more useful candidates for human editorial review than pages with extremely low visibility.

---

# Final Outputs

Running `notebooks/capstone.ipynb` end to end regenerates the recommendation files under:

```text
work/outputs/
```

Files (generated locally at runtime — **not committed to git**, per Rule 2 above):

```text
final_refresh_recommendations.csv
top20_refresh_recommendations.csv
```

The top-20 shortlist is also reproduced as a markdown table inside `capstone_report.md`, so a reviewer can see the actual output without needing to re-run the notebook.

The detailed analysis is available in:

```text
work/capstone_report.md
```

The main experiment notebook is:

```text
work/notebooks/capstone.ipynb
```

---

# Capstone Report

The detailed capstone report covers:

1. Problem framing
2. Data safety
3. Baseline
4. Model / analysis
5. Evaluation
6. Interpretation
7. Recommendation
8. Reproducibility

See:

```text
capstone_report.md
```

for the complete analysis.

---

# Reproducibility

The main capstone experiment can be reproduced using:

```text
notebooks/capstone.ipynb
```

The notebook contains the workflow for:

1. connecting to the data environment
2. inspecting the available data
3. preparing the modelling dataset
4. checking data quality
5. defining the target
6. creating the baseline
7. preparing modelling features
8. splitting data by client
9. training the Random Forest
10. evaluating Precision@K
11. analysing feature importance
12. generating ranked recommendations
13. assigning editorial actions
14. exporting recommendation outputs (locally; not committed to git)

The Random Forest uses:

```text
random_state = 42
```

and the experiment should be run with the same environment and feature definitions used in the notebook.

---

# Repository Structure

```text
work/
│
├── notebooks/
│   ├── w01_research_question.ipynb
│   ├── w02_ml_task_framing.ipynb
│   ├── w03_data_contract.ipynb
│   ├── w03_feature_leakage_check.ipynb
│   ├── w04_signal_audit.ipynb
│   ├── w04_baseline_score.ipynb
│   ├── w05_model.ipynb
│   ├── w06_validation_audit.ipynb
│   ├── w07_action_playbook.ipynb
│   └── capstone.ipynb
│
├── outputs/                      (gitignored — regenerated by running capstone.ipynb)
│
├── README.md
├── capstone_report.md
└── capstone_report_template.md
```

---

# Assignment Index

The assignment notebooks are retained in the repository as the development trail for the project.

**Only tick a box once that notebook is actually filled in and saved in your repo** — an unchecked box here is safer than a checked one a reviewer can't find.

| Notebook                                    | Assignment          | Status |
| -------------------------------------------- | ------------------- | ------ |
| `notebooks/w01_research_question.ipynb`     | ML-02               | ☑      |
| `notebooks/w02_ml_task_framing.ipynb`       | ML-03               | ☑      |
| `notebooks/w03_data_contract.ipynb`         | ML-04               | ☑      |
| `notebooks/w03_feature_leakage_check.ipynb` | ML-05               | ☑      |
| `notebooks/w04_signal_audit.ipynb`          | ML-06               | ☑      |
| `notebooks/w04_baseline_score.ipynb`        | ML-07               | ☑      |
| `notebooks/w05_model.ipynb`                 | ML-08               | ☑      |
| `notebooks/w06_validation_audit.ipynb`      | ML-09               | ☑      |
| `notebooks/w07_action_playbook.ipynb`       | ML-10               | ☑      |
| `notebooks/capstone.ipynb`                  | ML-11               | ☑      |

---

# Limitations

The results should be interpreted with the following limitations:

1. The target represents observed search-performance decline and does not establish causality.

2. A high priority score does not guarantee that refreshing a page will improve search performance.

3. The Random Forest did not outperform the baseline at K=10, K=20, or K=50 on the cleaned holdout split.

4. The Random Forest showed an advantage over the baseline at K=100.

5. An earlier cross-validation pass, run before a data-quality bug was fixed, showed a much larger (and less trustworthy) model advantage — see `capstone_report.md` Section 5.1.

6. Model performance can vary across client-grouped evaluation folds; cross-validation standard deviations were large.

7. Pages with extremely low visibility may receive high model scores but may not represent the best business opportunity for immediate editorial work.

8. Recommendations should be reviewed by a human editor before content changes are made.

9. The model should not be interpreted as predicting or reproducing Google's ranking algorithm.

---

# Interpretation Policy

This project uses the following language when describing findings:

* observed
* measured
* directional
* decision-support

The project does not claim:

* causal impact
* guaranteed traffic improvement
* guaranteed ranking improvement
* prediction of Google's ranking algorithm

The output is a prioritization tool to help humans decide where to investigate first.

---

# Recommended Editorial Workflow

A FlyRank editor can use the output as follows:

```text
1. Open the ranked recommendations
        ↓
2. Start with HIGH_PRIORITY_REFRESH pages
        ↓
3. Check impressions, CTR and average position
        ↓
4. Review content freshness and page quality
        ↓
5. Decide whether a refresh is actually appropriate
        ↓
6. Record the editorial decision
        ↓
7. Monitor future search performance
```

The model should therefore be treated as a **ranking and prioritization aid**, not an automatic content-update system.

---

# Submission

The capstone report is:

```text
work/capstone_report.md
```

The main capstone notebook is:

```text
work/notebooks/capstone.ipynb
```

The final recommendation outputs are generated locally (not committed) at:

```text
work/outputs/final_refresh_recommendations.csv
work/outputs/top20_refresh_recommendations.csv
```

When the paper is deployed, put its exact URL in:

```text
../submission/paper_url.txt
```

The URL should contain one exact paper URL on a single line.

---

# Final Capstone Status

**Lane:** Search Intelligence — Content Refresh Prioritization

**Research Question:** Which webpages are most likely to experience declining search performance and should be prioritized for content refresh?

**Primary Model:** Random Forest Classifier

**Primary Metric:** Precision@K

**Validation:** Client-grouped train/test split

**Best reported model result:** Precision@100 = 0.80

**Baseline Precision@100:** 0.72

**Observed improvement at K=100:** +0.08

**Decision Output:** Ranked content refresh recommendations

**Human Review:** Required

**Causal Claims:** None

**Google Ranking Algorithm Prediction:** None
