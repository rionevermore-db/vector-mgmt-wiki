---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md, systems/vespa.md]
cited-count: 2
---

# Post-snapshot (chen-2024-bge-m3): index-architecture-global-vs-routed

## TL;DR

BGE-M3 3 output 各自 architecture: dense (a-h 任选), sparse (sharded inverted index), colbert (token-level inverted file). 关键 NEW: BGE-M3 production hybrid pipeline 需要 vendor 同时支持 3 architecture path——Vespa rank-profile + 4-phase ranking 是 wiki 内 first-class 唯一支持完整 BGE-M3 3-way hybrid 的 vendor.

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
- [systems/vespa.md](../../systems/vespa.md)
