# UK Energy Drinks — Weekly Demand Prediction

Forecasting UK energy-drink demand with machine learning, and modelling the business impact of England's incoming ban on selling high-caffeine energy drinks to under-16s (April 2027).

> **Academic context.** This repository is the code companion to a Master's thesis — *"Predictive Modelling of Consumer Demand for Energy Drinks Using Machine Learning Methods"* — submitted to Neoversity in fulfilment of the degree of Master of Science in Computer Science (2026). The notebook in this repo is the full analysis referenced by that thesis: every result below is drawn directly from it.

## What this project does

The project has two parts, run in a single notebook:

1. **Part 1 — Demand forecasting.** Five regression models (Linear Regression, Decision Tree, Random Forest, Gradient Boosting, XGBoost) are trained on 313 weeks (2020–2025) of UK energy-drink market data and compared on a held-out final year using a strict time-based split, so no model ever sees the future during training. Linear Regression is then pushed further with polynomial terms and extra Fourier seasonal harmonics, and every tree model is retested with walk-forward retraining.
2. **Part 2 — Business scenario analysis.** The best-performing model is used to produce a 2026–2030 baseline forecast, onto which the estimated impact of the April 2027 under-16 sales ban is layered as an explicit, transparent scenario adjustment — not something the model is asked to "discover" on its own.

## Data

The modelling table (`Weekly_Sales`) holds 313 weekly observations (2020–2025) of estimated UK energy-drink market value in GBP, distributed from public annual market-value figures — it is a **modelled allocation, not observed point-of-sale data** (see the `Overview` sheet in the source workbook). Three columns are deliberately excluded from every model (`Annual Benchmark GBP`, `Model Weight`, `Weekly Share of Annual`) because they are mathematically derived from the target itself and would leak the answer.

The source Excel workbook is not included in this repository. To reproduce the notebook end-to-end, supply your own copy at `../data/UK_energy_drinks_weekly_2020_2025.xlsx` relative to the notebook.

## Methodology

- **Feature engineering:** calendar features (cyclical week sin/cos, month, quarter, year), plus autoregressive features (`lag_1`, `lag_2`, `lag_4`, `lag_52`, 4/8/52-week rolling mean and std).
- **Validation:** a chronological train/test split (final 52 weeks held out) combined with `TimeSeriesSplit` cross-validation for hyperparameter search — never a random shuffle.
- **Metrics:** MAE, RMSE, MAPE, and R², evaluated primarily on the held-out final year.
- **Three follow-up experiments**, all included in the notebook:
  - **Polynomial regression on the trend** (Section 9.1) — every variant made Linear Regression worse, sometimes catastrophically (R² in the thousands-negative for the unregularized full expansion), because polynomial curves extrapolate unpredictably past the training range while a straight line just keeps its slope.
  - **Extra Fourier seasonal harmonics** (Section 9.2) — letting Linear Regression's seasonal term be more than a single sine wave improved it further, since harmonic terms stay bounded and add no extrapolation risk. Tree models barely moved, since they can already carve up the seasonal cycle without help.
  - **Predicting the year-over-year ratio instead of the level** (Section 10) — this made every tree model dramatically worse, and the notebook documents exactly why (the trees used `lag_52`'s magnitude as an implicit "which year is this" lookup rather than a genuine ratio signal).
  - **Walk-forward retraining** (Section 11) — refitting each tree model every 4 weeks through the test year, which recovered most of their lost accuracy by turning a blind year-ahead forecast into a short-horizon interpolation problem.

## Key results

Final model comparison, ranked by MAE (from the notebook's own conclusion, Section 13):

| Model | MAE (GBP) | R² |
|---|---:|---:|
| **Linear Regression (3 harmonics)** | **409k** | **0.80** |
| **Linear Regression (2 harmonics)** | **414k** | **0.81** |
| Linear Regression (1 harmonic, baseline) | 437k | 0.78 |
| Gradient Boosting (walk-forward retrain) | 630k | 0.58 |
| Random Forest (walk-forward retrain) | 683k | 0.55 |
| XGBoost (walk-forward retrain) | 719k | 0.57 |
| Decision Tree (walk-forward retrain) | 974k | 0.31 |
| XGBoost (static, 2024 cutoff) | 1.30M | 0.07 |
| Gradient Boosting (static) | 1.36M | 0.00 |
| Random Forest (static) | 1.37M | −0.01 |
| Decision Tree (static) | 1.45M | −0.14 |
| Random Forest / XGBoost / Decision Tree / Gradient Boosting (ratio target) | 4.75M–4.81M | −6.78 to −6.99 |

(Polynomial-regression variants are omitted from this table — all four lost to the plain Linear Regression baseline; see Section 9.1 for their numbers.)

**Headline finding:** Linear Regression wins because this series has a single dominant multi-year growth trend, and tree-based models structurally cannot extrapolate past the maximum value they saw in training — they plateau while the real series keeps growing. Adding Fourier harmonics lets Linear Regression represent the seasonal cycle more precisely without introducing any extrapolation risk, taking it from MAE £437k/R²0.78 to **£409k/R²0.80** — the best result in the notebook. Walk-forward retraining closes most, but not all, of the extrapolation gap for the tree models by shortening their effective forecasting horizon to a few weeks. **Recommendation: use the harmonics-enhanced Linear Regression for long-horizon strategic forecasting, and walk-forward-retrained tree ensembles for short-horizon (4–8 week) operational forecasting** — there is no single model that is best for both.

## Business scenario: the April 2027 under-16 ban

England is banning the sale of drinks with more than 150mg of caffeine per litre to under-16s from April 2027. Since no model can learn a future law from historical sales, the ban's impact is modelled as a scenario layered on top of the Linear Regression (harmonics) baseline, using the government's own estimated 59.3% demand-displacement factor together with three illustrative assumptions about the under-16 share of the market:

| Scenario | Youth share (assumed) | Total-market impact | Cumulative modelled impact |
|---|---:|---:|---:|
| Low | 3% | 1.78% | £187.5M |
| Central | 5% | 2.96% | £312.6M |
| High | 8% | 4.74% | £500.1M |

The under-16 revenue share is not present in the dataset — it is the single biggest source of uncertainty in these figures, which is exactly why the analysis is run as three scenarios rather than one point forecast.

## Conclusions

- No single model wins for every use case: model choice should follow the forecasting horizon, not the other way around.
- Extrapolation ability — not raw fit to training data — is what actually separated the models here. Harmonics helped Linear Regression because they stay bounded; polynomial terms hurt it because they don't.
- Regulatory shocks should be modelled as explicit scenarios layered on a forecast, never learned by the forecast itself.
- The most valuable next step for this analysis is not more modelling, but real age/region/brand-segmented sales data, which would replace the Low/Central/High assumption with a measured number.
- Every number in this project should be read as a demonstration of a forecasting *method* on realistic-shaped data, not as a claim about actual UK energy-drink demand — the underlying weekly series is itself a modelled allocation, not observed sales.

## Repository contents

| File | Description |
|---|---|
| `demand_prediction.ipynb` | The full analysis: data loading, EDA, feature engineering, the five-model comparison, the polynomial/harmonics and walk-forward experiments (Part 1), and the ban-scenario business analysis (Part 2). |
| `README.md` | This file. |

## Author

Olga Goldberg — Master of Science in Computer Science, Neoversity, 2026.
