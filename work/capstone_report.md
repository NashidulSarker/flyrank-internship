# Capstone Report — Lane 2: Refresh / Content Opportunity Scoring

- **Author:** Nashidul Sarker
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repo:** [https://github.com/NashidulSarker/flyrank-internship](https://github.com/NashidulSarker/flyrank-internship)
- **Deployed Research Paper:** [https://nashidulsarker.github.io/flyrank-internship/](https://nashidulsarker.github.io/flyrank-internship/)
- **Date:** March 2026

---

## 1. Problem Framing

### Decision Context
In digital publishing and enterprise SEO operations, managing large-scale content inventories across multi-client portfolios requires continuous editorial upkeep. However, editorial teams face severe bandwidth limitations: a human editor can thoroughly audit, update, and rewrite only 20 to 50 URLs per week. Standard practice relies either on unguided manual intuition or coarse, arbitrary heuristics (e.g., refreshing all content older than 90 days), resulting in substantial wasted effort.

### Unit of Analysis & Operational Action
* **Unit of Analysis:** One unique content item (`content_hash_id`) belonging to a specific client (`client_hash_id`), evaluated over a trailing 90-day search and engagement performance window.
* **Output:** A priority rank and continuous risk probability score, mapped into discrete action archetypes (`refresh_content`, `optimize_title_ctr`, `refresh_and_expand`, `expand_depth`, `monitor`) with transparent reason codes and human-review requirements.
* **Operational Action:** Editorial teams pull the top-ranked URLs each week to conduct targeted manual interventions (fact-checking, expanding thin sections, updating outdated statistics, or adjusting title/snippet tags).

### Error Economics & Value of ML
* **Cost of False Positives:** ~$25 (0.5 editor hours) per URL spent researching and rewriting evergreen content that was already stable, yielding zero incremental traffic recovery.
* **Cost of False Negatives:** Missed opportunity to rescue high-authority, revenue-driving URLs from steady search decay, resulting in cumulative loss of organic visibility and conversion revenue.
* **Value of ML:** Machine learning captures multi-feature non-linear interactions (e.g., search presence consistency $\times$ position tier $\times$ staleness), elevating precision among top-priority recommendations far above arbitrary rules.

---

## 2. Data Safety

### Data Scope & Public Safety
* **Source:** Extracted from FlyRank's 79-million-row search intelligence warehouse (`dim_content`, `fact_content_daily_performance`), evaluated on an anonymized representative slice of 30,000 unique content items across 32 distinct client portfolios (`data/raw/content_refresh_anonymized.csv`).
* **Zero PII / Anonymization:** All raw URLs, client domain names, client names, search queries, and proprietary identifiers are fully anonymized (`client_hash_id`, `content_hash_id`). No private data appears anywhere in code or documentation.

### Feature Exclusions & Leakage Prevention
To prevent circular logic and target leakage:
1. **Target-Derived Columns Excluded:** `trend_direction`, `trend_pct`, and `is_declining_label` are reserved strictly as labels and omitted from feature matrices.
2. **Post-Decision Performance Windows Excluded:** Narrow trailing comparison windows (`impressions_last_30d`, `clicks_last_30d`, `sessions_last_30d`, `impressions_prev_30d`) are excluded to avoid temporal leakage.
3. **Internal Product Flags Excluded:** FlyRank's rule-based decision outputs (`health_score`, `priority_score`, `action_type`) are omitted to ensure the model learns raw search dynamics rather than reverse-engineering heuristic outputs.
4. **Client IDs Excluded from Feature Vector:** `client_id` is used exclusively for grouped cross-validation partitioning, never as a predictive feature.

---

## 3. Baseline

### Heuristic Rule Design
We established a transparent, non-leaking heuristic baseline score combining four normalized dimensions:
$$\text{Baseline Score} = 0.40 \cdot \text{Visibility} + 0.30 \cdot \text{Freshness Risk} + 0.20 \cdot \text{Position Risk} + 0.10 \cdot \text{CTR Gap}$$
* **Visibility Score:** Percentile rank of log-transformed 90-day impressions ($\log(1 + \text{impressions\_90d})$).
* **Freshness Risk Score:** Percentile rank of days since last update (`days_since_last_update`).
* **Position Risk Score:** Risk penalty for URLs in Page 1 and striking distance positions ($1 \le \text{avg\_position} \le 20$).
* **CTR Gap Score:** Penalty for URLs with below-median CTR despite substantial impression demand ($\ge 100$ impressions).

### Baseline Performance on Held-Out Test Clients
* **Base Rate (Decline Rate):** $52.50\%$ ($0.5250$)
* **Precision@20:** $0.4000$ ($40.0\%$)
* **Precision@50:** $0.4400$ ($44.0\%$)
* **Precision@100:** $0.4900$ ($49.0\%$)
* **Average Precision (AP):** $0.5505$
* **ROC-AUC:** $0.5689$

*Why the baseline struggles:* Heuristic rules over-penalize age alone, mistakenly flagging high-authority evergreen content that remains stable, while missing active decay in volatile mid-position URLs.

---

## 4. Model / Analysis

### Model Selection & Rationale
We evaluated four candidate models from our toolkit against the baseline:
1. **Logistic Regression (L2 Scaled):** Linear benchmark for coefficient interpretability.
2. **Decision Tree ($d=5$):** Interpretable hierarchical decision tree exposing threshold boundaries.
3. **Random Forest ($n=100$, max depth=10):** Bagged ensemble reducing feature variance.
4. **Gradient Boosting ($n=100$, max depth=4):** Non-linear boosting ensemble capturing interaction effects.

### Feature Catalog (19 Non-Leaking Signals)
* **Search Visibility & Demand:** `impressions_90d`, `clicks_90d`, `ctr`, `avg_position`, `days_with_impressions`.
* **User Engagement & Analytics:** `pageviews_90d`, `sessions_90d`, `users_90d`, `engaged_sessions_90d`, `scroll_events_90d`, `days_with_sessions`, `engagement_rate`, `scroll_rate`, `ai_sessions_90d`, `ai_traffic_pct`.
* **Content Metadata & Freshness:** `content_age_days`, `days_since_last_update`, `word_count`, `char_count`.

---

## 5. Evaluation

### Validation Strategy: Client-Holdout Split
To simulate true production deployment—where models prioritize queues for new, unseen client domains—we utilized a **Client-Holdout Split** (80% train clients, 20% test clients):
* **Train Set:** 26 client portfolios ($n=26,619$ URLs | Base rate: $54.42\%$)
* **Test Set:** 6 held-out client portfolios ($n=3,381$ URLs | Base rate: $52.50\%$)

### Model vs Baseline Comparison Table (Held-Out Test Clients)

| Model Strategy | Base Rate | Precision@20 | Precision@50 | Precision@100 | Average Precision (AP) | ROC-AUC | Lift over Base Rate (P@50) |
|---|---|---|---|---|---|---|---|
| **Baseline (Rule)** | 0.5250 | 0.4000 | 0.4400 | 0.4900 | 0.5505 | 0.5689 | $-16.2\%$ |
| **Logistic Regression** | 0.5250 | 0.8000 | 0.6800 | 0.7000 | 0.6193 | 0.6162 | $+29.5\%$ |
| **Decision Tree ($d=5$)** | 0.5250 | 0.5500 | 0.6600 | 0.6600 | 0.6258 | 0.6565 | $+25.7\%$ |
| **Random Forest ($n=100$)** | 0.5250 | 0.4000 | 0.3600 | 0.4300 | 0.6252 | 0.6646 | $-31.4\%$ |
| **Gradient Boosting ($n=100$)** | **0.5250** | **0.8500** | **0.8400** | **0.7600** | **0.6829** | **0.6901** | **+60.0% (1.60×)** |

### Error Analysis (3 Concrete Test Cases)
1. **False Positive (`content_9b2575f9efdd`):** Predicted risk $81.2\%$, Actual label $0$. URL at striking position 11.4 with 1,612 impressions. Model expected decay, but a recent minor update 15 days prior stabilized performance.
2. **False Negative (`content_06248e69dbfe`):** Predicted risk $10.6\%$, Actual label $1$. Top-5 position URL updated 20 days prior, but with tiny total volume ($n=2$ impressions). Measurement variance on low impressions triggered the percentage drop label.
3. **Borderline Miss (`content_167c1e549117`):** Predicted risk $49.8\%$, Actual label $1$. Moderate search activity (4,210 impressions, pos 14.2) sat right on the decision boundary, experiencing decay from competitor keyword moves.

---

## 6. Interpretation

### Feature Importance Drivers
1. **`days_with_impressions` (Tree Imp: 0.354, Permutation Imp: 0.146):** The primary signal. URLs with intermittent impressions (<45 active days) exhibit significantly higher decline rates than daily search anchors.
2. **`content_age_days` (Tree Imp: 0.207):** Older articles experience steady-state stability, whereas young articles undergo volatile rank adjustments.
3. **`avg_position` (Tree Imp: 0.095):** Striking-distance URLs (pos 10–25) decay significantly faster than top-3 URLs.
4. **`impressions_90d` & `ctr`:** Establish demand scale and click efficiency.

### Negative & Surprising Results
* **Word Count Counter-Intuition:** In our signal audit, thin articles (<1,000 words) showed an observed decline rate of only $20.7\%$, compared to $59.7\%$ for long-form articles (>3,500 words). Heuristic rules that automatically penalize short articles are heavily biased; thin content often serves focused navigational queries that rarely decay.

---

## 7. Recommendation (Content Action Playbook)

### Editorial Decision Matrix & Reason Codes

| Archetype | Condition / Reason Code | Action Label | Editorial Action |
|---|---|---|---|
| **Stale High-Demand** | `high_model_risk_stale` (Prob $\ge 0.70$, Days $\ge 90$) | `refresh_content` | Fact-check, update statistics, refresh outdated sections |
| **Striking Distance Decay** | `striking_distance_decay` (Prob $\ge 0.60$, Pos 10–25) | `refresh_and_expand` | Add missing subtopics to push URL to Page 1 |
| **Underperforming CTR** | `ctr_underperformance` (Prob $\ge 0.60$, Pos $\le 20$, CTR $< 0.5\%$) | `optimize_title_ctr` | Rewrite title tag and meta snippet for CTR |
| **Thin Visible Content** | `thin_content_gap` (Words $< 1,200$, Imps $\ge 250$) | `expand_depth` | Deepen content coverage for broader search intent |
| **Stable / Low Risk** | `routine_monitoring` (All other inventory) | `monitor` | Retain in automated monitoring; no manual action |

### Mandatory Review Triggers & Strict NO-GO List
* 🚫 **No Autonomous CMS Overwrites:** Direct LLM publishing to live URLs without human sign-off is prohibited.
* 🚫 **Top-3 Ranking URLs (`avg_position` $\le 3.0$):** Protected core revenue assets (1,113 URLs in dataset). Require mandatory senior SEO review before edits.
* 🚫 **High-Traffic Assets ($\ge 50,000$ imps):** Mandatory multi-stakeholder approval.
* 🚫 **Seasonal Queries:** Holiday and event traffic dips must not trigger permanent content rewrites.

### Retraining & Drift Triggers
* **Precision Drift:** Trigger retrain if human-verified Precision@50 drops below $70.0\%$.
* **Google Core Update:** Mandatory re-calibration 30 days following major core search algorithm updates.
* **Feature Shift:** Re-scale normalization parameters if portfolio-wide mean staleness shifts by $>15\%$.

---

## 8. Reproducibility

### Execution & Environment
* **Python Environment:** Python 3.11 / 3.12 with `pandas>=2.2`, `numpy>=1.26`, `scikit-learn>=1.4`, `matplotlib>=3.8`, `duckdb>=1.0`.
* **Random Seeds:** `random_state=42` for all split partitions, cross-validations, and tree estimators.
* **Re-run Pipeline:**
  ```bash
  git clone https://github.com/NashidulSarker/flyrank-internship.git
  cd flyrank-internship
  python -m venv .venv
  .venv/Scripts/activate
  pip install -r requirements.txt
  python scripts/run_pipeline.py # Or run work/notebooks/capstone.ipynb top-to-bottom
  ```

---

## 9. Acknowledgments & Data Credit

**Built on the FlyRank ML Internship dataset** — [https://flyrank.ai/](https://flyrank.ai/).  
We gratefully acknowledge FlyRank for providing access to anonymized multi-client production search performance logs and search intelligence infrastructure.
