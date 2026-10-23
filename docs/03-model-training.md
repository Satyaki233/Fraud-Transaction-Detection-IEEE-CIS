# Chapter 3: Model Creation

**Goal:** a notebook that trains a fraud model on the oldest 80% of `silver.train`, logs it to MLflow, and registers it in Unity Catalog.

## Concepts

- **Time-based split:** train on the past and validate on the future. A random split lets future patterns leak into training and inflates the score.
- **Gradient boosting** (`HistGradientBoostingClassifier`): about 100 small decision trees built one after another, each one correcting the mistakes the earlier trees still make. It puts values into buckets ("Hist") for speed and handles missing values on its own.
- **Class imbalance:** only 3.5% of rows are fraud. `class_weight="balanced"` makes each fraud row count about 27 times as much as a legit row.
- **sklearn Pipeline:** bundles preprocessing (turning text columns into numbers) with the model, so scoring code can't forget a step.
- **MLflow tracking:** every training *run* records its parameters, metrics and model file.
- **UC model registry:** registering the model creates numbered *versions* (v1, v2, …) of `ml.fraud_model`. **Aliases** such as `@challenger` and `@champion` are movable labels that point at a version.

## Build: notebook `02_train` (Python)

### 1. Setup cell
```python
%pip install -U mlflow scikit-learn
dbutils.library.restartPython()
```
Then:
- add the `catalog` widget and run `spark.sql(f"USE CATALOG {catalog}")`
- `mlflow.set_registry_uri("databricks-uc")`
- `mlflow.set_experiment("/Users/<your-email>/fraud-detection")`
- set `MODEL_NAME = f"{catalog}.ml.fraud_model"`

### 2. Load and split
- Load `silver.train` into pandas with `.toPandas()`.
- Compute `cutoff = df["TransactionDT"].quantile(0.8)`. Train on rows `<= cutoff` and validate on rows `> cutoff`.
- Define `NUM_COLS`, `CAT_COLS` and `FEATURES = NUM_COLS + CAT_COLS`.
  - `CAT_COLS` are the text columns: `ProductCD`, `card4`, `card6`, `P_emaildomain`, `R_emaildomain`, `M4`, `M6`, `DeviceType`.
  - **Leave `TransactionID`, `TransactionDT`, `event_ts` and `isFraud` out of the features.** IDs and timestamps don't carry over to new data; the model would just memorise them.

### 3. Build the model
The model can't read text, so the text columns must be turned into numbers first. Build:
1. A `ColumnTransformer` that applies an `OrdinalEncoder` to `CAT_COLS` and passes everything else through (`remainder="passthrough"`).
2. A `HistGradientBoostingClassifier` with `class_weight="balanced"`, `random_state=42`, and `categorical_features` set to the positions of the encoded columns.
3. A `Pipeline` that chains the two.

Hints:
```python
OrdinalEncoder(handle_unknown="use_encoded_value", unknown_value=-1,
               encoded_missing_value=-1)
```
- The model treats **negative category codes as missing**, which is why `-1` works for both unseen and missing values.
- `ColumnTransformer` outputs the transformed columns **first**, followed by the passed-through ones. So the categorical columns sit at positions `0 … len(CAT_COLS)-1`.

### 4. Train and log, in one MLflow run
Inside `with mlflow.start_run(run_name="...") as run:`
- `mlflow.log_params(...)`: log `cutoff`, the number of features and the model settings. **`cutoff` is required, because Chapter 4 must use exactly the same split.**
- Fit on the training rows.
- Record where the data came from:
  `mlflow.log_input(mlflow.data.from_spark(sdf, table_name=f"{catalog}.silver.train"), context="training")`
- Log and register the model:
  ```python
  model_info = mlflow.sklearn.log_model(
      model, name="model",
      input_example=X_train.sample(100, random_state=0),
      registered_model_name=MODEL_NAME,
  )
  ```
  - MLflow 3 uses `name=`; MLflow 2 used `artifact_path=`.
  - UC requires a *signature* (the expected input and output columns). `input_example` lets MLflow work it out.

### 5. Label the new version `@challenger`
```python
from mlflow import MlflowClient
client = MlflowClient()
client.set_registered_model_alias(MODEL_NAME, "challenger",
                                  model_info.registered_model_version)
```

### 6. Look around
- **Experiments → your run:** the parameters, the dataset, and the artifacts (model folder, input example).
- **Catalog → workspace → ml → fraud_model:** version 1, the `challenger` alias, and the lineage.

## Done when
- [ ] The run is visible, with its parameters, dataset and model artifact
- [ ] `fraud_model` v1 is in Unity Catalog with the alias `@challenger`
- [ ] `cutoff` is logged as a parameter

## Check yourself
- Why do we leave `TransactionID` and `TransactionDT` out of the features?
- What does `class_weight="balanced"` change? What side effect does it have on the predicted probabilities?
- What's the difference between a run, a version and an alias?
- Why put the encoder inside the Pipeline instead of encoding the data before training?

## Pitfalls
- Forgetting `restartPython()` after `%pip` keeps the old library versions loaded.
- "Categorical feature … cardinality > 255" means you included a text column with too many distinct values (such as `DeviceInfo`). Drop it.
- "Model signature required" means you forgot `input_example`.
- MLflow can't work out a column's type when every value in the example is null. Use a bigger sample.
- Running out of memory on `toPandas()`: select only the columns you need before converting.

Next: [Chapter 4, Testing the model](04-model-testing.md)
