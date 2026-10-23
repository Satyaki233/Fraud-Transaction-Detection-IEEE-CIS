# Chapter 6: Version Control

**Goal:** automate champion/challenger promotion, prove you can roll back, and keep code, data history and project assets versioned.

## Concepts

| Thing | Versioned by |
|---|---|
| Code | Git (GitHub) |
| Model | UC registry versions and aliases |
| Data | Delta table history |
| Job and dashboard | Exported files kept in Git |

- **Champion/challenger:** a new model goes live only if it *beats* the current one on the same test data.
- **Rollback:** move the alias back to an older version. The next scoring run uses the old model, with no code change and no redeploy.
- **Reproducibility:** for any prediction, you can say which model version produced it, what data that version was trained on, and which code commit trained it.

## Build

### 1. Automate promotion (at the end of `03_evaluate`)
The logic, in plain words:
```
challenger_auprc = the AUPRC you just computed
try:
    champion = the version with alias "champion"
    champion_auprc = float(champion.tags["auprc"])
except: there's no champion yet, so promote
if challenger_auprc > champion_auprc: move "champion" to the challenger's version
else: keep the current champion
```
Write it yourself. Also record the decision as a version tag, `promoted = true/false`.

### 2. Create v2 and watch the rule work
1. Change something in `02_train`: add `card_txn_24h`, add more C/D columns, or change `learning_rate`.
2. Run `02_train`, then `03_evaluate`.
3. Did v2 win? Check the aliases in Catalog Explorer and the rows in `gold.model_metrics`.
4. Then make a deliberately worse v3 (for example, only 3 features) and confirm it is **not** promoted.

### 3. Rollback drill
1. Point `@champion` back at v1, either in the UI or with `set_registered_model_alias`.
2. Re-run `04_score`. `gold.predictions.model_version` should now say 1.
3. Point the alias forward again and re-score.

### 4. Data history
`gold.predictions` is a Delta table, and every overwrite creates a new table version:
```sql
DESCRIBE HISTORY workspace.gold.predictions;

SELECT model_version, COUNT(*)
FROM workspace.gold.predictions VERSION AS OF 1
GROUP BY model_version;
```
You can still query the predictions from before the rollback. This is called *time travel*.

### 5. Link code to model
Open a training run in MLflow and look at its tags. Runs from a Git folder usually record the repo and commit automatically. If yours doesn't, commit before training and add the commit hash yourself as a run tag.

### 6. Git habits
- Commit at the end of every chapter, with a message that says what changed and why.
- Try a branch for one experiment (for example `exp/velocity-feature`), and merge it only if the model wins.
- Add a `.gitignore` (at least `.DS_Store`).

## Done when
- [ ] `03_evaluate` promotes only models that are better
- [ ] At least 3 versions exist, and the champion is the best one
- [ ] Rollback drill done; `gold.predictions` history shows both versions
- [ ] Every run shows which code commit produced it

## Check yourself
- A bad model got promoted on Friday. What exact steps undo it in one minute?
- Why compare the challenger and the champion on the *same* validation window?
- What four things would you need to reproduce v2 exactly?

## Pitfalls
- `get_model_version_by_alias` raises an exception when the alias doesn't exist yet. Catch it.
- Comparing AUPRC numbers that were measured on different validation windows. That isn't a fair contest.

Next: [Chapter 7, Job orchestration](07-job-orchestration.md)
