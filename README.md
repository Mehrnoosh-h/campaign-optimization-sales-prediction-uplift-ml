# campaign-optimization-sales-prediction-uplift-ml

# Outbound Campaign Optimization with Sales Prediction and Uplift Modeling

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













## Predictive Modeling (Prioritization)
A predictive model is trained to estimate each customer’s probability of purchasing within 45 days. The objective is predictive accuracy and generalizability to support ranking customers for outreach. Validation is performed using a holdout or cross-validation approach to estimate future performance. The output is a purchase propensity score per customer.

## Interpretable Modeling (Call Impact)
A separate interpretable model is built to describe the relationship between outbound calling and probability of sale while controlling for customer attributes that may confound the relationship. Because the campaign is not randomized, results are interpreted as associations rather than definitive causal effects.

## Calling Strategy and ROI
A call strategy is created to improve productivity by:
- ranking customers by expected purchase propensity or incremental value
- flagging customers recommended for calling when expected benefit exceeds cost

Expected improvement is estimated relative to the current approach by comparing:
- projected lift in conversion rate
- projected revenue impact using available sale-related fields when appropriate
- net value after dialing costs, assuming a cost of $0.10 per dial

## Deliverables
- business_presentation.pptx: short slide deck for non-technical stakeholders summarizing findings, recommendations, and ROI
- technical_writeup.md (or .pdf): brief documentation of data preparation, testing, modeling choices, and validation
- data/processed/: intermediate datasets created during cleaning, joining, and feature engineering
- src/ or notebooks/: reproducible code to regenerate results with minimal modifications

## Assumptions and Limitations
- All datasets are synthetic and fabricated for the exercise.
- Customer attributes are treated as measured consistently three days after account creation.
- The 45-day window is applied relative to account creation for sold and attribution.
- Call attempts and contact outcomes are aggregated at the customer level to match customer-level outcomes.
- Outreach is not randomly assigned; segment differences and call effects should be interpreted as associations.
- Operational constraints (capacity limits, calling hours) can be incorporated if available.

## Suggested Repository Structure
- data/
  - raw/ (optional or placeholder)
  - processed/
- notebooks/
- src/
- reports/
  - business_presentation.pptx
  - technical_writeup.md
- README.md
