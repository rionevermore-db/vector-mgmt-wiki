---
query-key: index-architecture-global-vs-routed
date: 2026-05-12
phase: post
ingest-context: range-filter-ann-2024
wiki-pages-total: 97
cited-pages: [concepts/range-filter-ann-2024.md]
cited-count: 1
---

# Post-snapshot (range-filter-ann-2024): index-architecture-global-vs-routed

## TL;DR

NEW: 3 paper 引入 **"single index 含多 range 信息"** 新拓扑思路, 是 (a) global / (c) routed 之外的**第 6 类**:
- SeRF: 单 segment graph 含 n indexes 信息 (compression)
- iRangeGraph: pre-built elemental graphs + segment tree dynamic compose
- UNIFY: 单 HSIG + range-aware strategy 路径

关键 NEW: filter-aware 索引拓扑 ≠ unfiltered ANN 拓扑分类——本 bundle 在 filter axis 上的拓扑选择**正交于**之前 wiki 5 类 (a)/(a')/(b)/(c)/(c'). production 必须 2 axis 联合设计.

## Cited Pages

- [concepts/range-filter-ann-2024.md](../../concepts/range-filter-ann-2024.md)
