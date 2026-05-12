---
query-key: hnsw-vs-nsg-selection
date: 2026-05-12
phase: post
ingest-context: range-filter-ann-2024
wiki-pages-total: 97
cited-pages: [concepts/range-filter-ann-2024.md]
cited-count: 1
---

# Post-snapshot (range-filter-ann-2024): hnsw-vs-nsg-selection

## TL;DR

间接 NEW: UNIFY HSIG = **HNSW-inspired hierarchical SIG**——HNSW 在 range filter ANN scope 内仍是 base model, NSG 在 range filter 论文内 zero usage. **HNSW 在 filter ANN 维度也是 dominant base**——production HNSW 不仅 unfiltered ANN 主导, 在 filter ANN algorithm core 也是 base.

SeRF 同样基于 PG (proximity graph), 但 SeRF 自身 segment graph 不是直接 HNSW; iRangeGraph elemental graphs 是 PG-based 未明示具体算法.

## Cited Pages

- [concepts/range-filter-ann-2024.md](../../concepts/range-filter-ann-2024.md)
