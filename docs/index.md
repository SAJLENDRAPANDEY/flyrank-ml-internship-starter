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

The model produces a priority score and ranked recommendation. A human editor then decides whether a page should actually be refreshed.

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

A total of **47,365 rows** were removed because their update-date information could not be reliably reconstructed.

The public paper does not expose client names, domains, raw URLs, private queries, credentials, or raw warehouse exports.

---

## 3. Methodology

### Target

The decline label represents a directional month-over-month search-performance decline based on the change in impressions.

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

Future/label-derived fields were excluded to reduce leakage risk.

### Model

The primary final model was a:

**Random Forest Classifier**

with:

```text
n_estimators = 300
max_depth = 12
min_samples_leaf = 10
class_weight = balanced
random_state = 42
n_jobs = -1
