# Chapter 7: Orchestrate with a Job

**Goal:** one job, `fraud_e2e`, that runs pipeline → train → evaluate → score in order.

## Concepts

- **Job:** a set of tasks linked by dependencies, forming a graph (a DAG). A task can be a notebook, a pipeline, a SQL query, and more.
- **Dependencies:** a task starts only after the tasks it depends on have succeeded.
- **Job parameters:** values, such as `catalog`, passed to every task. Notebooks receive them as widget values.
- **Triggers:** manual, scheduled (cron), on file arrival, or continuous. Manual is the cheapest while you're learning.
- **Repair run:** re-runs only the failed task and the tasks after it.
- **Compute:** Free Edition runs everything on serverless. On Azure (Chapter 9) you'll switch the tasks to job compute.

## Build

### 1. Create the job
Go to **Jobs & Pipelines → Create → Job** and name it `fraud_e2e`.

### 2. Add tasks

| Task | Type | Source | Depends on |
|---|---|---|---|
| `ingest_clean` | Pipeline | `fraud_pipeline` | none |
| `train` | Notebook | `notebooks/02_train` | `ingest_clean` |
| `evaluate` | Notebook | `notebooks/03_evaluate` | `train` |
| `score` | Notebook | `notebooks/04_score` | `evaluate` |

For each notebook task's source, pick either:
- a **workspace path** inside your Git folder, which is simpler, or
- **Git provider**, where the job pulls the committed code from GitHub on every run. That's closer to how production jobs work.

### 3. Job parameter
Add `catalog` = `workspace` at the job level. Your widgets pick it up without any change to the notebooks.

### 4. Notifications
Turn on email notifications for failures.

### 5. Run it, then break it
1. Click **Run now** and watch the graph.
2. Put a typo in `03_evaluate` and run again. It fails.
3. Fix the typo and click **Repair run**. Only `evaluate` and `score` run again.

### 6. Save the job definition to Git
Find the job's code view (look for **View as code** / YAML), copy the YAML into `resources/fraud_e2e.yml`, and commit it. Chapter 9 recreates the job from this file.

### 7. Trigger
Leave the job on manual. If you want a schedule, run it weekly at most, because Free Edition has a daily compute quota.

## Done when
- [ ] A full run is green
- [ ] You've tried a repair run
- [ ] The job YAML is in Git

## Check yourself
- `train` succeeds but `evaluate` fails. What does `score` do, and why is that good?
- What do you gain by using "Git provider" as the task source?
- Why pass `catalog` as a job parameter instead of editing the notebooks?

## Pitfalls
- The pipeline task has a **full refresh** option. Leave it off; otherwise every run reloads all the CSVs from scratch.
- A notebook that hard-codes values instead of reading widgets ignores the job parameters.

Next: [Chapter 8, Dashboard](08-dashboard.md)
