---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md]
cited-count: 1
---

# Post-snapshot (chen-2024-bge-m3): scale-tier-shifts

## TL;DR

BGE-M3 三 representation 在每 tier 各自有 storage scaling. 关键 NEW: production tier-shift 不仅看 scale, 也看 hybrid representation 选择——小 tier 可 3-way 全 active, 大 tier 可能仅 dense + sparse (skip colbert) 以减 storage. BGE-M3 灵活性: 同 model 三 output 可 selectively use per-tier.

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
