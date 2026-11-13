# CIIRT153-Predictive_Analytics_Tool

## Model Architecture and Methodology

### 1. Data Filtering and 2026 Inflation Escalation
* **Empirical Data Cleansing:** CII RT-153 Appendix F contains 144 usable industrial projects, of which 109 reported measured field rework percentages and 35 reported only subjective 1-to-5 ratings. The model filters out all subjective-only records and corrupted non-numeric entries so that KNN matching relies strictly on verified rework measurements.
* **Inflation Escalation (2001 to 2026):** Because CII RT-153 was published in July 2001, historical project costs are compounded over 25 years at 3.0% annually, resulting in an escalation multiplier of approximately 2.094. This scales the baseline 10.0 million dollar pre-construction exposure anchor to 20.94 million dollars in 2026 currency, ensuring modern budgets are compared against historically equivalent project scopes.

### 2. Feature Vectorization and Weighted Euclidean Distance
Each project is mapped into a normalized three-dimensional feature space covering facility sector risk, compressed project cost, and engineering readiness:
* **Facility Sector Risk:** Subgroup mean rework rates normalized to the 4.4% dataset average. Chemical, petrochemical, and oil and gas facilities carry a multiplier of 1.25; pharmaceutical plants carry 1.10; power, paper, manufacturing, and metals facilities serve as the 1.00 baseline; water and wastewater treatment carry 0.85; transportation infrastructure carries 0.80; and general buildings carry 0.75.
* **Log-Transformed Cost:** Project budgets in 2026 dollars are compressed using a natural logarithm transformation so multi-hundred-million-dollar budgets do not geometrically overwhelm normalized percentage scales.
* **Engineering Completion:** Design completion tiers at the start of construction are converted into continuous interval midpoints: 0.10 for under 20% complete, 0.30 for 21% to 40%, 0.50 for 41% to 60%, 0.70 for 61% to 80%, 0.88 for 81% to 95%, and 0.98 for over 95% complete.

Similarity between a user's project and historical peers is measured using a correlation-weighted Euclidean distance. Engineering completion receives the highest importance weight (2.5) because RT-153 identified upstream design readiness as the strongest predictor of field rework. Facility sector receives a moderate weight (1.5) to capture process complexity, while log-cost receives a lower weight (0.8) to act strictly as a scale anchor, since project size alone has no statistically significant correlation with rework percentage.

### 3. Inverse-Distance KNN Regression and Winsorization
Rework percentage is predicted from the K closest historical projects using inverse-distance weighting, meaning closer historical matches carry proportionally greater influence over the final forecast. If an exact geometric match exists in the database (zero distance), the model assigns 100% of the weight to that identical precedent and drops all other neighbor weights to zero to prevent division-by-zero errors.

Predicted rework rates are clamped (Winsorized) between a 0.5% lower quality floor and an 18.0% upper outlier fence. This protects baseline staffing estimates from being distorted by isolated historical execution disasters (such as projects that suffered 25.0% to 26.8% rework).

### 4. Sub-Linear Scale Regularization and Project Exposure Index
On large capital projects, quality supervision exhibits economies of scale where senior inspection leadership covers broader scopes without duplicating management overhead. To reflect this, project cost is regularized using a sub-linear power-law elasticity exponent of 0.72. Effective scaled cost is then combined with the facility sector multiplier and engineering completion multiplier to produce the Project Exposure Index and baseline full-time equivalent (FTE) inspector headcount.

### 5. L2 Ridge Regularized Discipline Risk Weights
Raw craft risk multipliers are derived by dividing each discipline's share of total field rework hours (from CII RT-153 Table 6.4) by its average share of direct project workhours (from Appendix D). Because piping accounts for 64% of field rework hours while averaging 35% of direct labor hours, it carries a raw risk weight of 1.83, compared to 1.10 for Electrical and Instrumentation, 0.90 for Equipment Installation, 0.70 for Civil and Structural, and 0.60 for Other Crafts.

To prevent single-craft projects from routing nearly all inspectors to one trade and leaving secondary scopes unmonitored, the raw weights are regularized toward the mean baseline of 1.0 using an L2 Ridge shrinkage penalty of 0.35.

### 6. Two-Tier Discrete Inspector Rounding Engine
Because contractors cannot mobilize fractional personnel or leave active trades uninspected, continuous discipline quotas are converted into whole-number inspector counts using a two-tier rule:
* **Mandatory Hold-Point Floor Rule:** Any active trade on site (workhour share greater than 0%) with a calculated quota under 1.0—whether 0.10 or 0.99—is rounded up to 1 whole inspector. This ensures code-mandated witness and hold points are never left unstaffed.
* **GC Prudence Rule:** Once a discipline requires at least 1.0 inspector, standard arithmetic rounding applies: fractional demand below 0.5 rounds down to eliminate unnecessary overhead burden on the contractor, while fractional demand at or above 0.5 rounds up to maintain adequate field oversight.

---

## Defensive Gates and Protections

1. **Data Verification Gate:** Strips unmeasured subjective projects and corrupted records prior to distance computation.
2. **Simplex Conservation Gate:** Automatically rebalances craft density sliders so workhour shares always sum to exactly 100%.
3. **Domain Bounding Gate:** Clips budget inputs between 0.1 million and 5,000.0 million dollars to prevent logarithmic or division-by-zero singularities.
4. **Exact-Match Singularity Gate:** Assigns full weight to an exact historical match and zeroes out all other neighbor weights whenever Euclidean distance equals zero.
5. **Winsorization Outlier Gate:** Clamps predicted rework rates between 0.5% and 18.0% to neutralize extreme historical failure spikes.
6. **Sub-Linear Elasticity Gate:** Caps mega-project staffing growth via a 0.72 power-law scaling exponent.
7. **L2 Ridge Shrinkage Gate:** Prevents single-trade staffing monopolies via a 0.35 weight shrinkage factor.
