# Chapter 9: Move to Azure Databricks

**Goal:** run the same project on Azure Databricks with job compute, while keeping the cost near zero when it's idle.

## Concepts

- **Workspace tier:** Unity Catalog and serverless SQL need the **Premium** tier.
- **Compute types:**

  | Type | What it is | Use it for |
  |---|---|---|
  | All-purpose | Interactive; you start and stop it yourself. Highest price per DBU. | Developing in notebooks |
  | Job compute | Created for one job run and deleted when the run ends. Cheaper. | Scheduled jobs |
  | Serverless | Managed by Databricks and billed per second | Quick, small workloads |

- **DBU (Databricks Unit):** Databricks' billing unit. On classic (non-serverless) compute, Azure bills the virtual machines separately on top.
- **Single-node cluster:** a driver machine only, with no workers. scikit-learn trains on one machine, so worker machines would sit idle and cost money.
- **Managed resource group:** the Azure resources Databricks creates for you (storage, networking). Some of them bill even when nothing is running.

## Build

### 1. Cost guardrails first
- In the Azure portal, go to **Cost Management → Budgets** and create a budget on the resource group (for example $20 a month), with email alerts at 50%, 80% and 100%.
- Check whether a free trial applies to you. Azure Databricks has offered a 14-day Premium trial that covers DBUs (you still pay for the virtual machines).

### 2. Create the workspace
1. In the Azure portal, go to **Create → Azure Databricks**, choose the **Premium** tier and a region near you.
2. On the **Networking** tab: if *secure cluster connectivity* (No Public IP) is on and you use the default network, Azure adds a **NAT gateway that bills hourly even when idle**. For a learning project, turning this off avoids that cost. The trade-off is that cluster machines get public IP addresses.
3. After it's created, open the managed resource group (`databricks-rg-…`) and check what's in it.
4. New workspaces come with Unity Catalog and a workspace catalog. Note that catalog's name; it's your new `catalog` value.

### 3. Bring the project over
1. Link GitHub and create a Git folder from the same repo.
2. Run `00_setup` with `catalog` set to the new catalog name.
3. Upload the 4 CSVs.
4. Remove the hard-coded catalog from the paths in `01_pipeline`. In the pipeline settings, under **Configuration**, add a `catalog` key, then use it in the SQL:
   `'/Volumes/${catalog}/bronze/raw/train_transaction/'`.
   Also set the pipeline's default catalog.

### 4. Recreate the pipeline and job
- **Pipeline:** use the same notebook, in triggered mode, on serverless if it's available or otherwise the smallest classic compute.
- **Job:** create it from `resources/fraud_e2e.yml`, or rebuild it by hand, and set the `catalog` parameter.

### 5. Switch the notebook tasks to job compute
Create one **job cluster** shared by `train`, `evaluate` and `score`. One shared cluster means it only has to start up once.

| Setting | Value |
|---|---|
| Type | Single node |
| Runtime | Latest **LTS ML** version (scikit-learn and MLflow come preinstalled) |
| Node | Small general-purpose machine, about 4 cores and 16 GB (for example `Standard_D4ds_v5`) |

A job cluster starts with the run and shuts down when the run ends, so nothing sits idle.

### 6. Run and retrain
Run the job. It retrains the model on Azure. Registered models don't copy between workspaces, and retraining is quicker and cheaper than migrating one.

### 7. Dashboard
1. In the `.lvdash.json`, find-and-replace `workspace.` with your new catalog name, then import it (**Workspace → Import**).
2. Create a **serverless SQL warehouse**, size 2X-Small, with auto-stop at the minimum. It only bills while queries are running.

### 8. Idle checklist (whenever you're not presenting)
- [ ] No all-purpose clusters running (set auto-terminate to 10–15 minutes on any you create)
- [ ] The job has no schedule
- [ ] The warehouse has auto-stop on
- [ ] The budget alert is set
- [ ] No NAT gateway in the managed resource group, or you've accepted its cost

If you won't need Azure for months, delete the workspace. With Git and the exports you can rebuild it in about an hour, and you can present from Free Edition in the meantime.

## Done when
- [ ] The job runs green on job compute
- [ ] The dashboard works on Azure
- [ ] You've checked the first week's cost in Cost Management

## Check yourself
- Why a single node instead of a driver plus 2 workers?
- What still costs money when nothing is running?
- Job compute or all-purpose compute: when would you use each?

## Pitfalls
- Leaving an all-purpose cluster running overnight. Always set auto-terminate.
- Choosing a non-ML runtime, which means scikit-learn and MLflow aren't preinstalled.

Next: [Chapter 10, GenAI](10-genai-future.md)
