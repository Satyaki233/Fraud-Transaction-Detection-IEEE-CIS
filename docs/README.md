# Fraud Detection on Databricks — Build Guide

A learning project: you build an end-to-end fraud-detection system on Databricks
using the [IEEE-CIS Fraud Detection](https://www.kaggle.com/c/ieee-fraud-detection)
dataset, one chapter at a time.

These docs tell you **what** to build and **why**, with hints for the tricky parts.
The code is yours to write.

## How to use these docs

1. Read the chapter's **Concepts** first.
2. Do the **Build** steps. Hints show syntax, not full solutions.
3. Tick every box in **Done when** before moving on.
4. Answer **Check yourself** from memory. If you can't, re-read or ask.
5. Commit to Git at the end of every chapter.
6. Stuck? Ask Claude: paste the error and say which chapter and step you're on.

## What you'll build

```
Kaggle CSVs ──► Volume (raw files)
                   │
                   ▼
   ┌──── Pipeline: 01_pipeline ─────────────────────┐
   │  Bronze: 4 raw tables ──► Silver: train, test  │
   └────────────────────────────────────────────────┘
                   │                       │
                   ▼                       ▼
          02_train (sklearn)        04_score (@champion)
          → registers model vN             │
                   │                       ▼
                   ▼                gold.predictions
          03_evaluate                       │
          → gold.model_metrics              │
          → promotes @champion              │
                   │                       │
                   └──────► Dashboard ◄─────┘
                               ▲
                     05_genai (future): LLM notes, Genie
```

The job `fraud_e2e` runs the pipeline, then `02` → `03` → `04`, in that order.

## Chapters

| # | Chapter | You build | Main concept |
|---|---|---|---|
| 0 | [Setup](00-setup.md) | Git folder, schemas, volume, uploaded CSVs | Unity Catalog, medallion layers |
| 1 | [Ingestion (Bronze)](01-ingestion-bronze.md) | 4 streaming tables | Declarative pipelines, Auto Loader |
| 2 | [Cleaning (Silver)](02-cleaning-silver.md) | `silver.train`, `silver.test` | Joins, expectations, missing values |
| 3 | [Model creation](03-model-training.md) | `02_train` notebook, model v1 | Time split, gradient boosting, MLflow |
| 4 | [Testing the model](04-model-testing.md) | `03_evaluate` notebook | Precision, recall, AUPRC, threshold |
| 5 | [Prediction](05-prediction.md) | `04_score` notebook | Batch inference, loading by alias |
| 6 | [Version control](06-version-control.md) | Promotion rule, rollback drill | Champion/challenger, Delta history |
| 7 | [Job orchestration](07-job-orchestration.md) | `fraud_e2e` job | Task dependencies, parameters |
| 8 | [Dashboard](08-dashboard.md) | 4-page AI/BI dashboard | Telling the story with data |
| 9 | [Move to Azure](09-move-to-azure.md) | Same project on Azure, job compute | Compute types, cost control |
| 10 | [GenAI (future)](10-genai-future.md) | LLM analyst notes, Genie space | AI Functions, grounding |

## Names used throughout

| Thing | Name |
|---|---|
| Catalog | `workspace` in Free Edition. Every notebook reads it from a `catalog` widget. |
| Raw files | Volume `bronze.raw`, i.e. `/Volumes/<catalog>/bronze/raw/` |
| Bronze tables | `bronze.train_transaction`, `bronze.train_identity`, `bronze.test_transaction`, `bronze.test_identity` |
| Silver tables | `silver.train` (has `isFraud`), `silver.test` (no label) |
| Gold tables | `gold.model_metrics`, `gold.validation_predictions`, `gold.predictions` |
| Model | `<catalog>.ml.fraud_model`, with aliases `@challenger` and `@champion` |
| MLflow experiment | `/Users/<your-email>/fraud-detection` |
| Pipeline / job | `fraud_pipeline` / `fraud_e2e` |

## Final repo layout

```
fraud-detection-IEEE-CIS-dataset/
├── docs/            ← you are here
├── notebooks/
│   ├── 00_setup
│   ├── 01_pipeline      (SQL, used by the pipeline)
│   ├── 02_train
│   ├── 03_evaluate
│   ├── 04_score
│   └── 05_genai         (future)
├── resources/       ← job definition (YAML)
└── dashboards/      ← exported dashboard + screenshots
```

## Before you start

- **Accounts:** Databricks Free Edition, Kaggle (accept the competition rules), GitHub.
- **Skills:** basic Python and SQL. pandas helps. You'll learn PySpark and MLflow along the way.

## Free Edition limits that affect this project

- **Serverless compute only.** There are no custom or job clusters; those come in Chapter 9 on Azure.
- **One SQL warehouse (2X-Small),** at most 5 job tasks running at once, and one active pipeline per type.
- **Daily compute quota.** If you exceed it, compute stops until the next day, so test with small runs.
- **Restricted internet access.** Download the data on your laptop, then upload it.
- **Inactive accounts can be deleted.** Keep everything in GitHub.
