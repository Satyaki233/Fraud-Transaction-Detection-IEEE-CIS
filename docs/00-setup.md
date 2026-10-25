# Chapter 0: Setup

**Goal:** a workspace, a Git-backed project folder, a storage layout, and the raw data uploaded.

## Concepts

- **Unity Catalog (UC):** Databricks' system for organising and controlling access to data. Everything has a three-part name, `catalog.schema.object`. In Free Edition your catalog is `workspace`.
- **Schema:** a folder for tables. This project uses one per layer: `bronze` (raw), `silver` (clean), `gold` (results), `ml` (models).
- **Medallion architecture:** data flows Bronze → Silver → Gold, getting cleaner and more useful at each step. You can always rebuild a layer from the one before it.
- **Volume:** UC storage for *files* (CSV, images, …), as opposed to tables. Its path is `/Volumes/<catalog>/<schema>/<volume>/`.
- **Git folder:** a workspace folder linked to a GitHub repo. You commit and push from inside Databricks.
- **Widget:** a notebook parameter. In Chapter 7 the job fills these in.
- **Idempotent:** running a step twice gives the same result as running it once. Setup code should always be idempotent.

## Build

### 1. Accounts and data
1. Sign up for Databricks Free Edition.
2. On Kaggle, open the IEEE-CIS Fraud Detection competition → **Data** → accept the rules → download these four files to your laptop (about 1.3 GB in total):
   - `train_transaction.csv`
   - `train_identity.csv`
   - `test_transaction.csv`
   - `test_identity.csv`
3. On GitHub, create an empty repo named `fraud-detection-IEEE-CIS-dataset`.

### 2. Git folder
1. In Databricks, go to **Settings → Linked accounts** and connect GitHub.
2. Go to **Workspace → Create → Git folder** and paste the repo URL.
3. Inside it, create a `notebooks/` folder. All your notebooks go there.

### 3. Notebook `00_setup` (Python)
It should:
- define a widget `catalog` with default value `workspace`
- create the schemas `bronze`, `silver`, `gold` and `ml` in that catalog
- create a volume `bronze.raw`
- create 4 folders in the volume, one per CSV

Hints:
```python
dbutils.widgets.text("catalog", "workspace")
catalog = dbutils.widgets.get("catalog")

spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog}.bronze")
spark.sql(f"CREATE VOLUME IF NOT EXISTS {catalog}.bronze.raw")
dbutils.fs.mkdirs(f"/Volumes/{catalog}/bronze/raw/train_transaction")
```
Use loops rather than copy-pasting each line.

### 4. Upload
In **Catalog Explorer**, go to `workspace` → `bronze` → `raw`, open each folder, and click **Upload**. Put one CSV in each folder:
```
/Volumes/workspace/bronze/raw/
├── train_transaction/train_transaction.csv
├── train_identity/train_identity.csv
├── test_transaction/test_transaction.csv
└── test_identity/test_identity.csv
```
Why separate folders? Auto Loader (Chapter 1) reads *every* file in a folder. One folder per table keeps the tables apart.

### 5. Commit
Commit `00_setup` from the Git folder and push it.

## Done when
- [x] The 4 schemas and the volume are visible in Catalog Explorer
- [x] Each folder holds exactly one CSV
- [x] `00_setup` is on GitHub

## Check yourself
- What is the volume's full UC name, and what is its file path?
- Why does every notebook read the catalog from a widget instead of hard-coding `workspace`?
- What happens if you run `00_setup` twice? Why is that fine?

## Pitfalls
- You don't need `CREATE CATALOG` in Free Edition, and it may not be allowed. Use `workspace`.
- Putting all the CSVs in one folder would mix the tables together in Chapter 1.

Next: [Chapter 1, Ingestion](01-ingestion-bronze.md)
