---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md, concepts/hnsw.md]
cited-count: 2
---

# Post-snapshot (muennighoff-2023-mteb): hnsw-vs-nsg-selection

## TL;DR

MTEB 是 embedding model benchmark, 不直接评估 ANN algorithm (HNSW/NSG). 但 **MTEB Retrieval task** 实际包含 ANN 影响——若 MTEB protocol 用 exact search vs approximate, 不同 embedding model 在 ANN context 下 ranking 可能 shift. Wiki 内 HNSW vs NSG production deployment 数 不变 (7 OSS DBMS HNSW + 0 NSG).

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
