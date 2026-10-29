# Free-to-paid-conversion-Revenue-forecasting
Free-to-Paid Conversion & Revenue Forecasting (SaaS)

End-to-end analytics project for a simulated product-led SaaS company (Jan 2023 - Jun 2026): SQL funnel and
MRR metrics, a free-to-paid conversion model, rolling-origin revenue forecasting with scenarios, and a
Power BI dashboard.

> **Data note:**  Conversion drivers, prices, churn and
> seasonality are explicit assumptions, so the results show *method and pipeline*, not findings about a real
> company. 

## Business questions
1. Where does the signup -> paid funnel leak, and which early behaviours predict upgrading?
2. Can we rank free users by upgrade likelihood so growth teams can focus on the right people?
3. What will MRR be over the next 6 months, and how reliable is that forecast?
4. How much does MRR move if conversion improves or churn changes?

## Pipeline
```
simulate_saas.py -> data/{users, usage_14d, subscriptions, invoices}.csv   (102,866 users, 75,829 invoices)
sql/01..06 via run_sql_pipeline.py (SQLite) -> funnel, MRR, MRR bridge, modelling table, cohorts, driver lift
train_conversion.py -> Logistic Regression vs gradient boosting, drivers, deciles, scored users
forecast_mrr.py     -> backtest of 4 models, 6-month forecast, scenarios, retention KPIs
outputs/tables/*.csv -> Power BI (powerbi/POWERBI_GUIDE.md)
```

## How to run
```bash
pip install -r requirements.txt
python src/simulate_saas.py
python src/run_sql_pipeline.py
python src/train_conversion.py     # uses XGBoost if installed, otherwise scikit-learn fallback
python src/forecast_mrr.py         # adds SARIMA / Prophet automatically if installed
```

## Definitions (leakage-safe)
- **Prediction time:** day 14 after signup. **Population:** users still on the free plan at day 14.
- **Target:** upgrades to paid on days 15-44. Only users whose full 45-day window is observed are included.
- **Features:** first-14-day usage (sessions, active days, projects, invites, features, integrations, usage-limit hit,
  tickets, onboarding) plus channel, company size and region. Nothing from after day 14.
- **Split:** by signup month: train <= 2025-06, validation 2025-07..12, test 2026-01..05.
- **MRR bridge:** MRR(t) = MRR(t-1) + new + expansion - churned. The SQL bridge reconciles to the dollar (error $0.00).

## Results (from the run included here)
**Business snapshot (Jun 2026):** MRR $186,300 (+57.8% year over year), 3,760 paying customers, ARPA $49.5.
About 4.4% of signups upgrade within 30 days; roughly half complete onboarding.

**Conversion drivers (descriptive):** users who hit a usage limit in the first 14 days converted at 7.2% versus
2.0% for those who did not; users who invited a collaborator converted at 6.1% versus 2.4%; users active 4+
days converted at 4.7% versus 1.9%.

**Conversion model (test = 2026 signups, n = 15,037, 518 conversions, base rate 3.4%):**

| Model | ROC-AUC | PR-AUC | Top-decile capture | Top-decile lift |
|---|---|---|---|---|
| Logistic Regression | 0.707 | 0.090 | 31.5% | 3.1x |
| Gradient boosting* | 0.706 | 0.089 | 31.3% | 3.1x |

\*XGBoost was **not installed** in the build environment, so this row is scikit-learn
`HistGradientBoostingClassifier`. Install xgboost, re-run `train_conversion.py`, and update this row before
quoting XGBoost numbers.

The top 10% of scored users produce about 31% of all conversions, and predicted probabilities track actual
rates across deciles (decile 1: 11.6% predicted vs 10.8% actual). Usage-limit hits are by far the strongest
driver (odds ratio 2.7), followed by collaborator invites and session volume. A ROC-AUC near 0.71 is modest
but realistic for a 3% base-rate conversion problem; it is useful for ranking, not for confident individual
predictions. Boosting does not beat logistic regression here because the simulated relationships are simple.

**MRR forecasting (rolling-origin backtest, 13 origins x 6 horizons):**

| Model | Mean MAPE | MAPE at 6 months | MAE |
|---|---|---|---|
| Damped Holt (selected) | 2.3% | 3.8% | $3,378 |
| Bottom-up driver model | 2.6% | 4.5% | $3,772 |
| Drift | 3.6% | 5.9% | $5,342 |
| Naive | 13.7% | 22.2% | $19,424 |

**Forecast:** MRR reaches about **$215K by Dec 2026** (approximate 80% interval $204K to $227K). Intervals come
from backtest errors at each horizon with only 13 origins, so treat them as indicative.

**Scenarios (bottom-up driver model; its baseline is $209K for Dec 2026, about 3% below the damped-Holt
forecast, which shows the model uncertainty):**

| Scenario | Dec 2026 MRR vs baseline |
|---|---|
| Conversion +10% | +$4.1K (+2.0%) |
| Churn -10% | +$3.2K (+1.5%) |
| Both | +$7.4K (+3.5%) |
| Churn +15% (downside) | -$4.7K (-2.3%) |

**Retention insight:** monthly gross MRR churn is 2.8% and expansion 1.1%, so net revenue retention is about 98.3%
per month (roughly 81% annualised). Growth is acquisition-driven, not expansion-driven. Pooled cohort retention at
12 months is 58% of customers and 78% of revenue.

## 

## 



## Layout
```
src/       simulate_saas.py, run_sql_pipeline.py, train_conversion.py, forecast_mrr.py
sql/       01_conversion_funnel ... 06_driver_lift
data/      generated CSVs
outputs/   tables/ (Power BI inputs, model and forecast results), figures/ (PNG)
powerbi/   POWERBI_GUIDE.md
```
