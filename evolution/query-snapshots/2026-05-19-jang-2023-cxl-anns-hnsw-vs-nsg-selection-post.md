---
query-key: hnsw-vs-nsg-selection
date: 2026-05-19
phase: post
ingest-context: jang-2023-cxl-anns
wiki-pages-total: 99
cited-pages: []
cited-count: 0
---

# Post-snapshot (jang-2023-cxl-anns): hnsw-vs-nsg-selection

## TL;DR

**无直接影响**。CXL-ANNS 用 NSG 作图算法（论文沿用 Alibaba production NSG，因 BFS-from-single-entry-node 的 node-level relationship 是其 caching 前提），但不改 HNSW vs NSG 的工程选型 trade-off（构建成本/查询性能/动态更新/内存开销）。唯一弱关联：CXL-ANNS 的 relationship-aware caching 依赖"单一固定 entry-node + 2-3 hop 热点集中"——这是 NSG（单 entry point）天然契合、HNSW（多层多 entry）需适配的特性，可作未来"图选择影响 near-data caching"的引子，但论文未对比 HNSW base。

## Cited Pages

(无)
