# Chapter 1: Data Ingestion (Bronze)

**Goal:** a Lakeflow pipeline that loads the 4 CSVs into 4 Bronze tables, incrementally.

## Concepts

- **Lakeflow Declarative Pipeline** (formerly Delta Live Tables, or DLT): you *declare* each table as a query, and Databricks works out the order, runs it, retries failures and tracks data quality. You write *what* you want, not *how* to get it.
- **Streaming table:** a table that processes only *new* input on each run. That suits ingestion, because re-running doesn't reload old files.
- **Auto Loader** (`STREAM read_files(...)`): finds new files in a folder and remembers which ones it has already loaded, using a *checkpoint*.
- **Schema inference:** Auto Loader works out column types from the CSV. Values it can't parse go into a `_rescued_data` column instead of failing the load.
- **The Bronze rule:** keep the data exactly as it arrived, and only add metadata (where and when it came from). Cleaning happens in Silver.

## Build

### 1. Create the pipeline notebook
In `notebooks/`, create `01_pipeline` with **SQL** as its default language.

> A pipeline notebook can't mix languages (no `%python` cells), and you don't run its cells yourself. The pipeline runs them.

### 2. Write 4 streaming tables
Write one table per CSV. Here's the pattern for the first:
```sql
CREATE OR REFRESH STREAMING TABLE bronze.train_transaction
COMMENT "Raw train transactions from the Kaggle CSV"
AS SELECT
  *,
  _metadata.file_path AS _source_file,
  current_timestamp() AS _ingested_at
FROM STREAM read_files(
  '/Volumes/workspace/bronze/raw/train_transaction/',
  format => 'csv',
  header => true
);
```
Write the other three yourself.

### 3. Create the pipeline
1. Go to **Jobs & Pipelines → Create → ETL pipeline** and name it `fraud_pipeline`.
2. For the source code, point it at your `01_pipeline` notebook. (The newer editor may suggest plain `.sql` files instead; notebooks work too.)
3. Set the default catalog to `workspace` and the default schema to `bronze`.
4. Set the mode to **Triggered**, which runs once and stops. Continuous mode runs around the clock and burns through your quota.
5. Click **Run**.

### 4. Inspect
- The pipeline graph should show 4 green boxes.
- Compare row counts:

  | Table | Rows |
  |---|---|
  | `train_transaction` | 590,540 |
  | `train_identity` | 144,233 |
  | `test_transaction` | 506,691 |
  | `test_identity` | 141,907 |

- In Catalog Explorer, check the column types. Is `TransactionAmt` a number? Is `_rescued_data` empty?
- Notice that `test_identity` names its columns `id-01` (with a hyphen) while train uses `id_01`. Leave that alone for now; you'll fix it in Silver.

### 5. Experiment: prove it's incremental
Run the pipeline again without changing anything. In the run details, how many rows were written this time? It should be 0.

### 6. Commit

## Done when
- [ ] 4 Bronze tables with the row counts above
- [ ] A second run loads 0 new rows
- [ ] Committed

## Check yourself
- What's the difference between a streaming table and a normal table?
- Where does Auto Loader remember which files it has already loaded?
- Why don't we clean the data in Bronze?
- If you dropped a 5th CSV into `train_transaction/` and re-ran, what would happen?

## Pitfalls
- If you leave out `STREAM`, the pipeline errors, because a streaming table needs a streaming source.
- A typo in the path gives "path does not exist". Paths are case-sensitive.
- Hard-coding `workspace` in the path is fine for now. Chapter 9 swaps it for a pipeline parameter.

Next: [Chapter 2, Cleaning](02-cleaning-silver.md)
