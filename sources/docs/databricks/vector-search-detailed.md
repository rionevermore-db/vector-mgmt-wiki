# Databricks Vector Search — Detailed Architecture (Azure docs source-of-truth)

Source URL: https://learn.microsoft.com/en-us/azure/databricks/generative-ai/vector-search (raw markdown, GitHub-hosted at MicrosoftDocs/databricks-pr)
Fetched: 2026-05-12
Acquisition: WebFetch raw markdown (Azure docs 是 markdown on GitHub, 比 docs.databricks.com 更原始)
Last updated by Databricks: 2026-05-11

## 核心算法 + Distance Metric

> "Vector Search uses the **Hierarchical Navigable Small World (HNSW)** algorithm for its approximate nearest neighbor (ANN) searches and the **L2 distance** metric"

Cosine: 必须 normalize embeddings client-side, 因为 L2 ranking == cosine ranking on normalized vectors.

## RRF 公式 (公开)

> "similarity search and keyword search results are combined using the Reciprocal Rank Fusion (RRF) function."

```
RRF(d) = 1 / (rrf_param + rank(d))
```

`rrf_param` controls relative importance of higher-ranked vs lower-ranked documents.

> "Based on the literature, `rrf_param` is set to 60."

Normalization 让 max score = 1.

## 4 种 Index 选项

1. **Delta Sync Index w/ Databricks-managed embeddings** — source Delta + Databricks 算 embedding, 自动 sync
2. **Delta Sync Index w/ self-managed embeddings** — pre-calc embedding column 在 Delta, 自动 sync. **不可转 managed** (须重建)
3. **Direct Vector Access Index** — REST API 手动 update vector + metadata
4. **Full-text search index (Beta)** — storage-optimized endpoint only, **dedicated BM25 only, no embeddings**——是 wiki 内 first vendor 提供"vector DB 内嵌 BM25-only index"

## BM25 tokenization (公开)

> "All text or string columns are searched, including the source text embedding and metadata columns... tokenization function splits at word boundaries, removes punctuation, and converts all text to lowercase."

## Endpoint Types

| Type | Capacity | 注 |
|---|---|---|
| **Standard** | 320M @ 768d / 160M @ 1536d / 80M @ 3072d (linear scaling) | High QPS 可配 `target_qps` |
| **Storage-optimized** (Public Preview) | ~1B @ 768d | 10-20× faster indexing, +250ms latency, 配 dim divisible by 16, 仅 Triggered sync |

## Resource limits (table 摘录)

| Resource | Limit |
|---|---|
| Vector search endpoints / workspace | 500 |
| Embeddings / Direct Vector Access standard endpoint | **~2M @ 768d** (small!) |
| Embeddings / storage-optimized endpoint | ~1B @ 768d |
| Embedding dim / index | 4096 |
| Indexes / endpoint | 50 |
| Columns / index | 50 |
| Index name length | 128 chars |

## Index 操作 limits

| Resource | Limit |
|---|---|
| Row size for Delta Sync Index | 100 KB |
| Embedding source column size | 32764 bytes |
| Bulk upsert / delete (Direct Vector) | 10 MB |

## Query API limits

| Resource | Limit |
|---|---|
| Query text length | 32764 chars |
| Tokens in hybrid | 1024 words / 2-byte chars |
| Filter conditions | 1024 elements / clause |
| Max results (ANN) | **10,000** |
| Max results (hybrid keyword-similarity) | **200** |
| Max results (full-text only) | 200 |
| Response size | 10 MB |

## Storage-optimized 限制 (Beta)

- **Continuous sync 不支持** — 仅 Triggered
- **Columns to sync 不支持**
- **Embedding dim 必须 div by 16**
- Incremental update 部分支持——每 sync 部分重建; previously computed embeddings reused if source row unchanged
- 1B embeddings sync **< 8 hours**
- FedRAMP / Customer-Managed Keys (CMK) 不支持
- Custom embedding model 需启用 AI Query for Custom Models preview

## Auth + 加密

- AES-256 at rest, TLS 1.2+ in transit
- **Service principal token (production 推荐)** vs PAT——SP **100ms faster per query** (相对 PAT)
- CMK supported on endpoints created on or after **May 8, 2024**

## Unity Catalog + Permission

- Unity Catalog enabled workspace 必须
- Serverless compute 必须
- Standard endpoint: source table 必须 enable CDF
- CREATE TABLE privilege on catalog schema
- Row/column 级 permission **不支持**——用 filter API 实现 application ACL

## 关键 features (full list)

- Hybrid keyword-similarity search
- Full-text keyword search (Beta) on any endpoint
- **Dedicated full-text indexes (Beta)** on storage-optimized endpoints
- Filtering (metadata)
- Reranking
- ACL for vector search endpoints
- Sync only selected columns
- Save and sync generated embeddings
