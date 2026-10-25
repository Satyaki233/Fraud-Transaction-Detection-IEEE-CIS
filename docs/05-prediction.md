# Chapter 5: Predict with the Model

**Goal:** score Kaggle's unlabeled test set with `@champion` into `gold.predictions`, and create a Kaggle submission.

## Concepts

- **Batch inference:** scoring a whole table of new rows in one go, for example once a day. The alternative is real-time serving, which scores one request at a time.
- **Loading by alias:** scoring always uses `@champion`, never a hard-coded version number. Switching models just means moving the alias, with no code change.
- **Train/serve consistency:** scoring must use the same columns, the same encoding (handled by the sklearn Pipeline) and the same threshold (read from the version tag) as training.
- **No labels in production:** you can't measure accuracy on today's predictions, because the answers aren't known yet. That's why Chapter 4 exists.

## Build: notebook `04_score` (Python)

### 1. Load the champion and its threshold
- `mv = client.get_model_version_by_alias(MODEL_NAME, "champion")`
- `threshold = float(mv.tags["threshold"])`
- Load the model by **version number**: `models:/{MODEL_NAME}/{mv.version}`.

> Why look up the alias once and then use the version? If someone moves the alias while this notebook is running, you could otherwise load model v3 alongside v2's threshold.

### 2. Score
- Load `silver.test` into pandas.
- Select the features with `model.feature_names_in_`, which lists the exact columns the model was trained on. Don't copy the feature list into the notebook again.
- `fraud_score = predict_proba(...)[:, 1]`, and `flagged = fraud_score >= threshold`.
- Write `gold.predictions` (overwrite) with these columns:
  `TransactionID, event_ts, TransactionAmt, fraud_score, flagged, model_version, scored_at`

### 3. Kaggle submission (optional but recommended)
- Use exactly two columns: `TransactionID,isFraud`, where `isFraud` is the **probability**, not 0/1.
- It must contain all **506,691** rows.
- Save it to the volume, for example with `pdf.to_csv(f"/Volumes/{catalog}/bronze/raw/submission.csv", index=False)`.
- Download it from Catalog Explorer, then upload it to Kaggle as a **Late Submission**. Kaggle returns a public and a private ROC AUC.
- Record the score as a model version tag (for example `kaggle_private_auc`).

### 4. Inspect
```sql
SELECT flagged, COUNT(*), ROUND(AVG(fraud_score), 3)
FROM workspace.gold.predictions
GROUP BY flagged;
```
Is the flagged percentage similar to what you saw on the validation set? If it's very different, the test period may behave differently from the training period (*drift*). That's a real-world finding worth noting.

## Done when
- [x] `gold.predictions` has 506,691 rows, all with `model_version` filled in
- [x] The submission CSV exists (and, optionally, has a Kaggle score)

## Check yourself
- Why look up the champion by alias, but then load it by version number?
- Why can't you calculate precision and recall from `gold.predictions`?
- If the flagged rate on the test set is double the validation rate, what could that mean?

## Pitfalls
- Using a different column list from training causes an error or silently wrong scores. Use `model.feature_names_in_`.
- Kaggle rejects a submission with missing rows. That's why `silver.test` only warns instead of dropping rows.
- Submitting 0/1 instead of probabilities gives a much lower Kaggle score.

Next: [Chapter 6, Version control](06-version-control.md)
