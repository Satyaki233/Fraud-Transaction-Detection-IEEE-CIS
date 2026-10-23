# Chapter 10: GenAI (Future Scope)

**Goal:** add LLM features on top of the Gold tables: short analyst notes for flagged transactions, and questions answered in plain English.

## Concepts

- **Foundation Model APIs:** large language models hosted by Databricks, billed per token and called by their endpoint name.
- **AI Functions:** SQL functions that call an LLM. `ai_query()` is the general one; there are also task-specific ones such as `ai_summarize()` and `ai_classify()`.
- **Genie:** a chat interface over your tables. You give it the tables plus some instructions, and it writes the SQL for each question.
- **Grounding:** an LLM can only explain the facts you give it. It does **not** know your model's real reasoning. Give it facts, not open questions.
- **Cost control:** generate the text once, store it in a table and display it from there. Never call an LLM every time someone views the dashboard.

## Build

### 1. "Why flagged" notes (notebook `05_genai`)
1. Open the **Serving** page and find a pay-per-token foundation model endpoint (its name starts with `databricks-`).
2. Take the top 100 flagged rows from `gold.predictions` and join them to `silver.test` to get their details.
3. Build a prompt full of facts: amount, product, card type, email domain, device type, `card_txn_24h` and the fraud score.
4. Save the results to a table:
   ```sql
   CREATE OR REPLACE TABLE gold.fraud_notes AS
   SELECT TransactionID,
          ai_query('<endpoint-name>',
                   format_string('You are a fraud analyst. In 2 sentences, explain why this
                                  transaction looks risky. Facts: amount=%s, product=%s, ...',
                                 TransactionAmt, ProductCD /* , ... */)) AS note
   FROM ...
   LIMIT 100;
   ```
5. Add a page 5 to the dashboard with a table widget showing the notes.

Start with `LIMIT 5` while you're refining the prompt.

### 2. Genie space
1. Go to **New → Genie space** and add `gold.predictions`, `gold.model_metrics` and `silver.train`.
2. Add instructions, for example: "`isFraud = 1` means fraud. Fraud rate = `AVG(isFraud)`. `event_ts` is the transaction time."
3. Add sample questions and test them, for example "Which product code has the highest fraud rate?" or "How many transactions did model version 2 flag?"
4. Read the SQL Genie writes. Is it correct?

### 3. Ideas for later
- **Real reasons:** calculate SHAP values per transaction (a standard way to show which columns pushed a score up) and include them in the prompt. Then the notes reflect what actually drove the model's score.
- **Databricks App** (Streamlit): an analyst chat next to a table of flagged transactions. Free Edition allows up to 3 apps, and they stop after 24 hours.
- **Agent with tools:** looks up a card's history and past notes, then explains the risk.
- **LLM evaluation:** use MLflow tracing and evaluation to check how good the notes are.

## Done when
- [ ] `gold.fraud_notes` holds notes for 100 flagged transactions, shown on the dashboard
- [ ] Genie answers 5 demo questions correctly

## Check yourself
- Why put facts in the prompt instead of asking "why is transaction X fraud?"
- Why store the notes in a table instead of generating them inside the dashboard?
- What can the LLM note claim that the model never actually "thought"?

## Pitfalls
- `concat()` returns NULL if any part is NULL. Use `format_string()`, which writes the text "null" instead, or wrap values in `coalesce()`.
- An endpoint name that's wrong or not available in Free Edition. Check the Serving page.
- Running `ai_query` on 500K rows is slow, expensive and uses up your quota. Always use `LIMIT`.
