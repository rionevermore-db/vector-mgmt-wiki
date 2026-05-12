---
query-key: scale-tier-shifts
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch
wiki-pages-total: 96
cited-pages: [systems/snowflake-cortex-search.md, systems/databricks-vector-search.md, systems/redis-stack.md, systems/elasticsearch.md]
cited-count: 4
---

# Post-snapshot (industry-incumbents-batch): scale-tier-shifts

## TL;DR

**重大 NEW** (industry vendor 给出明确 scale tier 上限数据点, 质变档实例化):

| 规模 tier | Production vendor 推荐 |
|---|---|
| **< 10M (10 万-1千万)** | 任何 vendor 都可——Redis (latency-critical) / Snowflake Cortex (BI-integrated) / MongoDB Atlas (document-centric) / pgvector |
| **10M-100M (1千万-1亿)** | **Snowflake Cortex (100M 上限是这档 sweet spot)** / Databricks Standard (320M 上限) / Elasticsearch HNSW (single-node + quantize) |
| **100M-1B (1亿-10亿)** | **Databricks Storage-Optimized (1B 上限)** / OpenSearch Faiss IVF / Redis Cluster (quantize) / ES 多节点 quantize |
| **1B-10B (10亿-百亿)** | **OSS academic (SPIRE 8B 验证) + 商业云 (Pinecone scale-out / Databricks 多 endpoint 联合)** |
| **> 10B (百亿+)** | **私有公司内部** (Meta DistributedANN 1.5T 闭门) + OSS academic 边界 (BatANN 1B 上限, SPIRE 8B 上限) |

**关键 NEW 质变点**:
1. **100M → 1B 质变**: Snowflake Cortex 在此档 cliff drop——超过 100M 必须换 vendor; Databricks 内部分 tier (Standard 320M → Storage-Optimized 1B) 体现此质变
2. **1B → 10B 质变**: 所有 industry incumbent vendor doc 都不公开此规模上限——production 1B+ 是高 customization 场景, OOTB 不存在 vendor
3. **每 vendor 一个 implicit scaling ceiling**——production 选型必须 first 知道未来 corpus 增长 trajectory

## Cited Pages

- [systems/snowflake-cortex-search.md](../../systems/snowflake-cortex-search.md)
- [systems/databricks-vector-search.md](../../systems/databricks-vector-search.md)
- [systems/redis-stack.md](../../systems/redis-stack.md)
- [systems/elasticsearch.md](../../systems/elasticsearch.md)
