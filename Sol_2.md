##  **Cosmos DB Analytical Store + Synapse Link**

###  Overview

Leverage **Azure Cosmos DB Analytical Store** with **Synapse Link**, which allows you to offload old data for analytics and long-term retention **without moving it out of Cosmos DB**, while reducing costs.

---

### How It Works

1. **Enable Analytical Store** on your existing Cosmos DB container.
2. **Configure TTL (Time-to-Live)** on operational (hot) data to retain only recent 3 months in the transactional store.
3. **Old data automatically moves to the Analytical Store** (seamless, no downtime, no custom code).
4. You can **query old records via Azure Synapse Serverless SQL**, or via **Azure Data Explorer** — both support querying with acceptable latency (a few seconds).
5. **Read API unchanged** — for old data access, you can transparently query Synapse as a fallback only if the transactional query fails.

---

### Why It’s Simpler

* **No code needed for data movement**
* **No need to manage Blob Storage**
* **No service downtime**
* **No change to existing Cosmos DB write path**
* Azure manages the tiering between hot and analytical storage.

---

### Steps to Implement

#### 1. **Enable Analytical Store**

```bash
# In Azure Portal or CLI:
az cosmosdb sql container update \
  --account-name <your-cosmos-account> \
  --database-name Billing \
  --name Records \
  --analytical-storage-ttl -1  # Keep forever
```

#### 2. **Enable TTL for Operational Store**

```bash
az cosmosdb sql container update \
  --account-name <your-cosmos-account> \
  --database-name Billing \
  --name Records \
  --ttl 7776000  # 90 days in seconds
```

This keeps recent 90 days in transactional storage; older data moves to analytical store.

#### 3. **Set Up Synapse Link**

* In Azure Portal, connect your Cosmos DB account to Azure Synapse.
* Use Synapse SQL Serverless Pools or Spark for querying.

#### 4. **Update Internal Access Layer (Optional)**

If needed, update the data access layer to:

```python
try:
    record = cosmos_transactional_store.read_item(...)
except NotFound:
    # fallback to Synapse query
    record = run_synapse_sql_query(f"SELECT * FROM Records WHERE id = '{record_id}'")
```

This logic can be wrapped in your service’s DAL without affecting API endpoints.

---

### Cost Comparison

| Storage Type            | Cost Impact   | Notes                          |
| ----------------------- | ------------- | ------------------------------ |
| Cosmos DB transactional | Reduced 90%   | Only recent data is kept       |
| Cosmos Analytical Store | Very low      | Designed for long-term queries |
| Synapse Link            | Pay-per-query | No provisioning unless needed  |

---

### Key Benefits

* Simple, native Azure solution
* Zero service disruption
* No need for Azure Functions, Blobs, or new services
* Preserves all historical data
* Fast implementation (can be done in 1–2 days)

---

## When to Choose This

* You prefer simplicity and **native integration**.
* You're fine with slightly **slower access** (\~seconds) to old data.
* You don’t want to maintain custom archival/retrieval logic.
