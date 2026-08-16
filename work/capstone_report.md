# Content Opportunity & Decay Scoring: A Machine Learning Approach to Proactive SEO

**Author:** Melih Yılmaz  
**Lane:** Refresh / Content Opportunity Scoring  
**Date:** August 2026

## 1. Problem Framing
Organic search traffic is highly volatile. By the time a company notices a significant drop in a page's traffic, revenue has already been lost. This project supports the decision of **proactive content refreshing**. 
- **Unit of analysis:** Individual web pages (anonymized via `content_hash_id`).
- **Output:** A ranked action engine that assigns a 'Reason Code' to decaying pages.
- **Action:** Editorial teams use this prioritized list to rewrite titles, update content, or perform competitor analysis before the decay worsens.
- **Cost of a wrong call:** Wasted editorial hours on pages that did not actually need a refresh. Machine learning minimizes this waste by identifying non-obvious decay patterns better than manual heuristics.

## 2. Data Safety
Data was sourced from the `fact_content_daily_performance` table. To strictly prevent **data leakage**, target-derived fields such as `curr_clicks`, and `click_change_pct` were entirely excluded from the feature set. Pseudonymous IDs (`client_hash_id`, `content_hash_id`) were utilized exclusively for grouping/windowing and were dropped before model training. No client names, raw URLs, or PII are exposed in this repository.

## 3. Baseline
Before applying ML, a transparent, human-readable heuristic (Baseline Rule) was established: *Predict decay (1) if `prev_ctr` < 0.01 AND `prev_avg_position` > 10*. 
This is a fair comparison because it represents a standard SEO alerting rule. Evaluated on the exact same 80/20 test split, this baseline achieved an **Accuracy of 52.12%** and a very poor **Recall of 21.35% (F1-Score: 0.2828)**, missing nearly 80% of actual decaying pages.

## 4. Model / Analysis
I trained an **XGBoost Classifier** on a time-aware split to capture non-linear relationships. 
- **Target Definition:** `is_decaying` = 1 if the page experienced a >= 20% drop in clicks between the previous 14-day window and the current 14-day window.
- **Features:** Strictly engineered from the *previous* 14-day window: `prev_clicks`, `prev_impressions`, `prev_ctr`, `prev_avg_position`, `prev_engagement_rate`, `prev_organic_ratio`.

## 5. Evaluation
Evaluated on the exact same 1,769-row test split, the XGBoost model achieved an **Accuracy of 52.35%** but increased **Recall to 51.69%**, yielding an **F1-Score of 0.5124**. 
- **Error Analysis:** While absolute accuracy remains near the base rate due to the inherently noisy and volatile nature of external search engine updates, the model successfully increased the detection rate of decaying pages by **over 2.4x (an 81.2% lift in F1-score)** compared to the baseline. This provides a strong *directional* and *decision-support* signal.

## 6. Interpretation
Feature importance extraction revealed a highly balanced model, indicating robustness against overfitting. 
1. `prev_organic_ratio` (18.0%) and `prev_clicks` (16.9%) were the strongest drivers. 
2. `prev_ctr` (15.3%) and `prev_engagement_rate` (16.6%) provided crucial quality signals. 
The balanced importance confirms that decay is a multivariate problem that simple rule-based baselines cannot effectively capture.

## 7. Recommendation
I built a **Ranked Action Engine** for the 857 pages predicted to decay, triaging them by `prev_clicks` to prioritize high-value traffic. The playbook outputs specific reason codes:
- **Title/Meta Rewrite (94.9% of flagged pages):** Low CTR (< 2%). Immediate action: A/B test new title tags.
- **Content Quality Refresh (4.9%):** Low engagement (< 10%). Immediate action: Improve readability, add multimedia, or update outdated facts.
- **Competitor Analysis (0.2%):** Traffic dropping despite healthy metrics. Immediate action: Manual SERP review for new competitor intent shifts.

## 8. Reproducibility
All code, queries, and random seeds (random_state=42) are documented sequentially in `work/refresh_scoring_capstone.ipynb`. The environment requires `duckdb`, `pandas`, `scikit-learn`, and `xgboost`.
