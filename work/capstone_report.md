# Capstone Report — Lane 2: Refresh / Content Opportunity Scoring

- **Author:** Nashidul Sarker
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repo:** [https://github.com/NashidulSarker/flyrank-internship](https://github.com/NashidulSarker/flyrank-internship)
- **Deployed Research Paper:** [https://nashidulsarker.github.io/flyrank-internship/](https://nashidulsarker.github.io/flyrank-internship/)
- **Date:** March 2026

---

## 1. Problem Framing (The FlyRank Case Study)

### Decision Context & Production Dilemma
At FlyRank, digital publishing clients monitor vast portfolios containing tens of thousands of indexed URLs. The platform's automated audits surface thousands of potential content update and refresh flags each month. However, enterprise content operations face a severe human bandwidth bottleneck: an editorial squad or SEO agency team can thoroughly audit, update, and rewrite only 20 to 50 URLs per week.

Standard industry practice relies on blunt, unguided rules—such as reviewing all articles exceeding 90 days since last update. In multi-client production environments, this creates massive operational waste:
* **Cost of False Positives:** ~$25 (~0.5 editor hours) per URL spent auditing and rewriting stable evergreen content that already satisfies search intent, producing zero incremental traffic recovery.
* **Cost of False Negatives:** Failure to detect high-value, steadily declining URLs before organic impressions collapse, causing compound losses in organic search traffic and client revenue.

### Unit of Analysis & Operational Action
* **Unit of Analysis:** One unique content item (`content_hash_id`) belonging to a specific client (`client_hash_id`), evaluated over a trailing 90-day search and engagement performance window.
* **Output:** A priority rank and continuous risk probability score, mapped into discrete action archetypes (`refresh_content`, `optimize_title_ctr`, `refresh_and_expand`, `expand_depth`, `monitor`) with transparent reason codes and human-review requirements.
* **Operational Action:** Content editors take the top 20–50 recommendations from the weekly priority queue to perform targeted manual interventions (fact-checking, expanding thin sections, updating outdated statistics, or adjusting title/snippet tags).
* **Value of ML:** Machine learning captures multi-feature non-linear interactions (e.g., search presence consistency × position tier × staleness), elevating precision among top-priority recommendations far above arbitrary rules.

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
* **Visibility Score:** Percentile rank of log-transformed 90-day impressions (`log(1 + impressions_90d)`).
* **Freshness Risk Score:** Percentile rank of days since last update (`days_since_last_update`).
* **Position Risk Score:** Risk penalty for URLs in Page 1 and striking distance positions (average position between 1 and 20 via `avg_position`).
* **CTR Gap Score:** Penalty for URLs with below-median CTR despite substantial impression demand (&ge; 100 impressions).

### Baseline Performance on Held-Out Test Clients
* **Base Rate (Decline Rate):** 52.50% (0.5250)
* **Precision@20:** 0.4000 (40.0%)
* **Precision@50:** 0.4400 (44.0%)
* **Precision@100:** 0.4900 (49.0%)
* **Average Precision (AP):** 0.5505
* **ROC-AUC:** 0.5689

*Why the baseline struggles:* Heuristic rules over-penalize age alone, mistakenly flagging high-authority evergreen content that remains stable, while missing active decay in volatile mid-position URLs.

---

## 4. Model / Analysis

### Model Selection & Rationale
We evaluated four candidate models from our toolkit against the baseline:
1. **Logistic Regression (L2 Scaled):** Linear benchmark for coefficient interpretability.
2. **Decision Tree (d=5):** Interpretable hierarchical decision tree exposing threshold boundaries.
3. **Random Forest (n=100, max depth=10):** Bagged ensemble reducing feature variance.
4. **Gradient Boosting (n=100, max depth=4):** Non-linear boosting ensemble capturing interaction effects.

### Feature Catalog (19 Non-Leaking Signals)
* **Search Visibility & Demand:** `impressions_90d`, `clicks_90d`, `ctr`, `avg_position`, `days_with_impressions`.
* **User Engagement & Analytics:** `pageviews_90d`, `sessions_90d`, `users_90d`, `engaged_sessions_90d`, `scroll_events_90d`, `days_with_sessions`, `engagement_rate`, `scroll_rate`, `ai_sessions_90d`, `ai_traffic_pct`.
* **Content Metadata & Freshness:** `content_age_days`, `days_since_last_update`, `word_count`, `char_count`.

---

## 5. Evaluation

### Validation Strategy: Client-Holdout Split
To simulate true production deployment—where models prioritize queues for new, unseen client domains—we utilized a **Client-Holdout Split** (80% train clients, 20% test clients):
* **Train Set:** 26 client portfolios (n = 26,619 URLs | Base rate: 54.42%)
* **Test Set:** 6 held-out client portfolios (n = 3,381 URLs | Base rate: 52.50%)

### Model vs Baseline Comparison Table (Held-Out Test Clients)

| Model Strategy | Base Rate | Precision@20 | Precision@50 | Precision@100 | Average Precision (AP) | ROC-AUC | Lift over Base Rate (P@50) |
|---|---|---|---|---|---|---|---|
| **Baseline (Rule)** | 0.5250 | 0.4000 | 0.4400 | 0.4900 | 0.5505 | 0.5689 | -16.2% |
| **Logistic Regression** | 0.5250 | 0.8000 | 0.6800 | 0.7000 | 0.6193 | 0.6162 | +29.5% |
| **Decision Tree (d=5)** | 0.5250 | 0.5500 | 0.6600 | 0.6600 | 0.6258 | 0.6565 | +25.7% |
| **Random Forest (n=100)** | 0.5250 | 0.4000 | 0.3600 | 0.4300 | 0.6252 | 0.6646 | -31.4% |
| **Gradient Boosting (n=100) ★** | **0.5250** | **0.8500** | **0.8400** | **0.7600** | **0.6829** | **0.6901** | **+60.0% (1.60x)** |

### Error Analysis (3 Concrete Test Cases)
1. **False Positive (`content_9b2575f9efdd`):** Predicted risk 81.2%, Actual label 0. URL at striking position 11.4 with 1,612 impressions. Model expected decay, but a recent minor update 15 days prior stabilized performance.
2. **False Negative (`content_06248e69dbfe`):** Predicted risk 10.6%, Actual label 1. Top-5 position URL updated 20 days prior, but with tiny total volume (n = 2 impressions). Measurement variance on low impressions triggered the percentage drop label.
3. **Borderline Miss (`content_167c1e549117`):** Predicted risk 49.8%, Actual label 1. Moderate search activity (4,210 impressions, pos 14.2) sat right on the decision boundary, experiencing decay from competitor keyword moves.

---

## 6. Interpretation

### Feature Importance Drivers
1. **`days_with_impressions` (Tree Imp: 0.354, Permutation Imp: 0.146):** The primary signal. URLs with intermittent impressions (<45 active days) exhibit significantly higher decline rates than daily search anchors.
2. **`content_age_days` (Tree Imp: 0.207):** Older articles experience steady-state stability, whereas young articles undergo volatile rank adjustments.
3. **`avg_position` (Tree Imp: 0.095):** Striking-distance URLs (pos 10–25) decay significantly faster than top-3 URLs.
4. **`impressions_90d` & `ctr`:** Establish demand scale and click efficiency.

### Negative & Surprising Results
* **Word Count Counter-Intuition:** In our signal audit, thin articles (<1,000 words) showed an observed decline rate of only 20.7%, compared to 59.7% for long-form articles (>3,500 words). Heuristic rules that automatically penalize short articles are heavily biased; thin content often serves focused navigational queries that rarely decay.

---

## 7. Recommendation (Content Action Playbook)

### Editorial Decision Matrix & Reason Codes

| Archetype | Condition / Reason Code | Action Label | Editorial Action |
|---|---|---|---|
| **Stale High-Demand** | `high_model_risk_stale` (Prob >= 0.70, Days >= 90) | `refresh_content` | Fact-check, update statistics, refresh outdated sections |
| **Striking Distance Decay** | `striking_distance_decay` (Prob >= 0.60, Pos 10–25) | `refresh_and_expand` | Add missing subtopics to push URL to Page 1 |
| **Underperforming CTR** | `ctr_underperformance` (Prob >= 0.60, Pos <= 20, CTR < 0.5%) | `optimize_title_ctr` | Rewrite title tag and meta snippet for CTR |
| **Thin Visible Content** | `thin_content_gap` (Words < 1,200, Imps >= 250) | `expand_depth` | Deepen content coverage for broader search intent |
| **Stable / Low Risk** | `routine_monitoring` (All other inventory) | `monitor` | Retain in automated monitoring; no manual action |

### Mandatory Review Triggers & Strict NO-GO List
* 🚫 **No Autonomous CMS Overwrites:** Direct LLM publishing to live URLs without human sign-off is prohibited.
* 🚫 **Top-3 Ranking URLs (`avg_position` <= 3.0):** Protected core revenue assets (1,113 URLs in dataset). Require mandatory senior SEO review before edits.
* 🚫 **High-Traffic Assets (>= 50,000 imps):** Mandatory multi-stakeholder approval.
* 🚫 **Seasonal Queries:** Holiday and event traffic dips must not trigger permanent content rewrites.

### Retraining & Drift Triggers
* **Precision Drift:** Trigger retrain if human-verified Precision@50 drops below 70.0%.
* **Google Core Update:** Mandatory re-calibration 30 days following major core search algorithm updates.
* **Feature Shift:** Re-scale normalization parameters if portfolio-wide mean staleness shifts by > 15%.

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

## 9. 5-Minute Showcase Demo Outline (Week-8 Ready)

1. **Minute 1: The Problem & The FlyRank Dilemma**
   - The Hook: *"Every digital publisher knows content decays, but no editorial team has the hours to rewrite thousands of articles every month."*
   - Setting: FlyRank monitors multi-client portfolios; editorial squads can review only 20–50 URLs/week.
   - Cost: Fixed 90-day staleness rules waste $25/URL on evergreen pages while missing active decay in revenue drivers.
2. **Minute 2: The 79M Warehouse & Honest Validation Design**
   - Data Scale: 30,000 URLs across 32 clients from the 79M-row warehouse.
   - 19 Pre-decision features (impressions, positions, days with impressions, CTR, engagement).
   - Client-Holdout Split: Holding out 6 unseen client portfolios (20% of domains, n = 3,381 URLs) prevents site template and domain authority leakage.
3. **Minute 3: The Model & The Winning Result (Figure 1: `model_vs_baseline.png`)**
   - Baseline rule achieved 44.0% Precision@50 (worse than the 52.5% random base rate).
   - Gradient Boosting achieved **84.0% Precision@50** (+31.5% lift over base rate, 1.60x precision boost).
4. **Minute 4: The Skeptic's Catch & Honest Limitations**
   - Thin-content discovery: Articles <1,000 words have a 20.7% decline rate vs 59.7% for >3,500 words.
   - Observational association, non-causal framing, zero autonomous publishing.
5. **Minute 5: The Action Playbook & Operational Impact (Figure 2: `action_mix.png`)**
   - 5 Action archetypes mapping directly to editor tasks.
   - Flags 1,052 actionable URLs (~3.5% of inventory) matching real team capacity.
   - Strict No-Go rules protecting Top-3 ranking URLs and >50k impression assets.

---

## 10. Shareable Cuts of the Work

### Cut 1: Methodology Social Post (LinkedIn / X / Community)
> 🔍 **How we doubled content refresh precision on 79M production search rows:**
>
> Most SEO teams rely on arbitrary staleness rules (e.g., *"refresh every article older than 90 days"*). Across 30,000 production URLs, we discovered this blunt heuristic achieves only **44.0% Precision@50** on held-out client domains—actually worse than the unguided **52.5% base rate**—wasting hundreds of editor hours on stable evergreen content.
>
> By engineering 19 pre-decision search visibility & engagement signals and training a Gradient Boosting priority ranker evaluated strictly under an honest **Client-Holdout Split** (testing on completely unobserved client portfolios), our model achieves **84.0% Precision@50** (+31.5% lift over base rate, 0.690 ROC-AUC).
>
> Rather than handing editors raw probabilities, we operationalized predictions into a human-in-the-loop **Content Action Playbook** featuring 5 discrete decision archetypes, reason codes, and strict No-Go safeguards for protected Top-3 ranking assets.
>
> 📄 **Live Research Paper:** https://nashidulsarker.github.io/flyrank-internship/  
> 💻 **GitHub Repo & Notebooks:** https://github.com/NashidulSarker/flyrank-internship  
> 🌐 *Built on the FlyRank ML Internship dataset (https://flyrank.ai/)*

### Cut 2: Employer-Facing 3-Sentence Summary
* **What I Built:** A machine learning decision-support ranking system and operational Content Action Playbook that prioritizes weekly content refresh queues for digital publishing and SEO teams.
* **On What Data:** Evaluated on 30,000 unique content URLs across 32 client domains extracted from FlyRank's 79-million-row production search intelligence warehouse with zero target leakage.
* **What It Showed:** Gradient Boosting achieved **84.0% Precision@50** on unobserved client holdouts (vs a 52.5% random base rate and 44.0% rule baseline, delivering a 1.60× precision lift), successfully isolating the top 3.5% actionable inventory while enforcing strict No-Go protections on core ranking assets.

---

## 11. Acknowledgments & Data Credit

**Built on the FlyRank ML Internship dataset** — [https://flyrank.ai/](https://flyrank.ai/).  
We gratefully acknowledge FlyRank for providing access to anonymized multi-client production search performance logs and search intelligence infrastructure.
