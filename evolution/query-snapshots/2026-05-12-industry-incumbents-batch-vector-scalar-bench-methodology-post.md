---
query-key: vector-scalar-bench-methodology
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch
wiki-pages-total: 96
cited-pages: [
  systems/elasticsearch.md,
  systems/databricks-vector-search.md,
  systems/opensearch.md,
  systems/snowflake-cortex-search.md,
  systems/mongodb-atlas-vector-search.md,
  systems/redis-stack.md,
]
cited-count: 6
---

# Post-snapshot (industry-incumbents-batch): vector-scalar-bench-methodology

## TL;DR

**重大 NEW** (cross-vendor hybrid retrieval fusion strategy 完整 industry 数据点):

| Vendor | Hybrid Fusion Strategy |
|---|---|
| **Elasticsearch** | **Disjunction + weighted sum**: `score = b1 * match_score + b2 * knn_score`, 用户控 boost |
| **Databricks Vector Search** | **RRF with rrf_param=60** (Reciprocal Rank Fusion) + Okapi BM25 + HNSW vector |
| **OpenSearch** | **RRF processor** + Neural Sparse + BM25 + Vector (3-way fusion) |
| **Snowflake Cortex Search** | **Hybrid + semantic rerank** (keyword + vector + rerank pipeline, BM25 未明示) |
| **MongoDB Atlas** | **Aggregation pipeline** combining `$vectorSearch` + `$search` results (fusion 算法未明示) |
| **Redis Stack** | **`FT.SEARCH`** 内 vector + text filter (fusion via filter intersection, 不是 RRF) |

**关键 NEW**: fair vector + scalar + text bench methodology 现在有 **6 vendor 不同 fusion strategy 横向对比基础**:
- **Weighted sum** (ES): 简单, user-tunable, OOD 不鲁棒
- **RRF (Databricks/OpenSearch)**: 通用, rank-based, OOD 鲁棒, 是 **2 vendor production 验证**
- **Multi-stage rerank** (Snowflake): keyword + vector + rerank 3 阶段, latency 高 但 quality 好
- **Aggregation-pipeline** (MongoDB): 灵活但 fusion 算法 opaque
- **Filter intersection** (Redis): 不是真 hybrid fusion, 是 boolean AND

**关键 NEW fair benchmark 设计原则** (从 industry 数据点导出):
1. **Fix embedding model** (e.g. BGE-M3 OSS) 排除 embedding 变量
2. **Specify fusion strategy** (RRF vs weighted vs rerank) - 6 vendor 至少 4 类
3. **Filter selectivity sweep** 是关键 axis - 不同 vendor 在 high/low selectivity 性能差异巨大
4. **Latency budget** 与 fusion 复杂度 trade-off 显化

## Cited Pages

- [systems/elasticsearch.md](../../systems/elasticsearch.md)
- [systems/databricks-vector-search.md](../../systems/databricks-vector-search.md)
- [systems/opensearch.md](../../systems/opensearch.md)
- [systems/snowflake-cortex-search.md](../../systems/snowflake-cortex-search.md)
- [systems/mongodb-atlas-vector-search.md](../../systems/mongodb-atlas-vector-search.md)
- [systems/redis-stack.md](../../systems/redis-stack.md)
