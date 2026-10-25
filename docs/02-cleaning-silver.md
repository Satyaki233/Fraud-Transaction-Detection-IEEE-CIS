# Chapter 2: Data Cleaning (Silver)

**Goal:** two clean, joined tables with data-quality rules: `silver.train` (with the label) and `silver.test` (without it).

## Concepts

- **Materialized view (MV):** a table defined by a query. The pipeline stores its result and refreshes it on each run. Good for joins and aggregations over the whole dataset.
- **Expectations:** data-quality rules on a table, written `EXPECT (condition)`. When a row breaks the rule you can keep it and count it (the default), `DROP ROW`, or `FAIL UPDATE`. The pipeline UI shows how many rows passed and failed.
- **Left join:** keeps every transaction, even those with no identity row. About 76% of transactions have none, so an inner join would throw most of the data away.
- **Missing values are information:** don't fill in blanks. The model handles them, and "no device info" can itself be a fraud signal.
- **Feature selection:** 434 columns is too many to start with. Pick about 40, then add more later and measure the effect.
- **Data leakage:** the model seeing information it wouldn't have at prediction time, which makes its scores look better than they really are. Every feature must use only the *past*.

## Build
Add these tables to the same `01_pipeline` notebook, below the Bronze tables.

### 1. `silver.train`
Pattern:
```sql
CREATE OR REFRESH MATERIALIZED VIEW silver.train (
  CONSTRAINT valid_id     EXPECT (TransactionID IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT valid_amount EXPECT (TransactionAmt > 0)        ON VIOLATION DROP ROW,
  CONSTRAINT has_card     EXPECT (card1 IS NOT NULL)          -- warn only
)
COMMENT "Clean, joined training data"
AS SELECT
  t.TransactionID,
  -- … your columns …
FROM bronze.train_transaction t
LEFT JOIN bronze.train_identity i ON t.TransactionID = i.TransactionID;
```

Starter column set:

| Group | Columns | Why |
|---|---|---|
| Keys and time | `TransactionID`, `TransactionDT`, `event_ts` | Identify rows, split by time |
| Label | `isFraud` | Train only |
| Purchase | `TransactionAmt`, `ProductCD` | |
| Card | `card1`–`card6` | `card1` roughly identifies a card |
| Address | `addr1`, `addr2`, `dist1` | |
| Email | `P_emaildomain`, `R_emaildomain` | Buyer and recipient |
| Counts | `C1`–`C14` | Meaning hidden by Vesta |
| Time gaps | `D1`, `D4`, `D10`, `D15` | |
| Match flags | `M4`, `M6` | |
| Device | `DeviceType` | From the identity table |

Turn `TransactionDT` (seconds since an unknown start) into a real timestamp:
```sql
timestampadd(SECOND, TransactionDT, TIMESTAMP '2017-12-01') AS event_ts
```

### 2. `silver.test`
Use the same columns from the test tables, without `isFraud`. Two differences:
- **Expectations only warn here; they don't drop rows.** In production you can't skip a transaction just because it looks odd, because every transaction must get a score. Kaggle also rejects a submission with missing rows.
- If you add any `id_xx` column, the test file names it `id-01`. Select it with backticks and rename it: `` i.`id-01` AS id_01 ``.

### 3. Optional: your first fraud feature
How many transactions did this card make in the previous 24 hours?
```sql
COUNT(*) OVER (
  PARTITION BY card1
  ORDER BY TransactionDT
  RANGE BETWEEN 86400 PRECEDING AND 1 PRECEDING
) AS card_txn_24h
```
It only looks at *earlier* rows, which is why it doesn't leak. A sudden burst of activity on one card is the classic fraud signal.

### 4. Run and inspect
- The graph should show Bronze → Silver.
- Click `silver.train` → **Data quality**. How many rows did each rule drop?
- Quick checks in the SQL editor:
  ```sql
  SELECT COUNT(*), AVG(isFraud) FROM workspace.silver.train;  -- fraud rate ≈ 0.035
  SELECT COUNT(*) FROM workspace.silver.test;                 -- must be 506,691
  ```

## Done when
- [x] `silver.train` and `silver.test` exist, and the graph is green
- [x] Fraud rate ≈ 3.5%
- [ ] `silver.test` has every test transaction (506,691)
- [ ] You know how many rows each expectation dropped

## Check yourself
- Why a LEFT JOIN and not an INNER JOIN? What would an INNER JOIN do to the row count?
- Why drop bad rows in train but only warn in test?
- Why is `card_txn_24h` safe from leakage? What change would make it leak?

## Pitfalls
- Writing `JOIN` instead of `LEFT JOIN` silently loses about 76% of the rows.
- `id-01` without backticks is read as `id - 01`, which is subtraction.
- An "ambiguous column" error after the join means you need to prefix columns with `t.` or `i.`.

Next: [Chapter 3, Model creation](03-model-training.md)
