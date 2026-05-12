# Snowflake Cortex Search — Overview (WebFetch summary)

Source URL: https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview
Fetched: 2026-05-12
Acquisition: WebFetch AI summary (not raw markdown — Snowflake docs are JavaScript-rendered, no llms.txt available)

## Architecture & Key Components

Cortex Search combines multiple retrieval and ranking approaches:

> "Each search query utilizes: Vector search for retrieving semantically similar documents. Keyword search for retrieving lexically similar documents. Semantic reranking for reranking the most relevant documents in the result set."

The system materializes source query results into optimized data structures for low-latency serving, separate from the warehouse compute used for indexing.

## Vector Index Type

The documentation does not specify the underlying index structure (HNSW, IVF, etc.). Only the hybrid retrieval approach is detailed.

## Embedding Model Defaults

Default: `snowflake-arctic-embed-m-v1.5` (768 dimensions, 512-token context, English-only).

Alternatives:
- `snowflake-arctic-embed-l-v2.0` (1024 dims, multilingual)
- `snowflake-arctic-embed-l-v2.0-8k` (8192-token context)
- `voyage-multilingual-2` (32,000-token context)

## Hybrid Retrieval

> "Cortex Search takes a 'hybrid' approach to retrieving and ranking documents" combining vector search and keyword-based retrieval.

BM25 is not explicitly named; the system uses unspecified keyword search mechanisms.

## Reranker Support

> "Semantic reranking for reranking the most relevant documents"

Reranking can be disabled via customization settings.

## Compute / Storage Separation

- **Indexing compute**: User-provided virtual warehouse (MEDIUM or smaller recommended)
- **Serving compute**: Multi-tenant, shared infrastructure (charged per GB/month of indexed data)
- **Storage**: Materialized query table + optimized index structures stored in user account

## Scale Limits

> "The result of the materialized query in the search service must be less than 100M rows in size"

Rate limiting: HTTP 429 if overloaded. Default throughput: 20 QPS per service, 140 QPS account-wide.

## Typical Use Cases

1. **RAG engines**: Retrieval for LLM chatbots with semantic search
2. **Enterprise search**: High-quality search bar backends
