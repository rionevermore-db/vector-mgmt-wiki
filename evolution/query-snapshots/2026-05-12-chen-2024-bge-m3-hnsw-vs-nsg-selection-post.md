---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md, concepts/hnsw.md]
cited-count: 2
---

# Post-snapshot (chen-2024-bge-m3): hnsw-vs-nsg-selection

## TL;DR

BGE-M3 是 embedding model 不是 ANN algorithm. 但 BGE-M3 三 output (dense + sparse + colbert) 各自走不同 retrieval index path——dense 走 HNSW, sparse 走 inverted index, colbert 走 token-level inverted file. **关键 NEW**: BGE-M3 production 需要 vendor 同时支持 3 index path——Vespa / Milvus 是 wiki 内最 first-class native (3-way 全支持).

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
