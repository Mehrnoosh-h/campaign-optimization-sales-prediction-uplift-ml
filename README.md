# campaign-optimization-sales-prediction-uplift-ml


## Overview
This project evaluates an outbound calling campaign run by a Customer Advocate Center. The campaign targets newly registered customers who have not yet purchased, aiming to answer questions about the purchase process and offer assistance. As the business scales, outreach costs increase, so the analysis focuses on quantifying campaign impact on sales and designing a strategy that improves productivity.

Key outputs include: (1) measurement of how outreach status relates to conversion, (2) a predictive model for prioritizing customers, (3) an interpretable model to describe the relationship between calls and sales while controlling for confounders, and (4) a ranked calling list with a call/no-call recommendation based on expected value. All data used in this project is synthetic and created for an exercise.

## Business Goal
- Evaluate whether outbound outreach is associated with higher conversion.
- Identify segments (credit and income) where the lift differs.
- Optimize the calling strategy to maximize net value after dialing cost ($0.10 per dial).

## Data
Three datasets are used:
- CustomerHistory: one row per customer account with customer and account attributes measured three days after account creation.
- OBCallHistory: one row per outbound call attempt (phone attempted, timestamp, contact outcome, call duration, Advocate ID).
- SaleHistory: one row per completed sale (sale timestamp and vehicle attributes such as odometer and sticker price).

## Campaign Rules and Attribution
- Eligibility: customers have an account and have not purchased yet.
- Attribution: a sale is attributed to the campaign when a call attempt exists for that customer and the purchase occurs within 45 days of account creation.

## Data Preparation and Feature Engineering
All tables are loaded and joined into a customer-level analytical dataset. Call history is aggregated to capture outreach exposure (attempted) and outcomes (contacted). Sales history is linked to determine whether a purchase occurred within the attribution window.

Engineered features:
- sold: 1 if a purchase occurs within 45 days of account creation, else 0
- high_credit: 1 if CreditScore1 is above the dataset median, else 0
- high_income: 1 if income is above the 80th percentile, else 0
- credit_mean: mean of CreditScore1 and CreditScore2 (row-wise, ignoring missing values)
- credit_gap: absolute difference between CreditScore1 and CreditScore2
- price_to_income: MedianVehiclePrice divided by (Income + 1) to avoid division by zero

One-hot encoded categorical columns: 
- CensusGeoRegionAddress
- LeadSource
- ModeDeviceType
- ModeVehicleType
- PhoneType
- ServiceProvider
- AgeCategory
- InboundCallContact
- ClicksQuintile

## Baseline Conversion by Outreach Status (Before Optimization)
Conversion rates are computed for three outreach conditions:
- not attempted: no outbound call attempts recorded
- attempted: at least one outbound call attempt recorded
- contacted: at least one successful contact recorded

Baseline conversion rates:
- Not attempted: 29.04%
- Attempted: 40.71%
- Contacted: 43.41%

These are descriptive differences and do not by themselves prove causality, since outreach is not randomly assigned.

## Statistical Validation of Baseline Differences
Observed gaps in conversion can occur by chance, so multiple checks are used to assess statistical reliability:
- Two-proportion z-test and an equivalent 2×2 chi-square test to compare conversion between groups.
- A 95% confidence interval for the absolute lift (difference in conversion rates) to quantify effect size and uncertainty.
- A permutation (randomization) test that shuffles group labels many times to estimate how often a lift as large as observed would occur under random labeling.

Example result from the permutation test:
- Observed lift: 0.1167 (≈11.7 percentage points)
- Permutation p-value: 5.0e-05

This indicates the observed lift is extremely unlikely under random labeling alone.

## Conversion Lift Heterogeneity by Credit Score and Income
This section checks whether lift differs across customer segments (high_credit vs not, high_income vs not), and whether differences are likely due to chance.
Note: in this section, `treat` refers to the attempted indicator (1 = attempted, 0 = not attempted).

Methods:
- Segment definition: create high_credit (CreditScore1 above median) and high_income (Income above 80th percentile).
- Stratified lift estimation: within each segment, compare conversion for contacted customers (treatment) vs not-attempted customers (control) and compute absolute lift.
- Significance + uncertainty: run a two-proportion z-test and compute a 95% confidence interval for lift within each segment.
- Formal heterogeneity test: fit an interaction logistic regression `sold ~ treat * segment` and evaluate the interaction term.

Results and conclusions:
- Credit: lift is large and statistically significant for non–high-credit customers, while high-credit customers show near-zero lift; the interaction model confirms the lift differs by credit segment.
- Income: lift is positive and statistically significant for both income groups, with a larger lift for high-income customers; the interaction model indicates the lift is statistically higher for high-income than for low-income customers.

## Predictive Modeling (XGBoost for Prioritization)

A gradient-boosted decision tree model (XGBoost) is used to predict purchase propensity and support customer ranking.

### Train/Validation/Test Split + Baseline XGBoost
The dataset is split into train, validation, and test sets using stratification to preserve class balance. A baseline XGBoost classifier is trained and evaluated using ranking metrics (ROC-AUC and PR-AUC) and threshold-based metrics (confusion matrix, precision, recall, and F1 at a default threshold) to establish initial out-of-sample performance.

### SMOTE + XGBoost (Class Imbalance Handling)
To address class imbalance, SMOTE is applied to the training set to oversample the minority class while keeping validation and test sets unchanged. The XGBoost model is retrained and re-evaluated to assess whether balancing improves minority-class detection and ranking performance (especially PR-AUC).

### Hyperparameter Tuning + Final XGBoost
A randomized hyperparameter search is run over key XGBoost settings (max_depth, min_child_weight, gamma, subsample, colsample_bytree, reg_alpha, reg_lambda), selecting the best configuration based on validation PR-AUC and using early stopping to reduce overfitting. The final tuned model achieves:

- BEST on VAL: PR-AUC = 0.6307  
- TEST ROC-AUC = 0.7320  
- TEST PR-AUC = 0.6168  
- Best parameters: `{'subsample': 1.0, 'reg_lambda': 2.0, 'reg_alpha': 0.5, 'min_child_weight': 5, 'max_depth': 5, 'gamma': 0, 'colsample_bytree': 0.6}`

### Ranking Performance (Tuned XGBoost)
To translate model scores into an actionable calling priority list, customers are ranked by predicted probability and evaluated using top-k precision and capture (recall-at-k). Results on the test set show that purchasers are concentrated near the top of the ranking:

- Top 5%: precision = 0.802, capture = 0.110  
- Top 10%: precision = 0.713, capture = 0.195  
- Top 20%: precision = 0.679, capture = 0.370  
- Top 30%: precision = 0.623, capture = 0.508  

Precision measures how many of the highest-ranked customers are purchasers, while capture measures what fraction of all purchasers are contained within the top-k slice.

### Feature Engineering: Affordability Alignment & Behavioral Ratios
To attempt performance gains without changing the model, additional ratio-based features are created to capture affordability alignment and browsing behavior relationships. These include:

- dreamer_gap = MedianVehiclePrice − Income  
- mileage_price_ratio = MedianVehicleMileage / (MedianVehiclePrice + 1)  
- searches_per_saved = NumberSearches / (NumberSavedVehicles + 1)  
- velocity30_per_search = VehicleVelocity30 / (NumberSearches + 1)

The tuned XGBoost model is then retrained using the same split and early stopping procedure. Test performance remains essentially unchanged (ROC-AUC ≈ 0.732, PR-AUC ≈ 0.620), indicating these engineered ratios did not add meaningful incremental predictive signal beyond the existing feature set for this dataset.

## Interpretable Modeling (Logistic Regression for Call Impact)
A logistic regression model is used to describe the relationship between outbound calling and conversion while controlling for customer attributes. Logistic regression is selected for interpretability: coefficients can be directly inspected to understand the direction and relative strength of associations, and interaction terms can be added to test whether the call relationship varies across segments.

### Multicollinearity Check (VIF)
To stabilize coefficients and avoid inflated standard errors, multicollinearity is screened using variance inflation factors (VIF) on the full design matrix (treatment indicator + numeric/binary controls + one-hot categorical controls). Highly collinear predictors are removed iteratively and VIF is recomputed after each update until the treatment variable (attempted) and remaining controls have acceptable VIF levels (most below ~10, with a small number slightly above but still usable). This reduces coefficient instability while preserving explanatory coverage.

Dropped columns due to collinearity:
- CreditScore1, CreditScore2, credit_mean
- MedianVehicleFuelEcon, MedianVehicleMileage, MedianVehiclePrice
- VehicleVelocity30, FraudScore

### Imbalance Handling: SMOTE + Logistic Regression
An `ImbPipeline` is used to keep preprocessing consistent while applying SMOTE only on the training split (median imputation → SMOTE → scaling → logistic regression). Test performance is modest:
- TEST ROC-AUC ≈ 0.580  
- TEST PR-AUC ≈ 0.425  

### Imbalance Handling: Class-Weighted Logistic Regression (No SMOTE)
A second logistic regression uses the same preprocessing but replaces SMOTE with `class_weight="balanced"` to address imbalance via weighted loss (median imputation → scaling → logistic regression). Results are similar:
- TEST ROC-AUC ≈ 0.581  
- TEST PR-AUC ≈ 0.424  

Overall, the class-weighted logistic regression is preferred for interpretation because it avoids synthetic samples while achieving comparable performance to the SMOTE version. Predictive performance from logistic regression is lower than the tuned XGBoost model, which remains the primary model for ranking and prioritization.

## Calling Strategy and ROI

### Goal
Use the selected model to decide whether the outbound calling program should continue and, if so, produce an ordered calling list that specifies which customers to attempt and in what priority order. The strategy is evaluated by estimating expected improvement in conversion relative to the current approach and translating that lift into expected revenue impact, then comparing expected benefit to dialing cost ($0.10 per attempt).

### Model Chosen (Ranking-Based Targeting)
Because the operational problem is capacity-constrained outreach, the model is selected based on ranking quality (PR-AUC and top-k precision/capture) rather than a single fixed classification threshold. XGBoost consistently outperforms logistic regression on these ranking metrics, making it the best choice for prioritizing calls. The model is used as a triage tool: customers are sorted by predicted purchase probability and called from the top of the list until capacity or ROI limits are reached.

### Strategy
A call strategy is created to improve productivity by:
- ranking customers by predicted purchase propensity (model score)
- selecting a calling cutoff (top-k or probability threshold) based on expected net value
- flagging customers recommended for calling and ordering them by priority

### ROI Estimation (How It Is Calculated)
After customers are ranked by the model score, ROI is evaluated by choosing a calling cutoff (top-k% or a score threshold) and comparing expected outcomes under that targeted calling rule versus the current approach.

- Projected change in conversion rate:
  - Expected buyers under a policy are estimated by summing predicted probabilities: `E[buyers] = Σ p_i` over the customers included by the calling rule.
  - The projected conversion rate is then `E[buyers] / N`.
  - Improvement is reported as the difference between the targeted policy conversion and the current baseline conversion.

- Projected revenue impact:
  - If a per-sale value proxy is available (e.g., sticker price or median vehicle price), expected revenue is computed as `E[revenue] = Σ (p_i × value_i)` for the targeted set (or using an average sale value when a customer-level value is not used).
  - Revenue uplift is the difference between targeted-policy expected revenue and the current baseline expected revenue.

- Net value after dialing costs:
  - Dialing cost is computed as `$0.10 × (# customers called)`.
  - Net value is `E[revenue] − dialing_cost`.
  - The program is considered worthwhile under a policy when incremental net value remains positive after costs (subject to operational capacity constraints).

## Repository Contents

The datasets used in this analysis are not included in this repository. 

- Notebooks/
  - campaign_impact_and_call_prioritization.ipynb
- README.md
