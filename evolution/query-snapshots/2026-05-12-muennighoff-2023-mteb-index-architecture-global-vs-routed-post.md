---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md]
cited-count: 1
---

# Post-snapshot (muennighoff-2023-mteb): index-architecture-global-vs-routed

## TL;DR

MTEB 不评估 index architecture. Embedding model 选择 (MTEB axis) 与 architecture 选择 (wiki 8 variant axis) 是**独立 axis**——production decision matrix 8 architecture × MTEB-evaluated 多 model 共同决策.

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
