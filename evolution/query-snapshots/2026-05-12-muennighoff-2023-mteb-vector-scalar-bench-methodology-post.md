---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md]
cited-count: 1
---

# Post-snapshot (muennighoff-2023-mteb): vector-scalar-bench-methodology

## TL;DR

MTEB 评估 embedding model output quality, **不评估 vector + scalar filter joint performance** (system-level benchmark). 但 MTEB-evaluated embeddings 是 production benchmark 的 embedding source——fair vendor benchmark 必须 control embedding choice (用 MTEB-validated same model). 关键 NEW: MTEB 提供 "control embedding axis" 而非"评估 system axis"——是 wiki 内 first benchmark provide embedding standardization framework.

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
