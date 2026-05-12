---
query-key: quantization-landscape
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch
wiki-pages-total: 96
cited-pages: [systems/elasticsearch.md, systems/opensearch.md, systems/mongodb-atlas-vector-search.md]
cited-count: 3
---

# Post-snapshot (industry-incumbents-batch): quantization-landscape

## TL;DR

**重大 NEW** (industry quantization 选型 production 数据点完整):

| Vendor | Quantization 选项 |
|---|---|
| **Elasticsearch** | int8_hnsw / bfloat16 / DiskBBQ / bbq_disk **4 级**; **rescore with original float** = 高 recall 兜底 |
| **OpenSearch (Faiss path)** | Scalar (8-bit, 4-bit) / Binary / PQ |
| **MongoDB Atlas** | Binary + Scalar (doc 提及, 细节未详) |
| **Databricks Vector Search** | doc 未明示, 推 HNSW + L2 raw float (Storage-optimized 可能 PQ-like) |
| **Snowflake Cortex Search** | 黑盒, 未公开 |
| **Redis Stack** | doc 未公开 quantization, 推全 RAM raw float (latency 优先) |

**关键 production pattern**: **Elasticsearch DiskBBQ + bbq_disk 是 wiki 内 first vendor 公开 "binary-bit + cluster + visit-percent control axis"**——recall vs throughput 拐点 user-tunable, 与 PathFinder / SPIRE academic 工作呼应.

**rescore with original float** 是 production 通用 pattern: ES 显式, OpenSearch Faiss path 通用——retrieve via quantized index → rescore top-k with full-precision vectors. wiki 内之前提到但 industry 验证现完整.

## Cited Pages

- [systems/elasticsearch.md](../../systems/elasticsearch.md)
- [systems/opensearch.md](../../systems/opensearch.md)
- [systems/mongodb-atlas-vector-search.md](../../systems/mongodb-atlas-vector-search.md)
