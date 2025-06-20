

# Proposed Solution: Tiered Storage with On-Demand Archival Retrieval

## Architecture Overview


```
              ┌──────────────────────┐ 
              │   Billing API (App)  │ 
              └────────┬─────────────┘ 
                       │ 
               ┌───────▼────────┐ 
               │ Primary Cosmos │◄────────┐ 
               │    DB (Hot)    │         │ 
               └───────┬────────┘         │ 
                       │                  │ 
          Record Not Found (Old Record)   │ 
                       │                  │ 
               ┌───────▼────────┐         │ 
               │  Archive Proxy │         │ 
               └───────┬────────┘         │ 
                       │                  │ 
       ┌───────────────▼──────────────┐   │ 
       │  Azure Blob Storage (Cool)   │   │ 
       │ JSON files / containers      │   │ 
       └──────────────────────────────┘   │ 
                       │                  │ 
                  (If requested)          │ 
                       │                  │ 
            ┌──────────▼───────────┐      │ 
            │  Load to Cosmos (TTL)│◄─────┘ 
            │   or Stream Record   │ 
            └──────────────────────┘ 
```

---

## Key Ideas

1. **Data Lifecycle Management**
   - **Hot Storage (Cosmos DB)**: Store only the most recent 3 months of billing records.
   - **Cold Storage (Blob)**: Archive older records in JSON format in Azure Blob Storage.
   - **Lazy Rehydration**: Retrieve archived records on demand and optionally cache in Cosmos DB with short TTL.

---

## Benefits

- **Massive Cost Savings**:  
  Cosmos DB storage is costly at scale. Archiving to Blob Storage (Cool or Archive tier) provides significant savings.

- **No API Changes**:  
  API remains unchanged; retrieval logic is updated internally.

- **No Downtime or Data Loss**:  
  Archival is handled asynchronously without disrupting production workloads.

---

## Implementation Plan

### 🔹 Phase 1: Data Archival Script

**Azure Function** (Timer-Triggered) to move old records to Blob Storage.

#### Sample Python Pseudocode

```python
import datetime, json
from azure.storage.blob import BlobServiceClient
from azure.cosmos import CosmosClient

COSMOS_URI = "<your-uri>"
COSMOS_KEY = "<your-key>"
DB_NAME = "Billing"
COLLECTION_NAME = "Records"

BLOB_CONN_STR = "<your-blob-connection-string>"
CONTAINER_NAME = "billing-archive"

cutoff_date = datetime.datetime.utcnow() - datetime.timedelta(days=90)

def archive_old_records():
    cosmos = CosmosClient(COSMOS_URI, credential=COSMOS_KEY)
    db = cosmos.get_database_client(DB_NAME)
    container = db.get_container_client(COLLECTION_NAME)

    blob_service = BlobServiceClient.from_connection_string(BLOB_CONN_STR)
    container_client = blob_service.get_container_client(CONTAINER_NAME)

    for item in container.query_items(
        query="SELECT * FROM c WHERE c.timestamp < @cutoff",
        parameters=[{"name": "@cutoff", "value": cutoff_date.isoformat()}],
        enable_cross_partition_query=True
    ):
        blob_name = f"{item['id']}.json"
        container_client.upload_blob(blob_name, data=json.dumps(item), overwrite=True)
        container.delete_item(item, partition_key=item['partitionKey'])
````

**Schedule**: Run daily or weekly depending on data volume.

---

### 🔹 Phase 2: Proxy for Old Record Retrieval

Enhance the data access layer to fallback to Blob Storage if the record is not in Cosmos DB.

#### Sample Logic

```python
def get_billing_record(record_id):
    try:
        # Attempt to retrieve from hot storage
        return cosmos_container.read_item(item=record_id, partition_key=record_id)
    except:
        # Fallback to archived storage
        blob_name = f"{record_id}.json"
        blob_client = blob_container.get_blob_client(blob_name)
        if blob_client.exists():
            data = blob_client.download_blob().readall()
            record = json.loads(data)
            record['ttl'] = 3600  # Optional TTL: 1 hour
            cosmos_container.upsert_item(record)
            return record
        else:
            raise Exception("Record not found")
```

---

### 🔹 Phase 3: Optional Enhancements

* **TTL Indexing**: Automatically expire rehydrated records in Cosmos DB.
* **Batch Archival & Retry Logic**: Use Azure Durable Functions or Logic Apps.
* **Monitoring**: Use Application Insights and Blob Storage metrics for visibility.

---

## Cost Optimization Summary

| Storage Type        | Usage                | Cost Benefit                 |
| ------------------- | -------------------- | ---------------------------- |
| Cosmos DB (Hot)     | Last 3 months only   | \~80-90% storage reduction   |
| Blob Storage (Cool) | All archived records | \~90% cheaper than Cosmos DB |
| Blob Retrieval      | On-demand            | Rare + pay-per-use only      |

---

## Deployment Simplicity

* No service interruption
* Can be deployed as standalone Azure Functions
* Reuses existing Azure services: Cosmos DB, Blob Storage, Azure Functions

---

## Bonus: Terraform Snippet to Create Blob Container

```hcl
resource "azurerm_storage_account" "billing_archive" {
  name                     = "billingarchivestorage"
  resource_group_name      = var.resource_group
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  access_tier              = "Cool"
}

resource "azurerm_storage_container" "archive" {
  name                  = "billing-archive"
  storage_account_name  = azurerm_storage_account.billing_archive.name
  container_access_type = "private"
}
```

---

## Solution Summary

* **Scalable**: Supports millions of records over time.
* **Maintainable**: Azure Functions are easy to manage and evolve.
* **Flexible**: Integrates easily into CI/CD pipelines.
* **Future-Proof**: Ready to extend to Azure Data Lake, Synapse, or analytics pipelines.

---
