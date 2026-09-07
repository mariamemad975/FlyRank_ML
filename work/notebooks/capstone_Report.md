# Capstone Report — <your lane>

- **Author:** Mariam Emad Labib
- **Lane:** Prediction / Classification [search ranking decay detection]
- **Repo:** https://github.com/mariamemad975/FlyRank_ML
- **Date:** 


## 0. Abstract

Can machine learning reliably detect web pages at risk of search-ranking decay so that content teams can prioritize reviews? Using approximately 79 million rows of FlyRank search and engagement data, the study represents each content page through signals including click-through rate, search impressions, and query-footprint characteristics. A binary classification model was trained and tuned for high recall because missing a genuinely decaying page is more costly than reviewing a healthy page. The model achieved 81.9% accuracy and 97% recall for decaying pages, outperforming the 50.1% majority-class baseline. The resulting decay flag and probability score are used to create a prioritized human-review queue, helping SEO and content teams identify potentially declining pages before performance losses become more difficult to reverse.


## 1. Problem framing

Decision supported: Which content pages should an SEO/content team review first because their search performance may be decaying.

Unit of analysis: A single content page, represented through its query-context and engagement footprint in Google Search Console and related performance data.

Output: A binary decay flag with a probability score indicating the likelihood that a page is experiencing search-performance decay.

Human action: A content strategist uses the model output to prioritize pages into a review queue. The model does not automatically modify, remove, or optimize any page.

Cost of a wrong call: False negatives are more costly because a genuinely decaying page may continue losing search traffic without intervention. False positives mainly cost reviewer time, so the model was deliberately tuned toward high recall rather than high precision.

Why data/ML helps: Ranking decay is influenced by multiple interacting signals, including click-through rate, search impressions, and query-footprint changes. At the scale of FlyRank’s ~79M-row warehouse, these patterns are difficult to identify reliably using manually defined thresholds. A machine learning model can combine these signals to identify subtler decay patterns beyond the obvious cases captured by a rule-based baseline.

## 2. Data safety

Data used: Query-context and engagement features derived from FlyRank’s fact_content_daily_performance table (~79M rows), accessed through Hugging Face rather than downloaded in full locally.

Columns deliberately excluded: Client-identifying fields and pseudonymous IDs were excluded from model features. Pseudonymous IDs were used only for grouping, such as constructing the train/test split, to prevent the model from learning client- or page-specific identity patterns.

Leakage risks considered: Label-derived fields, including trend_direction and trend_pct, were excluded because they directly encode information related to the target. avg_gsc_position was initially tested but caused AUC to increase from a realistic ~0.70 to a suspicious 1.0, indicating leakage because it moved with the underlying signal used to define the decay label. It was therefore removed from the final feature set; this was the most important data-safety finding and is documented in the ML-05 notebook.

Confirmation: No client-identifying data, including domain names, query text, or client names, appears anywhere in work/. All analysis uses anonymized or aggregated features, with identifying fields excluded from model inputs.

## 3. Baseline

A transparent, hand-written rule-based baseline was developed first, using fixed thresholds on click-through rate and impression trends to flag pages that may be experiencing search-performance decay. This approach provides a simple and interpretable starting point before applying machine learning.

The baseline is a fair comparison because it uses the same underlying performance signals available to the ML model. The difference is that the baseline combines these signals using manually defined thresholds, while the ML model learns the relationships between features from the data.

Both approaches were evaluated on the same evaluation split using the same metrics. The majority-class baseline achieved 50.1% accuracy, providing a reference point for judging whether the ML model meaningfully improves over a simple benchmark.

## 4. Model / analysis

A Logistic Regression classifier was selected as the final model because it provides an interpretable approach that fits the classification lane and the human-in-the-loop purpose of the project. Its coefficients can be explained directly to a non-technical content team, helping reviewers understand which signals contribute to a page being flagged rather than treating the model as a black box.

The final model uses five query-context and engagement features derived through DuckDB aggregation from FlyRank’s approximately 79M-row warehouse. avg_gsc_position was deliberately excluded after the leakage issue identified in Section 2, along with other label-derived fields, to ensure that the model only uses valid predictive information.

The target is a binary label indicating whether a page’s search performance is trending downward over the observed period (decaying vs. not decaying). The model outputs both a binary decay prediction and a probability score, which can then be used to prioritize pages for human review.

## 5. Evaluation

The dataset was divided into an 80/20 train/test split using random_state=42. The evaluation was performed on an 81,841-row held-out test set from the full 409,205-record feature dataset. The same split was used to evaluate both the model and the baseline, ensuring a direct and consistent comparison.

The majority-class baseline achieved 50.1% accuracy, while the Logistic Regression model achieved 82.1% accuracy. For the decaying class, the model achieved 97% recall and 75% precision, showing that it identifies most pages experiencing decay while accepting some false positives.

The error analysis reflects the deliberate recall-oriented design. The model misses only about 3% of genuinely decaying pages, but approximately one-third of the pages it flags are healthy pages. This creates additional review work but is an acceptable trade-off for a screening system where missing a decaying page is more costly than reviewing a false alarm.

## 6. Interpretation

The model found that ctr_last30 was the strongest feature, while anon_share and rare_share had smaller contributions. visible_queries and top_query_share had minimal impact on the model’s predictions. This indicates that recent CTR behavior was the main signal associated with decay predictions.

<img width="707" height="470" alt="image" src="https://github.com/user-attachments/assets/ab720c8c-8fdd-435c-8831-67bc32de3353" />

A separate GA4 audit produced a mixed result: flagged pages had lower GSC clicks and CTR but higher GA4 sessions. This suggests that search-performance decline does not necessarily mean lower overall traffic, and no causal relationship is claimed.

## 7. Recommendation

The model output should be used to create a ranked review queue, with pages having the highest decay probability reviewed first. Each week, a FlyRank editor can select the top flagged pages and investigate potential content staleness, competitive changes, or technical issues, instead of manually reviewing the entire site catalog.

The results should be interpreted as directional, not causal: the model identifies correlated decay signals but does not establish their causes. It was trained on a single 60-day snapshot and is not a predictor of Google’s ranking algorithm. With approximately 33% of flagged pages being false alarms, the model should be treated as a screening tool for human review, not an autonomous decision-maker.


## 8. Reproducibility

```bash
git clone <https://github.com/mariamemad975/FlyRank_ML>
cd flyrank-ml-internship
pip install -r requirements.txt       
```
The full capstone workflow is available in work/notebooks/capstone.ipynb and can be run from top to bottom after configuring Hugging Face warehouse access through the HF_TOKEN environment variable and obtaining gated access to FlyRank/internship-warehouse. For the bundled sample data, the reference pipeline can be reproduced without warehouse access using:
```bash
python scripts/run_all.py
```
All train/test splitting and model training use random_state=42 for reproducibility. The project environment is defined through requirements.txt; any package versions or changes from the starter repository should be recorded there to ensure consistent reruns.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset —  ⁠FlyRank AI. Special thanks to the FlyRank Machine Learning Program and my mentor for providing the project structure, real-world data, and emphasis on transparent, evidence-based claims throughout this work.
