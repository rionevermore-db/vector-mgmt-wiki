# Elasticsearch k-NN Search — Overview (WebFetch summary)

Source URL: https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
Fetched: 2026-05-12
Acquisition: WebFetch AI summary

## HNSW Algorithm & Segment-Level Indexing

> "Dense vector values per segment as an HNSW graph"

Built **per-segment**, not global. Alternative: **DiskBBQ clusters**.

## dense_vector Field Type

- Configurable dimensions + `similarity` (default cosine)
- Element types: `float` or `byte`
- `index: true` (default)

## Hybrid Retrieval

> "The knn and query matches are combined through a disjunction, as if you took a boolean or between them."

Weighted score: `score = 0.9 * match_score + 0.1 * knn_score` (boost-controllable).

## Quantization

- **int8_hnsw**: float vectors stored as quantized bytes internally
- **bbq_disk**: DiskBBQ clustering + `visit_percentage` tuning
- **bfloat16**: 2-byte per dim
- **Rescoring**: retrieve via int8_hnsw → rescore top-k with original float

## Scale Limits

> "For HNSW, all vector data must fit in the node's page cache for efficient performance."

Building is "compute-intensive"; indexing time-consuming.

## Lucene Filtering Integration

> "If the filtered document count is less than or equal to num_candidates, the search bypasses the HNSW graph and uses a brute force search on the filtered documents."

Adaptive: small filtered set → brute force; large → HNSW + post-filter.
