# Feature Documentation — FictiPay Churn Prediction

**Team submission · NSUCEC Datathon (bKash)**
**Team Fixeth** · Jawat Al Sovon (team lead) · Shafin Ahmed Soron
Churn definition: a Customer account is `CHURN = 1` if it initiates **no transaction in 2024-04-01 → 2024-04-30**. All features are built **only** from the observation window **2024-01-01 → 2024-03-31** (no leakage from the prediction window, which is not provided).

Reference date for all recency/age calculations: **2024-04-01** (the first day of the prediction window).

The model uses ~70 engineered features. They are organized below by family, each with a one-line rationale. The framework is **RFM + Clumpiness + Survival/Hazard + Sequence shape**, which the literature identifies as the strongest behavioral basis for non-contractual churn.

---

## 1. Recency (R) — "how long since they were active"

The single strongest churn axis: churn is *defined* by future inactivity, so recent activity is the best proxy for continued activity.

| Feature | Rationale |
|---|---|
| `recency_d` | Days since last sent transaction (day granularity) — primary churn signal. |
| `rec_d` | Days since last transaction at fractional-day resolution — sharper than `recency_d`. |
| `rec_P2P`, `rec_MerchantPay`, `rec_BillPay`, `rec_CashIn`, `rec_CashOut` | Days since last transaction *of each type* — captures channel-specific abandonment (e.g. stopped paying bills but still cashes out). |
| `rcv_recency_d`, `rcv_all_rec` | Days since the customer last *received* funds — inbound engagement decays before churn. |
| `overdue_d` | Recency minus the customer's median inter-transaction gap — how "overdue" the next transaction is relative to their own rhythm. |
| `rec_over_gap`, `rec_over_gapmax`, `rec_over_gap90`, `rec_z` | Recency normalized by personal cadence (mean / max / 90th-pct gap, and z-score) — silence means more for a daily user than a monthly one. |
| `rec_over_medgap` | Recency over personal median gap — robust cadence-normalized lateness. |

## 2. Frequency (F) — "how often they transact"

| Feature | Rationale |
|---|---|
| `w7_cnt`, `w30_cnt`, `w90_cnt` | Transaction counts over trailing 7/30/90-day windows — multi-scale activity level. |
| `freq_per_day` | Transactions per active day across the window — intensity. |
| `active_days` | Number of distinct days with activity — engagement breadth. |
| `weeks_active` | Number of distinct active weeks (of 13) — breadth at weekly scale. |
| `ratio_w7_w30`, `ratio_w30_w90` | Short- vs long-window count ratios — acceleration / deceleration of activity. |
| `m1_cnt`, `m2_cnt`, `m3_cnt` | Monthly transaction counts (Jan/Feb/Mar) — raw monthly trajectory. |
| `trend_m3_m1` | March count over January count — multiplicative growth/decay trend. |
| `late_share` | Share of all transactions falling in the last 4 weeks — recency-weighted intensity. |
| `last_wk` | Index of the last active week — when the trail goes cold. |
| `wk0 … wk12` | Per-week transaction counts (13 features) — full activity shape for the model to learn decay patterns. |
| `d63 … d90` | Per-day transaction counts for the final 28 days (28 features) — fine-grained end-of-window shape; the most predictive sequence block. |

## 3. Monetary (M) — "how much they move"

| Feature | Rationale |
|---|---|
| `w7_amt_sum`, `w30_amt_sum`, `w90_amt_sum` | Total amount per window — monetary engagement. |
| `w7_amt_mean`, `w30_amt_mean`, `w90_amt_mean` | Mean transaction size per window. |
| `w7_amt_std`, `w30_amt_std`, `w90_amt_std` | Amount volatility per window — erratic spend can precede churn. |
| `amt_max` | Largest single transaction — captures whales / one-off behavior. |
| `lamt_mean`, `lamt_std`, `lamt_skew` | Mean / std / skew of log-amounts — distribution shape robust to the heavy right tail. |
| `amt_p10`, `amt_p50`, `amt_p90` | Amount percentiles — typical and tail spend. |
| `amt_small_frac` | Fraction of transactions under 100 — micro-transaction reliance. |
| `amt_uniq_ratio` | Unique amounts / count — repetitive vs varied spending. |

## 4. Clumpiness (C) — "bursty vs evenly spread"

Formal RFMC extension. Two users with identical R/F/M can differ sharply if one is steady and the other transacts in bursts then goes quiet.

| Feature | Rationale |
|---|---|
| `clumpiness` | Entropy-based burstiness of inter-event gaps over the window (bursty → high) — distinguishes steady users from spike-then-silence users. |
| `gap_cv` | Coefficient of variation of inter-transaction gaps — irregularity of cadence. |
| `gap_mean`, `gap_std`, `gap_max`, `gap_p90`, `gap_med` | Inter-transaction gap statistics — the customer's baseline rhythm. |
| `activity_span`, `span_d` | Days between first and last transaction in the window — engaged lifespan. |

## 5. Balance dynamics — "is the wallet winding down"

Daily end-of-day balance series. Captures disengagement that transaction counts miss: a customer can have a recent transaction yet a stalling balance.

| Feature | Rationale |
|---|---|
| `bal_mean`, `bal_std`, `bal_max`, `bal_last` | Level and variability of the balance, and the final-day balance. |
| `bal_zero_frac` | Fraction of days at zero balance — empty-wallet users churn more. |
| `bal_trend`, `bal_m1`, `bal_m3` | March-vs-January mean balance ratio and monthly means — balance trajectory. |
| `bal_days_frozen` | Days since the balance last changed — a frozen balance signals a dormant account even with a recent last transaction. |
| `bal_chg_freq`, `bal_chg_last14`, `bal_chg_last7`, `bal_chg_last3` | Fraction of days the balance moved, over the full window and the last 14/7/3 days — the sharpest "winding-down" detector found (hard-segment AUC 0.74). |
| `bal_last7_mean`, `bal_last7_std` | Mean / volatility of balance in the final week. |
| `bal_mom7`, `bal_mom30`, `bal_drop_late` | Late-window balance momentum and drawdown ratios — recent depletion. |

## 6. Counterparty & network

| Feature | Rationale |
|---|---|
| `n_dst` | Distinct counterparties transacted with — relationship breadth. |
| `n_merch` | Distinct merchants/billers paid (non-customer DST) — service-usage breadth; strong in the hard segment (AUC 0.78). |
| `dst_collapse` | Unique counterparties in March over January — shrinking network = disengagement. |
| `rcv_cnt`, `rcv_amt`, `rcv_all_cnt`, `rcv_months` | Inbound P2P/all count, amount, and months received — being paid keeps wallets alive. |
| `nbr_churn`, `nbr_churn_w` | Smoothed churn rate of P2P neighbours (plain & edge-weighted), **encoded out-of-fold** to prevent leakage — tests social/regional contagion. |
| `nbr_n_lab`, `n_nbr`, `w_sum` | Count of labelled neighbours and graph degree (count & weight). |

## 7. Type mix

| Feature | Rationale |
|---|---|
| `r_p2p`, `r_mpay`, `r_bill`, `r_cashin`, `r_cashout` | Share of each transaction type — behavioral profile (a pure bill-payer churns differently from a P2P-heavy user). |

## 8. Account / demographic (KYC)

| Feature | Rationale |
|---|---|
| `tenure_d` | Days since account open — older accounts are stickier. |
| `months_active` | Count of Jan/Feb/Mar with ≥1 transaction — coarse persistence flag. |
| `bill_months` | Count of months with a bill payment — recurring-obligation stickiness. |
| `dom_std`, `dom_med` | Std / median of day-of-month of transactions — monthly-cycle regularity (salary/bill cadence). |
| `GENDER`, `REGION` | Demographic categoricals — native-categorical in LightGBM/CatBoost. |

## 9. Sparsity / activity flags

| Feature | Rationale |
|---|---|
| `no_trx` | Indicator: customer made no transaction in the window — separates the trivially-churning cold accounts. |

---

### Notes on feature quality (Stage 3)

- **Skew handling:** all monetary features modeled as `log1p` (amounts span 10 → 200,000, a classic Pareto tail). Tree models are split-invariant, but log scaling stabilizes the engineered ratios and the neural models.
- **Sparsity:** recency-type features for inactive users are filled with a sentinel beyond the window (91–120 days) plus the explicit `no_trx` flag, rather than zero, so "never happened" is distinct from "happened long ago."
- **Zero-inflation:** balance-zero and small-amount fractions are kept as explicit ratio features rather than discarded.
- **Leakage control:** graph neighbour-churn features are encoded with per-fold out-of-fold target statistics; all temporal features respect the 2024-03-31 cutoff.
