---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: range-filter-ann-2024
wiki-pages-total: 97
cited-pages: [concepts/range-filter-ann-2024.md]
cited-count: 1
---

# Post-snapshot (range-filter-ann-2024): giga-scale-sharding

## TL;DR

间接 NEW: range filter ANN 在千亿/万亿规模 production 仍是 open——3 paper 都在 < 100M scale 验证. SeRF Ω(n) 压缩在大规模有 attractive 节省, 但 incremental update 缺. UNIFY 2.29× SOTA + incremental update 是最适合大规模 production 的, 但 paper scale 未明示是否 viable at billion+.

关键 NEW: 千亿规模 range filter ANN 是 wiki **open frontier**——无 source-backed billion-scale data point.

## Cited Pages

- [concepts/range-filter-ann-2024.md](../../concepts/range-filter-ann-2024.md)
