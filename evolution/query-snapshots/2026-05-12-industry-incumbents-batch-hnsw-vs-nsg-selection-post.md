---
query-key: hnsw-vs-nsg-selection
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch
wiki-pages-total: 96
cited-pages: [systems/databricks-vector-search.md, systems/elasticsearch.md, systems/opensearch.md]
cited-count: 3
---

# Post-snapshot (industry-incumbents-batch): hnsw-vs-nsg-selection

## TL;DR

**重大 NEW**: HNSW 是**所有 6 个 industry incumbent vendor 的 default**——Databricks (HNSW + L2 公开), Elasticsearch (HNSW per-segment), OpenSearch (HNSW 跨 3 engine), Redis Stack (HNSW + FLAT), MongoDB Atlas (HNSW ANN + ENN exact), Snowflake Cortex (推 HNSW, 黑盒). **NSG 在 industry vendor 内 zero adoption**——production 选型实际是 "HNSW vs nothing"——NSG 仅学术 / Faiss library / 自建系统使用.

production HNSW 是事实 standard 原因: (a) Lucene 内嵌 HNSW Java native, ES/OpenSearch/MongoDB Atlas 自动继承; (b) HNSW dynamic update 优于 NSG (NSG is static); (c) HNSW 论文 (Malkov 2018) 早于 NSG (Fu 2019), 生态先发优势.

关键 NEW: HNSW vs NSG 之争在 **production = HNSW 全胜**, 学术 / 论文比较仍有价值, 但实际选型不存在.

## Cited Pages

- [systems/databricks-vector-search.md](../../systems/databricks-vector-search.md)
- [systems/elasticsearch.md](../../systems/elasticsearch.md)
- [systems/opensearch.md](../../systems/opensearch.md)
