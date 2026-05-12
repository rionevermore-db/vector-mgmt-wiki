---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: thakur-2021-beir
wiki-pages-total: 85
cited-pages: [benchmarks/beir-heterogeneous-zero-shot-ir.md, benchmarks/mteb-massive-text-embedding-benchmark.md]
cited-count: 2
---

# Post-snapshot (thakur-2021-beir): embedding-update-handling

## TL;DR

BEIR + MTEB 共同提供 production embedding upgrade decision framework. **MTEB 评估 embedding model output quality (8 task)**, **BEIR 评估 retrieval system OOD generalization (18 dataset)**——两者合并: 选 embedding model 时 MTEB score gap 决定值得 upgrade 与否, BEIR score 评估实际 retrieval quality OOD impact.

## Cited Pages

- [benchmarks/beir-heterogeneous-zero-shot-ir.md](../../benchmarks/beir-heterogeneous-zero-shot-ir.md)
- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
