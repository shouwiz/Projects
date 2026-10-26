# Credit Risk Prediction & Decision Dashboard

## Business problem

Lenders must distinguish borrowers who are likely to repay from borrowers who may default. A model that misses risky borrowers can expose the lender to credit losses; a model that flags too many creditworthy borrowers can reject good customers and reduce lending opportunities.

This project uses the German Credit Risk dataset (1,000 records) to build a credit-risk classification workflow and evaluate how decision thresholds affect the trade-off between identifying bad-credit borrowers and incorrectly flagging good-credit borrowers.

**Business objective:** support risk-aware lending decisions by combining predictive performance, explainability and an explicit cost-sensitive decision threshold.

> **Scope note:** this is an educational modelling project using a small public dataset. Its results are not a production-ready lending policy and should not be interpreted as evidence about any particular lender or population.

## What I built

- Explored the data and examined relationships between borrower characteristics and the `good` / `bad` credit target.
- Split the data into stratified training and test samples.
- Built preprocessing pipelines with median imputation for numeric variables, most-frequent imputation and one-hot encoding for categorical variables, and scaling where appropriate.
- Compared Logistic Regression, Support Vector Machine (SVM), and XGBoost using stratified 5-fold cross-validation.
- Tuned XGBoost hyperparameters using ROC-AUC as the grid-search selection metric.
- Evaluated the final tuned XGBoost model on a held-out test set using accuracy, ROC-AUC, precision, recall, F1 and PR-AUC.
- Compared the default 0.50 threshold with a cost-sensitive threshold selected using illustrative misclassification costs.
- Reviewed model feature importance and used SHAP for model explainability.
- Created a probability-based risk scoring function and saved-model workflow for possible Streamlit integration.

## Dataset and target

- **Dataset:** German Credit Risk dataset
- **Observations:** 1,000
- **Target:** `Risk`, mapped to `1 = bad credit` and `0 = good credit`
- **Observed class distribution:** 700 good-credit records (70%) and 300 bad-credit records (30%)
- **Features include:** age, sex, job category, housing, savings account category, checking account category, credit amount, loan duration and loan purpose.

The notebook expects a local `german_credit_data.csv` file. Ensure that the dataset is obtained and used according to its applicable terms before rerunning the notebook.

## Model comparison

The notebook's five-fold cross-validation summary reported:

| Model | Accuracy | ROC-AUC | Bad-credit precision | Bad-credit recall | F1 | PR-AUC |
|---|---:|---:|---:|---:|---:|---:|
| Logistic Regression | 63.5% | 0.674 | 42.1% | 59.2% | 0.491 | 0.483 |
| SVM | 62.2% | 0.653 | 39.6% | 51.7% | 0.447 | 0.462 |
| XGBoost | 70.0% | 0.648 | 48.9% | 29.2% | 0.364 | 0.458 |

The selected XGBoost pipeline was tuned by ROC-AUC in cross-validation. **The model with the highest accuracy was not the model with the highest ROC-AUC or bad-credit recall.** This illustrates why model selection should reflect the business objective rather than accuracy alone.

On the held-out test set, the tuned XGBoost model at a 0.50 threshold reported:
- Accuracy: 71.5%
- ROC-AUC: 0.653
- Bad-credit precision: 56.5%
- Bad-credit recall: 21.7%
- F1: 0.313
- PR-AUC: 0.490

The notebook's test set contains 200 observations, including 60 bad-credit records.

## Cost-sensitive threshold analysis

The notebook tested candidate thresholds from 0.10 to 0.90 and used illustrative costs:

- False negative (a bad-credit borrower classified as good): cost weight **5**
- False positive (a good-credit borrower classified as bad): cost weight **1**

Under these assumptions, the selected threshold was approximately **0.19**.

| Test-set measure | Threshold 0.50 | Threshold 0.19 |
|---|---:|---:|
| True negatives | 130 | 30 |
| False positives | 10 | 110 |
| False negatives | 47 | 2 |
| True positives | 13 | 58 |
| Bad-credit recall | 21.7% | 96.7% |
| Bad-credit precision | 56.5% | 34.5% |
| Illustrative weighted error cost | 245 | 120 |

**Interpretation:** lowering the threshold identified 45 more bad-credit cases in the test sample (58 versus 13), while also increasing false positives from 10 to 110. Under the chosen illustrative cost weights, the total weighted error cost fell from 245 to 120. The trade-off is substantial: many more good-credit borrowers are flagged.

The cost weights are assumptions, not calibrated financial loss estimates. In real lending, the threshold should be selected using validated business costs, loss given default, exposure at default, policy constraints and out-of-time validation. This threshold comparison was calculated on the held-out test data, so it should be treated as exploratory; final performance should be confirmed on a separate validation set or through nested / out-of-time evaluation.

## Business insights

1. **Accuracy alone can hide risk.** At the default 0.50 threshold, the tuned XGBoost model achieved 71.5% accuracy but identified only 21.7% of bad-credit cases. In a credit-risk context, that recall may be insufficient if missing a risky borrower is costly.
2. **The decision threshold encodes business priorities.** The 0.19 threshold raised bad-credit recall to 96.7%, but bad-credit precision fell to approximately 34.5%. Threshold selection is therefore a business decision, not merely a technical setting.
3. **False positives have a real opportunity cost.** Flagging 110 good-credit borrowers at the lower threshold may lead to unnecessary rejection or manual review. The decision should balance expected credit losses against lost profitable lending opportunities.
4. **Loan duration was the largest feature-importance signal in this fitted XGBoost model.** Checking-account category, housing category, age and job also appeared among the leading features. These are model associations, not causal effects; they should be investigated for stability, fairness and plausibility before policy use.
5. **Explainability supports governance.** Feature importance and SHAP can help analysts understand which variables influence predictions, investigate unexpected behaviour and communicate model output to business stakeholders.
6. **Model choice depends on the objective.** Logistic Regression had the highest cross-validation ROC-AUC and recall among the three reported candidates, while XGBoost had the highest reported cross-validation accuracy and precision. A production selection would require a defined objective, calibration, stability testing and economic evaluation.

## Power BI dashboard

This repository includes three CSVs designed to import into Power BI:

- `credit_risk_model_comparison.csv` — cross-validation model comparison.
- `credit_risk_threshold_comparison.csv` — test-set confusion-matrix counts and threshold trade-offs.
- `credit_risk_feature_importance.csv` — leading XGBoost feature-importance values.

### Recommended one-page report layout

**Page title:** Credit Risk Model Performance & Decision Strategy

**KPI cards**
1. **Test-set accuracy:** 71.5%
2. **ROC-AUC:** 0.653
3. **Bad-credit recall (0.50):** 21.7%
4. **Bad-credit recall (0.19):** 96.7%
5. **False positives at 0.19:** 110
6. **Illustrative weighted error cost:** 120

Label all threshold-dependent KPIs clearly so viewers do not confuse the 0.50 and 0.19 scenarios.

**Visuals**
- Clustered bar chart: model comparison for ROC-AUC, precision, recall and F1.
- Confusion-matrix style visual: true negatives, false positives, false negatives and true positives by threshold.
- Threshold trade-off chart: recall and precision across scenarios.
- Horizontal bar chart: top feature-importance values.
- Slicer: model or threshold scenario, where relevant.

### Suggested DAX measures

After importing the threshold CSV, use measures like these. Table names should match the names created in Power BI. The measures calculate metrics for the selected threshold scenario:

```DAX
True Positives = SUM(credit_risk_threshold_comparison[True_Positives])
False Positives = SUM(credit_risk_threshold_comparison[False_Positives])
True Negatives = SUM(credit_risk_threshold_comparison[True_Negatives])
False Negatives = SUM(credit_risk_threshold_comparison[False_Negatives])

Bad-Credit Recall =
DIVIDE(
    [True Positives],
    [True Positives] + [False Negatives]
)

Bad-Credit Precision =
DIVIDE(
    [True Positives],
    [True Positives] + [False Positives]
)

Accuracy =
DIVIDE(
    [True Positives] + [True Negatives],
    [True Positives] + [True Negatives] +
    [False Positives] + [False Negatives]
)

Illustrative Weighted Error Cost =
5 * [False Negatives] + [False Positives]
```

Use `Scenario` as a slicer or axis to compare the default and cost-sensitive thresholds. Avoid summing the confusion-matrix rows across both scenarios, because they are alternative predictions on the same test set.

### Import the data into Power BI Desktop

1. Open Power BI Desktop.
2. Select **Get data → Text/CSV**.
3. Import the three CSV files.
4. Create the DAX measures above.
5. Add card visuals for the KPIs and the charts listed above.
6. Format rates as percentages and add a note that the cost weights are illustrative.
7. Check that selecting a threshold filters to one scenario at a time.

## Limitations and next steps

- The dataset is small and may not represent a modern credit-card portfolio.
- The model is a classification exercise; it does not directly estimate calibrated probability of default, loss given default or exposure at default.
- The threshold cost weights are illustrative, not derived from actual financial outcomes.
- Threshold selection on the test set can make reported performance optimistic; validate the decision rule on independent data.
- Review model calibration, stability, fairness, explainability and potential proxy variables before operational use.
- A real business deployment would require monitoring for drift, policy governance, auditability and human review where appropriate.

## Repository structure

```text
credit-risk-project/
├── Credit_Risk_Modelling_SVM.ipynb
├── README.md
├── credit_risk_model_comparison.csv
├── credit_risk_threshold_comparison.csv
└── credit_risk_feature_importance.csv
```

## Tools

Python · Pandas · Scikit-learn · XGBoost · SHAP · Matplotlib · Power BI · DAX

## Summary

This project demonstrates a full credit-risk modelling workflow, from data preparation and model comparison to explainability and cost-sensitive threshold analysis. Its central business lesson is that **the best decision rule depends on the relative cost of missed risky borrowers and incorrectly flagged creditworthy borrowers**.
