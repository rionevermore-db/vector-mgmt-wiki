---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md]
cited-count: 1
---

# Post-snapshot (muennighoff-2023-mteb): scale-tier-shifts

## TL;DR

MTEB 不直接 tier-shift 视角. 但 MTEB 提供 **embedding model quality vs cost trade-off framework** — production tier-shift 决策同时考虑 embedding model 选择 (per-tier 最优 embedding 可能不同, 例如小 tier 用大 model + 大 tier 用 dim-truncated 或 quantized variant).

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
