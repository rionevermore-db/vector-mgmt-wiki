---
query-key: multimodal-bench-methodology
date: 2026-05-12
phase: post
ingest-context: range-filter-ann-2024
wiki-pages-total: 97
cited-pages: [concepts/range-filter-ann-2024.md]
cited-count: 1
---

# Post-snapshot (range-filter-ann-2024): multimodal-bench-methodology

## TL;DR

**重大 NEW** (talk 三模 retrieval 主题 algorithm building block):

3 paper 是 **1D numeric range filter ANN**——空间 (lat/lon 2D box) 是 2D range filter 的特例. SeRF 的 **2D segment graph** 是直接 1D → 2D 推广基础, 已**部分解 lat/lon range filter ANN** 问题.

**talk SIGMOD 2026 三模 (vector + scalar + spatial) retrieval 中:**
- vector: HNSW (已成熟)
- scalar (text/equality): ACORN / Filtered-Vamana (已 source-backed)
- **scalar numeric range**: SeRF / iRangeGraph / UNIFY (**新 source-backed**)
- **spatial (2D range / kNN within bbox)**: R-tree + vector hybrid (wiki 仍空白) + SeRF 2D segment graph 部分可用

关键 NEW: 空间能力 = R-tree (geometry-native) 或 SeRF 2D segment graph (numeric range 通用化) 两路径. 后者是 algorithm-unified-with-vector-ANN 路径, **wiki 内 first 提供 vector + spatial unified algorithm 候选**.

3 模检索 algorithm 路径:
1. **多 system 拼接** (vector DB + spatial DB join in app layer)
2. **Unified algorithm** (SeRF 2D 路径, UNIFY 风格 hybrid index 扩展)

但 production vendor 仍 zero 实现 unified spatial + vector—— talk multimodal-bench-methodology gap 持续, 但**algorithm-level 路径现已 source-backed**.

## Cited Pages

- [concepts/range-filter-ann-2024.md](../../concepts/range-filter-ann-2024.md)
