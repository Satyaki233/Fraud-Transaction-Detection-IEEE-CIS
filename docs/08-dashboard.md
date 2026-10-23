# Chapter 8: Dashboard

**Goal:** an AI/BI dashboard that tells the project's story in a presentation.

## Concepts

- **AI/BI dashboard:** *datasets* (SQL queries) plus *widgets* (charts), arranged on pages. It runs on a SQL warehouse.
- **SQL warehouse:** the compute that runs SQL queries. Free Edition gives you one, size 2X-Small.
- **Datasets:** write each query once and reuse it in several widgets.
- **Filters:** let viewers slice the data, for example by model version or product.
- **Publish:** makes a read-only copy that others can view, while your draft stays editable.
- **Storytelling:** each page answers one question, in order: the problem, then how good the model is, then the model in action, then how reliable the pipeline is.

## Build
Go to **New → Dashboard** and name it "Fraud Detection: IEEE-CIS".

### Page 1: The problem (the data)

| Widget | Query idea | Chart |
|---|---|---|
| Transactions, fraud rate | `COUNT(*)`, `AVG(isFraud)` on `silver.train` | Counters |
| Fraud rate by `ProductCD` | `GROUP BY ProductCD` | Bar |
| Fraud rate by card type (`card4`, `card6`) | `GROUP BY` | Bar |
| Fraud rate by email domain | `HAVING COUNT(*) >= 1000`, to skip tiny domains | Bar |
| Fraud rate by hour of day | `hour(event_ts)` | Line |
| Amount, fraud vs. legit | `TransactionAmt` split by `isFraud` | Histogram or box |

### Page 2: How good is the model?

| Widget | Source | Chart |
|---|---|---|
| AUPRC, precision, recall of the live model | `gold.model_metrics` | Counters |
| Caught, missed, false alarms | `gold.validation_predictions` | Counters |
| AUPRC by model version | `gold.model_metrics` | Bar (this shows off your versioning) |
| Score distribution, fraud vs. legit | `gold.validation_predictions` | Histogram |

### Page 3: The model in action

| Widget | Source | Chart |
|---|---|---|
| Model version in use | `SELECT DISTINCT model_version FROM gold.predictions` | Counter |
| Flagged count and % on the test set | `gold.predictions` | Counters |
| Flagged per day | `date(event_ts)` | Line |
| Top 50 riskiest transactions | `ORDER BY fraud_score DESC LIMIT 50` | Table |

### Page 4: Is the pipeline reliable?

| Widget | Query idea | Chart |
|---|---|---|
| Rows per layer | Counts for Bronze, Silver and Gold | Bar |
| Rows dropped by quality rules | Bronze count minus Silver count | Counter |

Advanced: the pipeline's event log has counts per expectation:
`SELECT * FROM event_log(TABLE(workspace.silver.train))`.

### Publish and save
1. **Publish** the dashboard.
2. Export it (**⋮ → Export**) as `.lvdash.json` into `dashboards/`, add 2–3 screenshots, and commit.
3. Don't add a refresh schedule. The data only changes when the job runs.

### Rehearse
Practise a 5-minute walkthrough: page 1 is the problem, page 2 is how good the model is, page 3 is the model in action, page 4 shows it's reliable.

## Done when
- [ ] 4 pages, published
- [ ] `.lvdash.json` and screenshots in Git
- [ ] Walkthrough rehearsed at least once

## Check yourself
- Why show precision and recall instead of accuracy to a non-technical audience?
- Which widget proves that model versioning works?
- Which widget would you show first to a fraud-team manager, and why?

## Pitfalls
- A slow dashboard usually means too much data. Aggregate in SQL, and never pull 500K rows into a table widget.
- Using full names like `workspace.gold.predictions` is fine. In Chapter 9 you'll find-and-replace the catalog name in the exported JSON.

Next: [Chapter 9, Move to Azure](09-move-to-azure.md)
