# Vector Search — Create Index + Endpoint (WebFetch)

Source URL: https://docs.databricks.com/aws/en/generative-ai/create-query-vector-search
Fetched: 2026-05-12
Acquisition: WebFetch AI summary

## Index Creation Types

### Delta Sync Index (auto-sync from Delta table)

```python
client = VectorSearchClient()
index = client.create_delta_sync_index(
  endpoint_name="vector_search_demo_endpoint",
  source_table_name="vector_search_demo.vector_search.en_wiki",
  index_name="vector_search_demo.vector_search.en_wiki_index",
  pipeline_type="TRIGGERED",
  primary_key="id",
  embedding_source_column="text",
  embedding_model_endpoint_name="e5-small-v2")
```

### Direct Vector Access Index (manual update)

```python
index = client.create_direct_access_index(
  endpoint_name="storage_endpoint",
  index_name=f"{catalog_name}.{schema_name}.{index_name}",
  primary_key="id",
  embedding_dimension=1024,
  embedding_vector_column="text_vector")
```

## Endpoint Creation

```python
client.create_endpoint(
    name="vector_search_endpoint_name",
    endpoint_type="STANDARD",     # or "STORAGE_OPTIMIZED"
    target_qps=500)
```

> "Setting `target_qps` provisions additional capacity, which increases the cost of the endpoint. You are charged for this additional capacity regardless of actual query traffic."

## Sync Modes

- **Continuous**: seconds latency, **higher cost**
- **Triggered**: manual sync initiation, lower cost
- Storage-optimized: **only Triggered**

## Full-text Search Query

```python
results = index.similarity_search(
  query_text="search terms",
  columns=["id", "text"],
  num_results=10,
  query_type="FULL_TEXT")
```

## Prerequisites

- Unity Catalog enabled
- Serverless compute enabled  
- CDF on source table (for Delta Sync)
- GPU endpoint for embedding (recommended for large dataset)
