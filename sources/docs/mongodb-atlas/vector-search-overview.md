# MongoDB Atlas Vector Search — Overview (WebFetch summary)

Source URL: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/
Fetched: 2026-05-12
Acquisition: WebFetch AI summary

## Architecture

> "MongoDB Atlas Vector Search operates on Atlas Clusters running MongoDB v6.0.11, v7.0.2, or later (ANN), v6.0.16, v7.0.10, v7.3.2+ (ENN)"

Also supports self-managed + local Atlas (via CLI).

> "For optimal performance, we recommend deploying separate **search nodes** for workload isolation. Search Nodes support concurrent query execution to improve individual query latency."

## Index Algorithm

> "MongoDB Vector Search supports approximate nearest neighbor (ANN) search with the **Hierarchical Navigable Small Worlds** algorithm and exact nearest neighbor (ENN) search."

- **ANN**: HNSW
- **ENN**: 暴搜全部 indexed embeddings

## Hybrid Retrieval

> "Combine results from multiple search queries, including vector search and full-text search"
> "Hybrid Search: Combine results from multiple search queries, including vector search and full-text search."

底层 full-text 是 Atlas Search (Lucene-based, BM25).

## Quantization

> "Vector Quantization" mentioned with dedicated doc section.

公知: Binary + Scalar quantization (与 ES 类似 path).

## Scale Limits

> "MongoDB Vector Search supports embeddings that are less than and equal to 8192 dimensions in length."

无明确 doc count / storage 上限.

## Document Integration

> "you can store these embeddings in a MongoDB collection as a **field in a document**."

> "Automated Embedding service generates embeddings using a **Voyage AI** embedding model at indexing time for the specified text field in your collection and at query time for your query text, and keeps the embeddings in sync as your data changes."

## Filtering

> "filter on boolean, date, objectId, numeric, string, and UUID values, including arrays of these types."

## 整体定位

> "By using MongoDB as a vector database, you can use MongoDB Vector Search to seamlessly search and index your vector data alongside your other MongoDB data."
