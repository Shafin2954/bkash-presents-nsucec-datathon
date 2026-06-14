# FictiPay Customer Churn Prediction — NSUCEC Datathon (bKash)

End-to-end churn-prediction pipeline on a synthetic mobile-wallet dataset at 73M-transaction scale.
**Churn** = a Customer initiates **no transaction in April 2024**; all features use **only** the Jan–Mar 2024 observation window. Metric: **AUC-ROC**.

**Team Fixeth** · Jawat Al Sovon (team lead) · Shafin Ahmed Soron

## Results

| Metric | Value |
|---|---|
| 5-fold OOF AUC-ROC (final blend) | **0.9852** |
| Average Precision | 0.926 |
| Precision @ top 10% | 0.918 |
| Recall @ top 10% | 0.724 |

Four independent model families (LightGBM, XGBoost, CatBoost, a per-event Transformer) converge at ~0.985 OOF. Train and test are confirmed identically distributed (adversarial AUC 0.52), so OOF tracks the leaderboard.

## Repository contents

| File | What it is |
|---|---|
| [`notebook.ipynb`](notebook.ipynb) | Clean, runnable end-to-end pipeline (Polars streaming → features → models → submission) |
| [`report.pdf`](report.pdf) | Full model report: data handling, FE, Pareto/imbalance treatment, model comparison, tuning, SHAP, leakage audit, ceiling analysis |
| [`presentation.pdf`](presentation.pdf) | 5-slide summary deck |
| [`features.md`](features.md) | ~70 engineered features across 8 families, each with a one-line rationale |
| [`explainability/`](explainability/) | SHAP summary, ranking, dependence plots, tuning curve + [`explainability_insights.md`](explainability/explainability_insights.md) |
| `predictions.csv` | Final test-set churn probabilities (submission format) |
| `src/` | Pipeline scripts for full reproducibility *(optional — add if you include them)* |

## Approach (mapped to challenge stages)

1. **Large-scale data handling** — Polars lazy/streaming over 73M transactions + 77M daily balances; columnar projection, per-month partitions, feature caching. Runs out-of-core in 16 GB RAM (no cluster). Spark/Dask is the right call at 200M+ rows across machines; for this single-node task Polars streaming gives the same guarantee at far lower cost.
2. **Feature engineering** — RFM + **Clumpiness** (RFMC) + balance dynamics + P2P network + type-mix + KYC. Multi-window (7/30/90d) counts, cadence-normalized recency, weekly/daily sequence shape, balance-change frequency. See `features.md`.
3. **Feature quality** — `log1p` on heavy-tailed amounts, percentile features, explicit zero-inflation ratios, sentinel + flag for sparse users.
4. **Class imbalance** (12.7% churn) — class-weighted boosting; SMOTE/undersampling tested and rejected (SMOTE hurt top-K precision).
5. **Models** — LightGBM (Optuna-tuned, 3-seed × 5-fold) + XGBoost + CatBoost + per-event Transformer, rank-blended for low variance.
6. **Tuning** — Optuna TPE Bayesian search, 30 trials (`explainability/tuning_curve.png`).
7. **Explainability** — SHAP ranking + dependence; full **leakage audit** (ID order, TrxID, timestamps, April-window bleed, P2P neighbours — all clean).
8. **Business recommendation** — flag the top-10% risk decile (0.91 precision, ~72% of churners captured); targeted interventions per risk driver.

## Key finding

Among customers with a recent transaction but a **frozen daily balance**, churn risk stays elevated — silent disengagement that pure transaction-recency misses. `bal_chg_last14` is the sharpest separator in the hard segment (within-segment AUC ≈ 0.74).

## Performance ceiling

An OOF error autopsy shows residual errors concentrate on customers active through 31 March who still churn in April. With 81% of accounts at recency ≤ 5 days and a smooth, non-deterministic churn-rate surface among active users, these cases are near-independent of observed behavior — the behavioral information ceiling for the provided tables.

## Reproduce

```bash
pip install polars lightgbm xgboost catboost scikit-learn shap optuna pandas pyarrow scipy torch
# place the competition data under /kaggle/input/... or edit BASE in notebook.ipynb
jupyter nbconvert --to notebook --execute notebook.ipynb
```
