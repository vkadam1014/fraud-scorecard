# Fraud Loss Scorecard

An SQL fraud scorecard and executive dashboard built in **Databricks** using 590K e-commerce transactions. It measures how much fraud costs, where it concentrates, how it trends over time, and whether simple decline rules can stop it without blocking good customers.

![Fraud Scorecard Dashboard](images/dashboard.png)

**Tools:** Databricks SQL · Delta Lake · Unity Catalog · AI/BI Dashboards · GitHub

---

## Business questions

1. How big is the fraud problem in transactions and in dollars?
2. Which products and card types account for the most fraud dollars?
3. How does fraud change over time?
4. Can simple decline rules stop fraud without blocking too many good customers?

---

## Key findings

| Metric | Value |
|---|---|
| Transactions analyzed | 590,540 |
| Total transaction volume | $79.7M |
| Fraud rate (by count) | 3.5% |
| Fraud share of dollars | 3.87% (~$3.1M) |
| Average fraud ticket | $149.24 |

**1. Fraud is concentrated in one product line.**
Product **W** accounts for **~67% of fraud dollars** among segments with 1,000+ transactions, far more than products C, R, and H combined.

**2. Fraud rose after the holidays, not during them.**
The weekly fraud rate was lowest in December (**2.3–2.9%**), even during the peak holiday week of Dec 18 (39K transactions, the busiest week in the data). From January through March it rose to **3.3–4.9%**.

**3. Simple rules don't separate fraud from good transactions well.**
A useful rule should catch a much larger share of fraud than the share of transactions it flags. Lift is the ratio of the two.

| Rule | % of txns flagged | % of fraud txns caught | % of fraud $ caught | Lift |
|---|---|---|---|---|
| Amount > $500 | 3.87 | 4.88 | 30.42 | 1.26 |
| Amount > $200 | 15.90 | 20.05 | 62.96 | 1.26 |
| Amount > $100 | 37.60 | 38.92 | 81.07 | 1.04 |
| Velocity ≥ 2 txns/hr | 23.70 | 28.01 | 26.07 | 1.18 |
| Velocity ≥ 3 txns/hr | 15.00 | 16.92 | 15.17 | 1.13 |
| Velocity ≥ 5 txns/hr | 7.06 | 6.86 | 5.79 | 0.97 |

- **Velocity rules perform close to random** (lift ≈ 1.0). `card1` groups many cards together, so it's too broad to measure true per-card velocity.
- **Amount rules capture many fraud dollars, but much of that is automatic.** Large transactions carry large dollar amounts whether they're fraud or not. The > $500 rule catches 30% of fraud dollars but only about 5% of fraud transactions.
- **Recommendation:** use **Amount > $500** as a **manual-review** trigger, not an auto-decline. It sends about 4% of traffic to review and covers about 30% of fraud exposure. Stopping fraud more precisely needs a scored model that combines many signals.

---

## Data quality checks

Automated checks run as a SQL view and appear on the dashboard:

| Check | Failed rows | Status |
|---|---|---|
| Duplicate transaction IDs | 0 | PASS |
| Missing or non-positive amounts | 0 | PASS |
| Invalid fraud labels | 0 | PASS |
| Row count matches source | 0 | PASS |
| Missing purchaser email domain | 94,456 (16.0%) | WARN |

---

## Pipeline

```
Kaggle CSV
   │
   ▼
Unity Catalog volume  (raw file)
   │
   ▼
train_transactions              (Delta table, all columns)
   │
   ▼
train_transactions_analysis     (selected columns + date, hour, velocity)
   │
   ├── ..._data_checks          data-quality PASS/WARN/FAIL
   ├── ..._headline_numbers     KPI counters
   ├── ..._weekly_trends        weekly fraud + 4-week rolling average
   ├── ..._fraud_concentration  segment share of fraud $ + lift
   └── ..._rule_tests           decline-rule tradeoff results
   │
   ▼
Databricks AI/BI dashboard
```

**SQL techniques:** CTEs, window functions (`RANGE` time windows, rolling averages, `SUM() OVER ()` for shares of total), `CROSS JOIN` threshold grids with `explode(sequence())` and `VALUES`, `COUNT_IF`, `NULLIF` protection against division by zero, `UNION ALL` to combine data-quality checks.

---

## Repository structure

```
fraud-scorecard/
├── README.md
├── notebooks/
│   └── fraud_scorecard.sql          # all table + view definitions
├── dashboard/
│   └── fraud_scorecard.lvdash.json  # importable Databricks dashboard
└── images/
    └── dashboard.png
```

---

## How to reproduce

1. Download `train_transaction.csv` from the [IEEE-CIS Fraud Detection](https://www.kaggle.com/c/ieee-fraud-detection) competition on Kaggle. The data is not included here because of its size and the competition's terms.
2. In Databricks, create a volume at `workspace.default.train_transactions` and upload the CSV.
3. Import `notebooks/fraud_scorecard.sql` and run all cells.
4. Import `dashboard/fraud_scorecard.lvdash.json` from the Dashboards page.

---

## Assumptions and limitations

- **Dates:** `TransactionDT` counts seconds from an undisclosed start point. Following common Kaggle practice, it is treated as starting **Dec 1, 2017**.
- **Hour of day:** for the same reason, the hour is relative and may not match real local time.
- **Card ID:** `card1` is an anonymized, broad card grouping, not a unique card, which weakens velocity features.
- **Partial weeks:** the first and last weeks are partial, and the first three rolling-average values cover fewer than four weeks.
- **Segment shares** are calculated among segments with 1,000+ transactions.
- **Rule results** assume each rule declines every transaction it flags. Real controls (review, step-up verification) would change the customer impact.

---

## Next steps

- Build a **fraud score model** with time-aware features and sweep decline thresholds to find the best balance of fraud loss and customer friction (in progress).
- Add a **three-band strategy** (approve / review / decline) with a manual-review cost.
- Compare each rule's share of fraud dollars caught with its share of **all** dollars flagged, for a fairer view of amount rules.
