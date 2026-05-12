---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch (Snowflake / Databricks / Elasticsearch / OpenSearch / Redis / MongoDB)
wiki-pages-total: 96
cited-pages: [
  systems/databricks-vector-search.md,
  systems/elasticsearch.md,
  systems/opensearch.md,
  systems/redis-stack.md,
  systems/snowflake-cortex-search.md,
  systems/mongodb-atlas-vector-search.md,
]
cited-count: 6
---

# Post-snapshot (industry-incumbents-batch): giga-scale-sharding

## TL;DR

**重大 NEW** (千亿规模 16-node + 1TB RAM 配置选型 production answer 显著加厚):
- **Databricks Vector Search Storage-Optimized endpoint = 1B vectors** ——**唯一 wiki 内公开 1B-class production-ready vendor 上限**, 10-20× faster indexing, +250ms latency; 千亿规模需多 endpoint 联合
- **Elasticsearch + page-cache-fit 硬约束**: 1TB RAM × 16 节点 = 16TB RAM, 减 JVM + 系统 overhead ≈ 12 TB. 768d float32 raw = 3 KB/vec, 1.5T vec × 3KB = 4.5 TB raw, **必须 int8 quantize (4×) 至 1.1 TB**, fit 12 TB 余量;
- **Snowflake Cortex Search 100M 上限 不适合千亿规模**——是 enterprise BI 中等规模工具
- **Redis Stack pure-RAM scaling**: 单节点 1 TB RAM × 16 节点 = 16 TB RAM, 量化后可 fit ~1B vector, **但无 disk tier** → 必须全 RAM + scaling 受限
- **MongoDB Atlas Search Nodes 分离**: workload isolation 模式适合 OLTP + RAG 混合 production, 但 vector scale 上限 doc 未公开
- **OpenSearch + Faiss IVF engine**: IVF + PQ 路径在 page-cache-fit 不要求情况下可突破 ES HNSW 硬约束, 是 large-scale path

关键 NEW: 千亿规模 production OSS 路径现在 **3 个 viable 选**——
1. Databricks Storage-optimized (商业, ~1B 上限, 易部署)
2. Elasticsearch + int8 quantize + 多节点 page cache (OSS-ish, 大量 RAM 需求)
3. OpenSearch + Faiss IVF (OSS Apache 2.0, build cost 高 but scale 突破)

## Cited Pages

- [systems/databricks-vector-search.md](../../systems/databricks-vector-search.md)
- [systems/elasticsearch.md](../../systems/elasticsearch.md)
- [systems/opensearch.md](../../systems/opensearch.md)
- [systems/redis-stack.md](../../systems/redis-stack.md)
- [systems/snowflake-cortex-search.md](../../systems/snowflake-cortex-search.md)
- [systems/mongodb-atlas-vector-search.md](../../systems/mongodb-atlas-vector-search.md)
