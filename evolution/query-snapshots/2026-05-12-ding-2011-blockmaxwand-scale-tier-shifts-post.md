---
query-key: scale-tier-shifts
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md]
cited-count: 1
---

# Post-snapshot (ding-2011-blockmaxwand): scale-tier-shifts

## TL;DR

BMW BM25 inverted index 在所有 scale tier (≤1B 到 ≥1T) 线性扩展, **无 tier-shift 拐点**——与 dense ANN tier-shift (HNSW → SPANN/DiskANN → DistributedANN) 复杂性形成对比. Sparse path 在 high-tier (≥1T) 仍可工作, dense path 需复杂 distributed architecture.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
