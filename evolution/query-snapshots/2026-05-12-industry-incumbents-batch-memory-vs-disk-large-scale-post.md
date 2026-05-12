---
query-key: memory-vs-disk-large-scale
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch
wiki-pages-total: 96
cited-pages: [systems/elasticsearch.md, systems/redis-stack.md, systems/databricks-vector-search.md, systems/opensearch.md]
cited-count: 4
---

# Post-snapshot (industry-incumbents-batch): memory-vs-disk-large-scale

## TL;DR

**重大 NEW** (production memory-vs-disk spectrum 完整):

| Vendor | Memory/Disk 策略 |
|---|---|
| **Redis Stack** | **Pure RAM-only**, no disk index tier. Sub-ms latency, RAM-bound scaling |
| **Elasticsearch HNSW** | **HNSW data must fit page cache** (硬约束). 单节点 ~ RAM-bound, 但 quantization 可压缩到 ~ 4× |
| **Elasticsearch bbq_disk** | DiskBBQ 部分 disk-resident, 突破 page cache 约束 |
| **OpenSearch Faiss IVF** | IVF + PQ 可 disk-resident, 大规模 path |
| **Databricks Standard endpoint** | RAM-optimized, 320M vectors @ 768d 上限 |
| **Databricks Storage-Optimized** | **Disk-tier explicit**, 1B vectors, +250ms latency trade-off, 10-20× faster build |
| **Snowflake Cortex Search** | 黑盒, 100M 行上限暗示 RAM-resident |
| **MongoDB Atlas** | Search Nodes 分离, doc 未明示 disk 策略 |

**完整 spectrum 显化**:
- **Pure RAM 端**: Redis Stack (sub-ms, scale 受限)
- **RAM-required + Quantize**: Elasticsearch HNSW + int8 (中等规模)
- **Hybrid RAM+Disk**: ES bbq_disk / Databricks Standard
- **Disk-tier explicit**: Databricks Storage-optimized / OpenSearch Faiss IVF
- **Academic 极端**: SPIRE recursive multi-level SSD / AiSAQ DRAM-free (Ingest #13, #5-8)

**关键 NEW**: 千亿规模 production 现在有 **3 个 viable disk-aware OSS/managed path**——之前 wiki 主要 academic 系统 (SPANN/DiskANN), 现 industry vendor 验证.

## Cited Pages

- [systems/elasticsearch.md](../../systems/elasticsearch.md)
- [systems/redis-stack.md](../../systems/redis-stack.md)
- [systems/databricks-vector-search.md](../../systems/databricks-vector-search.md)
- [systems/opensearch.md](../../systems/opensearch.md)
