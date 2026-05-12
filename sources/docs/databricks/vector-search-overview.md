# Databricks Vector Search — Overview (WebFetch summary)

Source URL: https://docs.databricks.com/en/generative-ai/vector-search.html
Fetched: 2026-05-12
Acquisition: WebFetch AI summary

## Architecture

Databricks Vector Search is "a vector search solution that is built into the Databricks Data Intelligence Platform" with integrated governance. Creates indexes from Delta tables, queries via REST API, returns semantically similar documents with metadata.

## Vector Index Algorithm

**Hierarchical Navigable Small World (HNSW)** for approximate nearest neighbor.

Similarity: **L2 distance**. Cosine similarity requires datapoint embeddings to be normalized.

## Delta Table Integration

**Delta Sync Index** mode: "As the Delta table is updated, the index stays synced."

Standard endpoints require source table to have **Change Data Feed (CDF) enabled** for row-level change tracking.

## Embedding Model Support

Three options:
1. Databricks-managed embeddings (with "a model that you specify")
2. Self-managed pre-calculated embeddings
3. Direct vector access

For storage-optimized endpoints: custom models require "the AI Query for Custom Models and External Models preview."

## Hybrid Retrieval

"Hybrid keyword-similarity search combines vector-based embedding search with traditional keyword-based search."

- Lexical: **Okapi BM25**
- Fusion: **Reciprocal Rank Fusion (RRF)** with `rrf_param = 60`

## Endpoint Types

- **Standard**: ~320M vectors at 768 dimensions; high QPS
- **Storage-optimized** (Public Preview): "over one billion vectors", "10-20× faster indexing", but ~250ms increased latency

## Scale Limits

- Per workspace: 500 endpoints
- Per standard endpoint: ~320M vectors (768d)
- Per storage-optimized endpoint: ~1B vectors
- Max embedding dimension: 4096
- Per index: 50 columns, 50 indexes per endpoint
