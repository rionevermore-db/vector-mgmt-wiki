---
query-key: vector-scalar-bench-methodology
date: 2026-05-12
phase: post
ingest-context: modern-embedding-paradigms
wiki-pages-total: 87
cited-pages: [concepts/modern-embedding-paradigms.md, benchmarks/mteb-massive-text-embedding-benchmark.md]
cited-count: 2
---

# Post-snapshot (modern-embedding-paradigms): vector-scalar-bench-methodology

## TL;DR

NEW: fair vector + scalar benchmark 需指定 **embedding model 维度 + paradigm**. 同一 vector DB 在 GTE-110M (768d) vs Gecko (256d) vs NV-Embed-v2 (4096d) 上 filter-vector 性能差异**比 vendor 间差异更大**——dim 影响 PQ compression rate / IVF probe count / scalar filter joint plan cost. 关键 NEW: future vector-scalar benchmark methodology 应**fix embedding model**为 baseline (推荐 BGE-M3 OSS Apache-2.0 + dimension-flexible MRL 切片), 然后比 DB-side impl, 否则 embedding model 噪音 dominates DB-side measurement.

## Cited Pages

- [concepts/modern-embedding-paradigms.md](../../concepts/modern-embedding-paradigms.md)
- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
