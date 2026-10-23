# Chapter 4: Testing the Model

**Goal:** measure the `@challenger` model honestly on the newest 20% of the training data, choose a flagging threshold, and save the results for the dashboard.

## Concepts

- **Validation set:** data the model never saw but whose answers you know. Here, the newest 20% of `silver.train`.
- **Why not accuracy:** a model that always says "not fraud" is 96.5% accurate and useless.
- **Precision:** of the transactions you flagged, how many were really fraud? Low precision means false alarms, which cost customer trust.
- **Recall:** of all the real fraud, how much did you catch? Low recall means missed fraud, which costs money.
- **AUPRC** (area under the precision-recall curve): one number that summarises precision and recall across every possible threshold. A model that guesses randomly scores about 0.035, the fraud rate.
- **Threshold:** the score above which you flag a transaction. Choosing it is a business decision. Because of `class_weight="balanced"`, a score of 0.5 doesn't mean a 50% chance of fraud.
- **Confusion matrix:** the four counts: caught, missed, false alarms, and correctly passed.

## Build: notebook `03_evaluate` (Python)

### 1. Load the challenger and its split
- Use the same setup cell as Chapter 3 (pip, widget, catalog, registry URI).
- `mv = client.get_model_version_by_alias(MODEL_NAME, "challenger")` gives you `mv.version` and `mv.run_id`.
- Read the cutoff back from the training run: `client.get_run(mv.run_id).data.params["cutoff"]`. Parameters come back as strings, so convert it with `float()`.
- Load the model: `mlflow.sklearn.load_model(f"models:/{MODEL_NAME}/{mv.version}")`.
- Rebuild the validation set: the rows of `silver.train` with `TransactionDT > cutoff`.

> Load it with `mlflow.sklearn` rather than `mlflow.pyfunc`. The sklearn version gives you `predict_proba`, which returns probabilities; pyfunc's `predict` only returns 0 or 1.

### 2. Score and measure
- `proba = model.predict_proba(X_valid[model.feature_names_in_])[:, 1]`
- AUPRC: `average_precision_score(y, proba)`. Also compute `roc_auc_score`, the metric Kaggle uses.
- Threshold: `precision_recall_curve(y, proba)`, then compute F1 = 2PR / (P + R) for every threshold and pick the highest. F1 balances precision and recall.
- At that threshold, compute precision, recall and `confusion_matrix`.

### 3. Save the results

| Where | What | Why |
|---|---|---|
| The MLflow training run | Validation metrics | Compare runs side by side in the UI |
| Model version tags | `threshold`, `auprc` | The threshold travels with the model, and scoring reads it from there |
| `gold.model_metrics` (append) | One row per version: version, auprc, roc_auc, precision, recall, threshold, caught, missed, false_alarms, evaluated_at | Dashboard |
| `gold.validation_predictions` (overwrite) | TransactionID, event_ts, TransactionAmt, actual, fraud_score, flagged, model_version | Dashboard |

Hints:
```python
with mlflow.start_run(run_id=mv.run_id):          # reopen the training run
    mlflow.log_metrics({"val_auprc": auprc, "val_recall": recall})

client.set_model_version_tag(MODEL_NAME, mv.version, "threshold", str(threshold))

spark.createDataFrame(metrics_pdf).write.mode("append").saveAsTable("gold.model_metrics")
```

### 4. Make the first champion (by hand)
In Catalog Explorer, open `ml.fraud_model` → version 1 → **add alias** `champion`. Chapter 6 automates this step.

### 5. Experiment: see leakage for yourself
Make a copy of `02_train`. Use a **random** split (`train_test_split(..., random_state=0)`) instead of the time split, train, and compute AUPRC on the random holdout. Don't register this model. Compare the score with your time-split AUPRC: the random split usually looks better, and that difference is leakage. It makes a good interview story.

## Done when
- [ ] v1 has the tags `threshold` and `auprc`
- [ ] `gold.model_metrics` has a row for v1
- [ ] `gold.validation_predictions` is filled
- [ ] v1 has the alias `@champion`
- [ ] You can say in one sentence: "Of N frauds in the hidden weeks, the model caught X; Y of its Z flags were false alarms."

## Check yourself
- Your AUPRC is 0.45. Is that good? Compared to what?
- If you lower the threshold, what happens to recall, precision and false alarms?
- Why must evaluation use exactly the same cutoff as training?

## Pitfalls
- Evaluating on training rows gives amazing numbers that mean nothing.
- `.predict()` returns 0/1 instead of probabilities, which makes the AUPRC wrong.
- `precision_recall_curve` returns one fewer threshold than precision/recall values. Drop the last precision and recall values before calculating F1 for each threshold.
- Running this notebook twice appends duplicate rows for the same version. Delete that version's rows before appending, or use `MERGE`.

Next: [Chapter 5, Prediction](05-prediction.md)
