---
query-key: index-architecture-global-vs-routed
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch
wiki-pages-total: 96
cited-pages: [systems/elasticsearch.md, systems/databricks-vector-search.md, systems/redis-stack.md]
cited-count: 3
---

# Post-snapshot (industry-incumbents-batch): index-architecture-global-vs-routed

## TL;DR

**重大 NEW** (industry vendor 拓扑选择实测):

| Vendor | 拓扑模式 |
|---|---|
| **Elasticsearch** | **Per-segment HNSW** (Lucene heritage). 单 shard 内多 segment 各自 HNSW, query 跨 segment union + rerank. **新增 wiki 内 first "per-segment local index" 拓扑** (vs (a) global / (c) partition-routing) |
| **Databricks** | Per-endpoint global HNSW (Standard) 或 Storage-optimized 内部架构 (推 IVF-like) |
| **OpenSearch** | 同 ES per-segment + engine 选择 |
| **Redis Stack** | 单 instance HNSW global, Cluster 模式跨 shard partition-routing |
| **MongoDB Atlas** | Per-Search-Node Lucene HNSW |
| **Snowflake Cortex** | 黑盒, materialized index 单 service 100M 上限暗示 single-graph |

**关键 NEW 拓扑分类升级**——之前 wiki 4 类 ((a) scatter-gather / (a') BatANN baton / (c) DSPANN routing / (c') SPIRE multi-level), 现增 **(b) Per-segment local index** (Lucene heritage):
- (b) 拓扑特点: 单 shard 多 segment, 每 segment 独立 HNSW, query 跨 segment 并行 union
- Trade-off: 与 (a) 全局相比 build 速度快 (per-segment 并行)、refresh 自然集成 BM25 inverted index, 但**跨 segment recall fragmentation** (HNSW per-segment 各自 best-first, top-K union 可能丢全局 nearest)
- Production vendor: ES + OpenSearch + MongoDB Atlas 都 inherit Lucene (b) 模式

**总结 5 类拓扑** 现在 wiki 完整:
- (a) Global scatter-gather (traditional 分布式)
- (a') Global + baton-passing (BatANN)
- (b) **Per-segment local** (ES/OpenSearch/MongoDB Atlas - Lucene heritage) ← **NEW**
- (c) Partition-routing per-level optimization (DSPANN/Pinecone-pod)
- (c') Recursive end-to-end accuracy (SPIRE)

## Cited Pages

- [systems/elasticsearch.md](../../systems/elasticsearch.md)
- [systems/databricks-vector-search.md](../../systems/databricks-vector-search.md)
- [systems/redis-stack.md](../../systems/redis-stack.md)
