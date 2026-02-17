# campaign-optimization-sales-prediction-uplift-ml

# Outbound Campaign Optimization with Sales Prediction and Uplift Modeling

## Overview
This project evaluates an outbound calling campaign run by a Customer Advocate Center. The campaign targets newly registered customers who have not yet purchased, with the goal of answering questions about the buying process and providing shopping assistance. As the business scales, the cost of outreach grows, so the analysis focuses on quantifying campaign impact on sales and designing a strategy that improves productivity.

The core outputs are (1) an estimated impact of outbound calls on short-term purchasing, (2) an interpretable explanation of drivers associated with purchase likelihood and campaign response, and (3) a prioritized calling list that recommends which customers should be called and in what order. All data used in this project is fabricated for the exercise and does not reflect real client performance or business economics.

## Business Goal
Advocate Center leadership needs practical guidance on whether the campaign is worth running and how to run it more efficiently. The analysis addresses:
- Whether outbound calling is associated with higher short-term sales conversion
- Whether responsiveness differs by customer segments such as credit and income
- How to prioritize outreach to maximize value after accounting for dialing costs

## Data
Three datasets are used:
- CustomerHistory: one row per customer account with customer and account attributes measured three days after account creation
- OBCallHistory: one row per outbound call attempt (phone attempted, timestamp, contact outcome, call duration, Advocate ID)
- SaleHistory: one row per completed sale (sale timestamp and vehicle attributes such as odometer and sticker price)

## Campaign Rules and Attribution
Customers are eligible for outreach when an account exists and no purchase has occurred yet. For analysis, a sale is attributed to the campaign when a call attempt exists for that customer and a purchase occurs within 45 days of account creation.

## Data Preparation
All tables are loaded and joined into a customer-level analytical dataset. Call history is aggregated to capture outreach exposure (attempted) and outcomes (contacted). Sales history is linked to determine whether a purchase occurred within the attribution window.

Engineered features include:
- sold: 1 if a purchase occurs within 45 days of account creation, else 0
- high_credit: 1 if CreditScore1 is above the dataset median, else 0
- high_income: 1 if income is above the 80th percentile, else 0

## Descriptive Analysis
Sale conversion rates are computed for three outreach conditions:
- not attempted: no outbound call attempts recorded
- attempted: at least one outbound call attempt recorded
- contacted: at least one successful contact recorded

These comparisons provide an initial view of how outcomes differ across levels of campaign exposure.

## Statistical Testing
Observed differences in conversion rates can occur by chance, so hypothesis tests are used to assess whether gaps are statistically distinguishable from random variation. Subgroup analyses evaluate whether the association between calling and sales differs for:
- high_credit vs. not high_credit
- high_income vs. not high_income

## Baseline 45-day conversion rates by outreach status (Before Optimization)
- Not attempted: 29.04%
- Attempted: 40.71%
- Contacted: 43.41%

These are descriptive differences from historical data and do not by themselves prove causality, since customers who were called or reached may differ systematically from those not called.

## Predictive Modeling (Prioritization)
A predictive model is trained to estimate each customer’s probability of purchasing within 45 days. The objective is predictive accuracy and generalizability to support ranking customers for outreach. Model validation is performed using a holdout or cross-validation approach to estimate future performance.

The output of this component is a purchase propensity score per customer.

## Interpretable Modeling (Call Impact)
A separate interpretable model is built to describe the relationship between outbound calling and probability of sale while controlling for confounding customer attributes. This component supports understanding of feature relationships, estimated call impact, and potential effect variation across customer characteristics.

Because the campaign is not randomized, results are interpreted as associations rather than definitive causal effects.

## Calling Strategy and ROI
A call strategy is created to improve productivity by:
- ranking customers by expected purchase propensity or incremental value
- flagging customers recommended for calling when expected benefit exceeds cost

Expected improvement is estimated relative to the current approach by comparing:
- projected lift in conversion rate
- projected revenue impact using available sale-related fields when appropriate
- net value after dialing costs, assuming a cost of $0.10 per dial

## Deliverables
This repository contains:
- business_presentation.pptx: short slide deck for non-technical stakeholders summarizing findings, recommendations, and ROI
- technical_writeup.md (or .pdf): brief documentation of data preparation, testing, modeling choices, and validation
- data/processed/: intermediate datasets created during cleaning, joining, and feature engineering
- src/ or notebooks/: reproducible code to regenerate results with minimal modifications

## Assumptions
- All datasets are synthetic and fabricated for the exercise
- Customer attributes are treated as measured consistently three days after account creation
- The 45-day window is applied relative to account creation for sold and attribution
- Call attempts and contact outcomes are aggregated at the customer level to match customer-level outcomes
- Any operational constraints (capacity limits, calling hours) can be incorporated if available

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
